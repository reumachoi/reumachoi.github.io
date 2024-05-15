---
title: Assume Role 이해하고 쓰자 (+EKS)
author: rumi
date: 2024-01-14
categories: [AWS, Security, IAM]
tags: [aws, security, iam, assume_role]
---

> IAM에 대한 전체적인 개념을 이해하고, 보안적으로 안정성을 고려하여 권한을 부여하는 방법을 알 수 있습니다.


## IAM 개념
Identity and Access Management의 약자로 신원 확인 및 접근을 제어하는 리소스입니다.

![메뉴창](./iam_1.png)

AWS 콘솔에서 IAM 리소스 페이지로 들어가면 위와 같이 있습니다.(하위 생략)


## IAM 상세 개념 관계도
![그림설명](./iam_2.png)
- 사용자(User)과 역할(Role)은 정책(Policy)을 권한으로 가질 수 있습니다.
정책(Policy)은 AWS 리소스에 대한 작업의 허용/거부를 JSON 형식으로 나타낸 것입니다.

![iam 사용자 페이지](./iam_3.png)
- imura라는 사용자의 상세페이지입니다.
현재 developers라는 사용자그룹에 속해있어서 AdministratorAccess 정책 권한이 있습니다. 그리고 해당 유저에만 IAMReadOnlyAccess 정책을 직접 부여했습니다.


## Assume Role
### 개념
- IAM 사용자를 사용하여 AWS 콘솔 또는 CLI로 리소스를 보거나 생성 및 삭제 등을 하는 경우엔 사용자에게 부여된 권한을 통해서 사용합니다. 
IAM 사용자에게 액세스키를 생성할 수 있는데, 이 키는 특히 CLI나 SDK로 AWS API 작업을 할때 필요합니다. 하지만 탈취된 액세스키로는 해당 사용자가 가진 권한을 전부 제어가능하기 때문에 보안적으로 위험합니다. 
- 이러한 문제를 해결하기 위해 Assume Role 개념이 생겼습니다. User에게 최소한의 권한만 부여하고, Role의 권한을 임시적으로 위임받는 방법입니다.
- EC2에서 _IAM 인스턴스 프로파일_ 이라는 설정을 봤었다면 이미 Assume Role을 경험해본 것입니다.


### 실습

1. 역할(Role) 생성
- _사용자 지정 신뢰 정책_ 에 Principal에 해당 Role을 사용할 수 있는 유저 관련 범위를 정해줍니다. 
그 이후 단계에서는 추가필요한 리소스등의 정책 연결.
![역할 생성 과정](./iam_4.png)

1. 생성된 역할 확인
- 저는 추가적으로 EC2FullAccess 권한을 부여했습니다.
_신뢰관계_ 탭을 누르면 위에서 _사용자 지정 신뢰 정책_ 으로 설정한 걸 확인할 수 있습니다.
저는 해당 AWS 계정이라면 모든 유저가 해당 역할을 사용가능하도록 "*" 전체허용 했습니다.

![역할 권한 확인](./iam_5.png)
![신뢰 관계 확인](./iam_6.png)
**Tip**
assume role에는 권한이 있지만 user에는 권한이 없어서 AWS 콘솔에서 리소스에 대한 접근이 어려운 경우가 있습니다. 이런 경우엔 해당 Role 페이지로 가면 (2번단계 이미지) **_콘솔에서 역할 전환 링크 _** 가 있는데 이 링크로 들어가서 assume role로 전환해서 사용 가능합니다.


### CLI에서 사용하기
- 아래 예시에서는 eks 클러스터를 kubeconfig 업데이트시 assume role의 ARN을 통해 권한을 임시부여 받아 사용했습니다.
```
aws eks update-kubeconfig --region ap-northeast-2 --name test-eks-cluster --role-arn arn:aws:iam::123456789012:role/test-assume-role
```


## 마무리
- IAM의 사용자, 역할, 정책을 기본적으로 안다면 어렵지 않게 사용할 수 있습니다. 저는 주로 사용자와 정책을 바로 연결해서 썼고 역할은 예시로 들었던 인스턴스 프로필로만 사용했었습니다. Assume Role을 통해 사용자에게 임의 권한 부여에 대해 처음에는 개념자체를 이해하는데 조금 헤매다 보니 어렵다고 생각했었지만 알고보니 정말 간단하고 편리한 것 같습니다. 

