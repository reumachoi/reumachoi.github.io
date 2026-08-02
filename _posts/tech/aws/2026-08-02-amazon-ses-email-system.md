---
title: Amazon SES 메일 시스템 구성 및 검증 메커니즘 (SPF, DKIM, DMARC)
author: rumi
date: 2026-08-02
categories: [Tech, AWS]
tags: [aws, ses, email, spf, dkim, dmarc, route53]
---

웹 서비스의 기본 통신이 HTTPS 중심인 반면, 이메일 전송 환경에서는 **SMTP, DNS, SPF, DKIM, DMARC**가 유기적으로 함께 동작합니다.

본 문서에서는 Amazon SES를 활용하여 `noreply@example.com` 주소로 메일을 안정적으로 발송하기 위한 전체 구성 흐름과 핵심 인증 프로토콜을 정리합니다.

---

## AWS SES 등록 및 프로덕션 전환 흐름

1. **Domain Identity 등록**: 발신 기본 도메인(`example.com`) 등록
2. **Custom MAIL FROM 설정**: Envelope From 도메인(`mail.example.com`) 설정
3. **DKIM 및 Route 53 DNS 레코드 등록**: DKIM 서명 활성화 및 CNAME/TXT 레코드 등록
4. **도메인 검증 완료 확인**: Identity, DKIM, Custom MAIL FROM 상태가 `Verified`인지 확인
5. **Sandbox 해제 신청**: Production Access 신청 및 AWS 승인 획득

```text
Domain Identity 등록 ➔ Custom MAIL FROM 설정 ➔ DKIM & DNS 레코드 등록 ➔ 검증 완료 ➔ Sandbox 해제 신청 ➔ Production Access 승인
```

> 💡 **Domain Identity vs. Production Access**  
> Domain Identity의 `Verified` 상태와 SES 계정의 `Production Access` 승인 상태는 별개입니다. 도메인 검증이 완료되더라도 샌드박스(Sandbox)가 자동으로 해제되지는 않으므로, 외부 일반 사용자에게 메일을 발송하려면 반드시 별도의 승인 절차가 필요합니다.

### Sandbox 상태의 주요 제한 사항
- 발신자 및 수신자 이메일 주소/도메인이 모두 **검증된 Identity**에 속해야 함
- 일일 발송량 및 초당 발송률(TPS) 제한 적용

---

## 주요 구성 요소 및 인증 프로토콜

### 1. Header From (Domain)
- **도메인**: `example.com`
- 최종 수신자가 이메일 클라이언트에서 확인하는 발신자 표시 도메인 (`From: noreply@example.com`).

### 2. Envelope From (Custom MAIL FROM)
- **도메인**: `mail.example.com`
- 실제 SMTP 통신 단계(Envelope)에서 사용되는 발신 도메인 (`Return-Path`).
- 반송(Bounce) 메일 처리와 **SPF / DMARC Alignment** 충족을 위해 사용됩니다.

### 3. SPF (Sender Policy Framework)
- **검증 대상**: 발송 서버의 IP 주소
- **역할**: 특정 IP/서버가 해당 도메인을 대리하여 메일을 보낼 권한이 있는지 검증
- **검증 기준**: Header From이 아닌 **Custom MAIL FROM 도메인**의 DNS TXT 레코드를 참조

```dns
mail.example.com. TXT "v=spf1 include:amazonses.com ~all"
```

### 4. DKIM (DomainKeys Identified Mail)
- **검증 대상**: 발신 도메인 서명 및 메일 본문 무결성
- **역할**: 디지털 서명을 통해 메일의 발신처를 입증하고 전송 과정에서의 위·변조 방지
- **원리**: SES가 비대칭키의 **개인키**로 메일에 서명하고, 수신 서버는 DNS에 등록된 **공개키**로 서명을 검증

### 5. DMARC (Domain-based Message Authentication, Reporting, and Conformance)
- **검증 대상**: SPF 및 DKIM 검증 결과 + Header From과의 Alignment(정렬)
- **역할**: 인증 실패 시 수신 서버가 해당 메일을 어떻게 처리할지에 대한 정책 정의

| 정책 | 처리 방식 |
| :--- | :--- |
| `none` | 처리 정책 없음 (모니터링 및 리포트 수집) |
| `quarantine` | 격리 처리 (스팸함 이동 권고) |
| `reject` | 수신 거부 (메일 차단 권고) |

```dns
_dmarc.example.com. TXT "v=DMARC1; p=none;"
```

> **DMARC 통과 조건 (다음 중 하나 이상 만족)**  
> 1. SPF 검증 성공 **AND** MAIL FROM 도메인이 Header From 도메인과 정렬됨  
> 2. DKIM 검증 성공 **AND** DKIM 서명 도메인이 Header From 도메인과 정렬됨  

---

## 인증 프로토콜 비교

| 구분 | 검증 대상 | 핵심 질문 |
| :--- | :--- | :--- |
| **SPF** | 발송 서버 IP | 해당 서버가 이 도메인을 대신해 메일을 보내도 되는가? |
| **DKIM** | 발신 도메인 & 메일 본문 | 올바른 도메인이 서명했으며 메일 내용이 위변조되지 않았는가? |
| **DMARC** | SPF·DKIM 결과 & 도메인 정렬 | 인증 결과를 바탕으로 메일을 어떻게 처리할 것인가? |

---

## End-to-End 메일 전송 흐름

```text
Application
    │ (AWS SDK / SES API)
    ▼
Amazon SES
    │ (SMTP 전송)
    ▼
수신 메일 서버 (예: Google MX)
    │ (SPF · DKIM · DMARC 검증)
    ▼
사용자 메일함 (Inbox / Spam / Reject)
```
