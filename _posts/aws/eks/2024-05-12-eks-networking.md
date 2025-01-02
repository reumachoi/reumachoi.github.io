---
title: EKS 네트워킹 방식 짚고 넘어가기
author: rumi
date: 2024-05-12
categories: [AWS, EKS]
tags: [aws, eks, vpc, subnet]
---


## 클러스터 엔드포인트 액세스 제어
[클러스터 엔드포인트](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/cluster-endpoint.html)

-> k8s api-server와 통신(kubectl)하기 위함

**종류**
1. only 퍼블릭
	- 모든 통신이 IGW를 거침
2. only 프라이빗
	- VPC 내부 통신 방법으로만 가능(ex. EC2 Bastion Host)
    - [프라이빗 클러스터 요구사항](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html)
3. 퍼블릭&프라이빗
	- 내부 통신은 X-ENI(클러스터 서브넷에 존재)를 통함
    - 외부 통신은 IGW&NAT를 통함
    - 노드는 프라이빗 서브넷에 생성되는 반면, Ingress 리소스(elb)는 퍼블릭 서브넷에 생성
   


## 클러스터 서브넷
[EKS VPC 및 서브넷 요구사항](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/network_reqs.html)
-> 문서에서 서브넷 상세 요구사항 확인
- 지정한 서브넷에 2~4개의 탄력적 네트워크 인터페이스(ENI)를 생성

## 워커노드 서브넷
- 클러스터 서브넷에 지정하지 않은 서브넷도 사용 가능(ENI 미생성)
- 로드 밸런서 배포 관련해서 서브넷에 태그 추가 필요
![](https://velog.velcdn.com/images/reuma/post/28ff10a9-2d54-4506-b4c3-f9168090db70/image.png)



### 참조
[EKS 네트워킹 모범사례](https://aws.github.io/aws-eks-best-practices/ko/networking/subnets/)