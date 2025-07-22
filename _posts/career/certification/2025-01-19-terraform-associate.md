---
title: Terraform Associate (003) 취득
author: rumi
date: 2025-01-19
categories: [Career, Certification]
tags: [career]
---

## Terraform 자격증을 취득하다
- [공식문서](https://developer.hashicorp.com/certifications/infrastructure-automation)
- 테라폼의 경우 Associate와 Professional 2가지가 존재합니다.   
    입문의 경우 Associate부터가 무난하기 때문에 저도 이것부터 시작했습니다.
- 금액: 70.5$
- 문제: 전부 객관식이고, 57문제
- 결과: 시험 종료 후 바로 확인 가능('PASS'면 합격)
- 뱃지: Credly 뱃지는 시험을 마친 후 25시간 후에 받았습니다.

![terraform-badge](/assets/img/posts/certification/terraform-associate-badge.png)

## 공부 방법
### 1. Udemy
- [연습문제](https://www.udemy.com/course/terraform-associate-practice-exam/?couponCode=KEEPLEARNING)
  - 연습문제들을 살펴보기 위해 구매했고 2번정도 돌려봤습니다.   

### 2. 공식문서
- [공식문서 튜토리얼](https://developer.hashicorp.com/terraform/tutorials/certification-003)
  - 자세히 다 살펴보기엔 오래걸려서 유데미 문제를 통해 부족하다고 느낀부분만 조금더 챙겨봤습니다.

### 3. 개인 준비
1. 커맨드 외우기
- 각 커맨드 사용법에 대해 간단한 숙지
2. 타입, for문, 리소스나 모듈, dynamic 등 코드 작성
- 코드를 보고 리소스명 확인, variable 타입과 실제 값 확인 등
3. terraform 디렉토리 이해
- `terraform.tfstate`, `.terraform` 하위에 저장되는 데이터 이해
4. state 파일 관리
- 저장되는 데이터 이해, 민감정보가 어떻게 저장되는지, state와 실제 코드가 불일치 하는 경우 발생하는 상황에 대한 이해
5. provider
- 어떻게 선언하고 동작하는지 개념
6. Vault, Sentinal 
- 왜 사용하고 어떤 특성인지, 어떻게 통합되는지
7. 퍼블릭 모듈
- 무조건 Github에 퍼블릭으로 등록되어야 함
8. CLI/Community vs HCP 버전별 지원범위 차이점
- VCS 여러개 사용
    - 가능: HCP
      - 워크스페이스에는 1개만 연결 가능하지만 여러개의 VCS 지원
    - 불가능: 단일사용 cli/community
- state lock
    - 가능: HCP
- encryption&backup
    - 가능: HCP
- Private Registry
    - 가능: HCP
9. 디버깅
- `TF_LOG=TRACE` 제일 자세하게 확인하는 법
10.  IaC의 이점
- idempotent, consistent, repeatable, and predictable
- asy to provision and apply infrastructure configurations, saving time
- 위와 비슷한 맥락에서의 이점들을 이해하면 다른 지문들도 가능
11.  테라폼 핵심 3단계
- Write -> Plan -> Apply
12.  Remote State with Locking
- to avoid two or more different users accidentally running Terraform at the same time, and thus ensure that each Terraform run begins with the most recent updated state.
- 락의 장점, 언제 걸고 풀어야 할지
13.  terraform configuration
```
terraform{
	required_version = ""
	required_providers {
		aws = {
		
		}
	}
	cloud {
	}
}
```
14. state 저장소 마이그레이션
- [공식문서](https://developer.hashicorp.com/terraform/tutorials/cloud/cloud-migrate#migrate-the-state-file)
15. 이전에 인프라에 배포된 리소스 import 하는방법
- Before you run terraform import you must manually write a resource configuration block for the resource. The resource block describes where Terraform should map the imported object.
- 무조건 구성블록 생성 후 `import` CLI 진행
- [공식문서](https://developer.hashicorp.com/terraform/cli/import)


## 후기
테라폼을 1년 넘게 사용해왔기 때문에 준비 과정은 오래 걸리지 않았고, 시험 내용도 비교적 어렵지 않았습니다. 유데미를 통해 문제를 풀어보기도 했지만, 실제 시험은 출제 방식이 조금 다르게 느껴졌습니다.

그럼에도 불구하고, 테라폼에 대한 이해도와 사용 경험이 있다면 충분히 해결할 수 있을 정도의 난이도였습니다. 테라폼을 사용하면서 자신의 이해도를 측정해보고 싶다면, 자격증에 도전해보는 것도 좋은 선택일 것 같습니다.

참고로 CNCF 시험은 PSI를 통해 진행되었지만, 이번 테라폼 시험은 Certiverse를 통해 진행되었습니다. Certiverse는 시험 시작 전 환경 확인이 간결하고, 시험 결과를 바로 보여주는 점이 정말 편리했습니다. 같은 객관식 문제라도 Certiverse는 시험 브라우저 환경이 넓고 전체적인 만족도가 높았습니다.