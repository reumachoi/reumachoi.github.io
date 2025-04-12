---
title: Sail Operator로 Istio 설치하기
author: rumi
date: 2025-04-13
categories: [Study, Istio]
tags: [Kubernetes, Istio, Service Mesh, Sail Operator]
---

# 개요
최신 Istio 설치 방법인 Sail Operator를 활용한 관리 방식에 대해 상세히 설명합니다.

## Istio 설치 방법의 변화
### Classic Operator(In-Cluster Operator) 지원 중단
Kubernetes에서 Istio 리소스를 관리하기 위한 초기 도구로 Istioctl 명령어를 사용하여 설치를 했습니다.
하지만 Istio 1.23에서 deprecated 되었고, Istio 1.24 출시와 함께 제거될 예정이라고 합니다. [공지 및 마이그레이션 가이드](https://istio.io/latest/blog/2024/in-cluster-operator-deprecation-announcement/)

이를 대안으로 제시된 Sail Operator에 대해 소개합니다.

## Sail Operator란
Red Hat에서 시작된 커뮤니티 프로젝트로, 2025년 4월 3일 정식 GA 1.0.0 버전이 출시되었습니다.
이는 Helm 차트를 기반으로 하는 새로운 설치 방식으로, Classic Operator보다 간소화 및 자동화의 이점을 가졌습니다.

**Sail Operator가 관리하는 주요 리소스**
1. Istio: Control Plane관리
2. Istio Revision: Istio 버전 관리
3. IstioRevisionTag: Istio 별칭 관리
4. IstioCNI: CNI 노드 에이전트 관리
5. ZTunnel: 데몬셋을 통한 앰비언트 모드 관리

**업데이트 전략**
1. In Place: Control Plane을 즉시 신규 버전으로 업데이트
2. Revision Based: 신규 Control Plane에 'updateWorkloads' 플래그 설정으로 변경

## Helm으로 Sail Operator 설치하기
Helm을 사용해본 경험이 있다면 기본적인 방법은 동일합니다.

**1. Helm 레포 설치**
```
> helm repo add sail-operator https://istio-ecosystem.github.io/sail-operator

> helm repo update
```

**2. Helm 차트 설치**  
Sail Operator를 설치하고 성공적으로 완료된 것을 확인합니다.
```
> helm install sail-operator sail-operator/sail-operator --version 1.0.0 --namespace sail-operator --create-namespace
NAME: sail-operator
LAST DEPLOYED: Sun Apr 13 01:50:34 2025
NAMESPACE: sail-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
**3. Sail Operator 상태 확인**
```
> kubectl -n sail-operator get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP          NODE                  NOMINATED NODE   READINESS GATES
sail-operator-6bfc7856c9-pm6pb   1/1     Running   0          2m28s   10.10.0.8   myk8s-control-plane   <none>           <none>
```

**4. Istio 리소스 생성**  
istio-system 네임스페이스를 생성하고, Istio 리소스를 적용합니다.

```
 kubectl create namespace istio-system
> cat <<EOF | kubectl apply -f-
apiVersion: sailoperator.io/v1
kind: Istio
metadata:
  name: default
spec:
  version: v1.24.1
  updateStrategy:
    type: RevisionBased
    inactiveRevisionDeletionGracePeriodSeconds: 30
  namespace: istio-system
---
apiVersion: sailoperator.io/v1
kind: IstioRevisionTag
metadata:
  name: default
spec:
  targetRef:
    kind: Istio
    name: default
EOF
```

**5. IstioRevision 확인**  
Istio 리소스가 생성되면 sail operator는 자동으로 IstioRevision을 생성합니다.
```
> k get IstioRevision
NAME       NAMESPACE      READY   STATUS    IN USE   VERSION   AGE
default    istio-system   True    Healthy   False    v1.24.1   104s
```

### Sail Operator 차트별 Istio 지원 버전 확인
Sail Operator는 차트 버전별로 특정 Istio 버전만 지원하며, 모든 패치 버전이 포함되지 않을 수 있습니다. 지원되는 정확한 버전 목록은 공식 GitHub 저장소의 chart/values.yaml 파일에서 확인해야 합니다.

**차트 v1.0.0의 Istio 지원 버전 예시**
```
- v1.24-latest
- v1.24.3
- v1.24.2
- v1.24.1
- v1.23-latest
- v1.23.5
- v1.23.4
- v1.23.3
- v1.23.0
```

## 마치며
Istio는 빠른 업데이트 주기로 신규 버전이 출시되는데, Sail Operator를 통해 손쉽게 버전 관리를 진행할 수 있어 매우 편리하다고 생각합니다.  
특히, Sail Operator는 Red Hat의 주도로 빠르게 개발되고 있기 때문에, 향후 새로운 기능들이 많이 지원될 것으로 기대됩니다. 