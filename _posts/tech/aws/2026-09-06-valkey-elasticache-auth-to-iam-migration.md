---
title: Zero-Downtime Migration from AUTH to IAM Authentication in Amazon ElastiCache Valkey
author: rumi
date: 2026-09-06
categories: [Tech, AWS]
tags: [aws, elasticache, valkey, redis, iam, auth, migration, zero-downtime, acl]
---

Amazon ElastiCache(Valkey / Redis OSS 호환 엔진)를 운영하다 보면 초기에는 고정 비밀번호(AUTH Token) 기반의 인증 모드로 구축하여 사용하는 경우가 많습니다. 그러나 보안 모범 사례(Best Practice)에 따라 암호의 하드코딩을 방지하고, 액세스 권한을 중앙 집권식으로 관리하며, 단기 보안 토큰을 활용하기 위해 **AWS IAM 인증(`elasticache:Connect`)**으로 전환하는 작업이 필요해집니다.

문제는 운영 중인 인프라에서 인증 방식을 변경할 때 서비스 중단(Downtime)이나 클라이언트 접속 끊김이 발생해서는 안 되며, 애플리케이션에 따라 접근 가능한 **키 접두사(Key Prefix) 보안 격리(ACL)**도 필요할 수 있다는 점입니다.

본 포스팅에서는 AWS 공식 문서를 기반으로 **Valkey/ElastiCache 클러스터를 AUTH 모드에서 IAM 인증 모드로 무중단(Zero-Downtime) 마이그레이션한 과정**과 함께, **`user-name`은 `default`로 유지하면서 `user-id`별로 키 접두사 패턴별 ACL을 분리 설정하는 기법**을 함께 정리합니다.

---

## 핵심 개념: ElastiCache RBAC와 무중단 원리

ElastiCache의 사용자 및 접근 제어는 **RBAC (Role-Based Access Control)** 구조를 따르며, 무중단 마이그레이션의 핵심은 **`user-name`과 `user-id`가 분리되어 있다는 점**입니다.

- **`user-name`**: 클라이언트 애플리케이션이 Valkey/Redis 접속 시 명령어로 제출하는 사용자 이름입니다. (기본값: `default`)
- **`user-id`**: AWS ElastiCache 계정 및 리전 내에서 고유하게 구분되는 AWS 리소스 식별자(Resource ID)입니다.
- **`access-string`**: 해당 `user-id`에 부여되는 Redis ACL 접근 규칙입니다. (예: 특정 키 접두사 허용, 명령어 제한 등)

> 💡 **무중단 인증 전환 및 서비스별 ACL 격리 원리**  
> ElastiCache의 **User Group**에는 동일한 `user-name`(`default`)을 공유하는 여러 개의 `user-id`를 동시에 할당할 수 있습니다.  
> 
> 1. **기존 사용자**: `auth-user-01` (`user-name: default`, 인증 방식: Password, 전체 키 접근)  
> 2. **신규 서비스 A 사용자**: `iam-service-a-user` (`user-name: default`, 인증 방식: IAM, ACL: `~service-a:*`)  
> 3. **신규 서비스 B 사용자**: `iam-service-b-user` (`user-name: default`, 인증 방식: IAM, ACL: `~service-b:*`)  
> 
> 이 사용자들을 동일한 User Group에 매핑하면:
> - 마이그레이션 과도기 동안 비밀번호 접속과 IAM 토큰 접속이 모두 동작하는 **이중 인증(무중단)**이 가능합니다.
> - 클라이언트 코드는 `username = default`를 동일하게 사용하더라도, 각 서비스의 IAM Role에 허용된 `user-id`에 따라 **서비스별로 허용된 키 접두사(`service-a:*`, `service-b:*`)에만 접근할 수 있도록 미세한 ACL 제어**가 이루어집니다.

---

## 1단계: AWS IAM Policy 권한 및 user-id 제어 구성

애플리케이션(EC2 IAM Role, EKS IRSA / Pod Identity 등)별로 허용할 ElastiCache `user-id`를 다르게 지정하여 `elasticache:Connect` IAM Policy를 생성합니다.

### 서비스 A IAM Policy 예시 (`iam-service-a-user` 바인딩)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticache:Connect"
      ],
      "Resource": [
        "arn:aws:elasticache:us-east-1:123456789012:serverlesscache:cache-01",
        "arn:aws:elasticache:us-east-1:123456789012:user:iam-service-a-user"
      ]
    }
  ]
}
```

### 서비스 B IAM Policy 예시 (`iam-service-b-user` 바인딩)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticache:Connect"
      ],
      "Resource": [
        "arn:aws:elasticache:us-east-1:123456789012:serverlesscache:cache-01",
        "arn:aws:elasticache:us-east-1:123456789012:user:iam-service-b-user"
      ]
    }
  ]
}
```

- **`Resource`**: 접속 대상 클러스터 ARN과 서비스별로 접근 권한을 제한할 **ElastiCache User ARN(`user-id`)**을 각각 매핑합니다.
- 서비스 A는 `iam-service-a-user`로만 접속 토큰을 발급받을 수 있고, 서비스 B는 `iam-service-b-user`로만 접속할 수 있습니다.

---

## 2단계: user-id별 키 접두사 ACL 설정 및 User Group 매핑

`user-name`은 `default`로 동일하게 지정하되, `user-id`를 분리하고 `--access-string`에 각 서비스 전용 키 패턴(Key Prefix Pattern)을 정의합니다.

### 1) 서비스별 IAM 인증 전용 User 생성 (ACL 키 접두사 설정)

**서비스 A 전용 사용자 생성 (`service-a:*` 키 패턴만 허용):**
```bash
aws elasticache create-user \
    --user-id iam-service-a-user \
    --user-name default \
    --engine "valkey" \
    --authentication-mode Type=iam \
    --access-string "on ~service-a:* +@all"
```

**서비스 B 전용 사용자 생성 (`service-b:*` 키 패턴만 허용):**
```bash
aws elasticache create-user \
    --user-id iam-service-b-user \
    --user-name default \
    --engine "valkey" \
    --authentication-mode Type=iam \
    --access-string "on ~service-b:* +@all"
```

> 🔑 **Redis ACL Access String 해설**  
> - `on`: 사용자 계정 활성화  
> - `~service-a:*`: 키 패턴 제어. `service-a:`로 시작하는 키에 대해서만 작업 허용  
> - `+@all`: 모든 명령어 그룹 허용 (필요 시 `-@dangerous` 추가 가능)

### 2) 기존 User Group에 신규 IAM User 추가
기존 클러스터에 바인딩된 User Group에 생성한 서비스별 IAM 사용자들을 추가합니다.

```bash
aws elasticache modify-user-group \
    --user-group-id my-user-group \
    --user-ids-to-add iam-service-a-user iam-service-b-user
```

이 시점부터 클러스터의 User Group에는 기존 비밀번호 사용자(`auth-user-01`)와 신규 IAM 사용자들(`iam-service-a-user`, `iam-service-b-user`)이 모두 포함되므로, **비밀번호 기반 접속과 IAM 토큰 기반 접속이 동시에 수용**됩니다.

---

## 3단계: 클라이언트 애플리케이션 IAM 토큰 접속 전환 및 무중단 배포

클라이언트 애플리케이션 코드를 수정하여 IAM 인증 토큰을 발급받아 접속하도록 전환합니다.

### IAM Auth Token 발급 및 동작 흐름

```bash
# AWS CLI를 이용한 사전 인증 토큰 발급 예시
aws elasticache get-auth-token \
    --user-name default \
    --region us-east-1
```

1. **토큰 생성**: 애플리케이션은 IAM Role 권한을 기반으로 ElastiCache IAM 인증 토큰을 생성합니다. (토큰 생성 시 `--user-name default` 사용)
2. **클러스터 인증**: Valkey/Redis 접속 시 계정 이름 `default`, 비밀번호 위치에 **생성한 IAM 인증 토큰**을 주입합니다.
3. **IAM 주체 검증 및 ACL 적용**: 
   - ElastiCache는 주체의 IAM Policy를 검증하여 허용된 `user-id`(예: `iam-service-a-user`)를 식별합니다.
   - 클라이언트는 접속 시 `username: default`를 사용했지만, ElastiCache는 `iam-service-a-user`에 설정된 ACL(`~service-a:*`)을 강제 적용합니다.
   - 만약 서비스 A가 `service-b:key`에 접근하려고 하면 `NOPERM Permission denied` 에러가 발생합니다.

애플리케이션은 **카나리(Canary) 배포나 롤링 업데이트(Rolling Update)** 방식으로 순차 배포합니다. 과도기 동안 구버전 Pod/인스턴스는 기존 비밀번호로 접속하고, 신버전 Pod/인스턴스는 IAM 토큰으로 접속하므로 **단 1초의 서비스 중단도 발생하지 않습니다.**

---

## 4단계: 기존 Auth Token(비밀번호) 사용자 제거

모든 애플리케이션의 IAM 인증 전환이 완료되고 비밀번호 접속 트래픽이 끊기면, User Group에서 기존 비밀번호 사용자를 제거합니다.

```bash
# User Group에서 기존 Password User 제거
aws elasticache modify-user-group \
    --user-group-id my-user-group \
    --user-ids-to-remove auth-user-01

# (선택) 불필요해진 비밀번호 User 삭제
aws elasticache delete-user \
    --user-id auth-user-01
```

이로써 Valkey 클러스터는 **IAM 인증 전용 보안 체계 및 키 접두사별 격리(Multi-tenant ACL)**가 완성됩니다.

---

## 요약 및 비교

| 사용자 (`user-id`) | 접속 `user-name` | 인증 방식 | 허용 키 접두사 (ACL Access String) | IAM Policy Resource |
| :--- | :--- | :--- | :--- | :--- |
| `auth-user-01` (기존) | `default` | Password (AUTH) | `~* +@all` (전체 키) | N/A (비밀번호 접속) |
| `iam-service-a-user` | `default` | IAM (`elasticache:Connect`) | `~service-a:* +@all` | `arn:...:user:iam-service-a-user` |
| `iam-service-b-user` | `default` | IAM (`elasticache:Connect`) | `~service-b:* +@all` | `arn:...:user:iam-service-b-user` |

> ⚠️ **운영 팁 (Operational Tips)**
> 1. **`user-name` 통일의 장점**: 모든 애플리케이션의 접속 코드에서 `username = "default"`로 고정할 수 있어, 클라이언트 코드 수정 없이 IAM Policy와 ElastiCache User 매핑만으로 서비스별 ACL 격리가 가능합니다.
> 2. **토큰 유효기간 및 재발급**: IAM 인증 토큰은 발급 후 약 15분간 유효합니다. 커넥션 풀(Connection Pool) 재연결이나 갱신 시 토큰을 동적으로 재발급받도록 클라이언트 라이브러리를 구현해야 합니다.

---

## 참고 문서
- [AWS Docs: Migrating from AUTH to IAM authentication](https://docs.aws.amazon.com/ko_kr/AmazonElastiCache/latest/dg/auth-to-iam-migration.html)
- [AWS Docs: Authenticating with IAM to Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/auth-iam.html)

---

> 🤖 *본 포스팅은 AI를 활용하여 작성되었습니다.*
