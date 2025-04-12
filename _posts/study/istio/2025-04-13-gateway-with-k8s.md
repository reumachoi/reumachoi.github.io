---
title: Istio Gateway를 Kubernetes API로 설정하기
author: rumi
date: 2025-04-13
categories: [Study, Istio]
tags: [Kubernetes, Istio, Service Mesh, Gateway API]
---

# 개요
Kubernetes Gateway를  Istio Gateway Controller와 함께 구성하여 구현하는 방법을 설명합니다.

## Kubernetes Gateway 소개
Gateway API는 Kubernetes에서 서비스 네트워킹을 모델링하기 위한 인터페이스로, Kubernetes Ingress의 차세대 버전으로 볼 수 있습니다. 
![kubernetes-gateway-api-role-tree](/assets/img/posts/istio/kubernetes-gateway-api-role-tree.png)


**핵심 목표**
- **역할 분리**: 인프라 제공자/클러스터 운영자/개발자 권한 분리
- **프로토콜 확장**: HTTP, TCP, UDP, gRPC 등 L4~L7 지원  
- **표준화**: 다양한 인그레스 컨트롤러(Istio, NGINX, Cilium) 호환 

**API 그룹** 
- `gateway.networking.k8s.io/v1`  
  
**주요 리소스**  
- GatewayClass: 게이트웨이 컨트롤러 유형 정의 (예: `istio`, `nginx`)
- Gateway: 실제 트래픽 진입점(IP/포트) 및 프로토콜 설정   
- HTTPRoute: 호스트/경로 기반 HTTP 트래픽 라우팅 규칙  


## Istio Gateway Controller
Kubernetes Gateway API의 구현체로, Istio 1.20+부터 공식 지원을 시작했습니다.

> **실습환경**  
> Istio: 1.25.1  
> kind: v1.32.2

### 1. Kubernetes Gateway CRD 설치  
Gateway API CRD가 클러스터에 설치되어 있지 않다면 다음 명령어로 설치합니다.
```
> kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

> kubectl get crd | grep .gateway.
gatewayclasses.gateway.networking.k8s.io    2025-04-10T12:41:23Z
gateways.gateway.networking.k8s.io          2025-04-10T12:41:23Z
grpcroutes.gateway.networking.k8s.io        2025-04-10T12:41:23Z
httproutes.gateway.networking.k8s.io        2025-04-10T12:41:23Z
referencegrants.gateway.networking.k8s.io   2025-04-10T12:41:23Z

> k get gatewayclass
NAME           CONTROLLER                    ACCEPTED   AGE
istio          istio.io/gateway-controller   True       48m
```
### 2. Gateway 및 HTTPRoute 리소스 생성  
httpbin 서비스로 트래픽을 라우팅하는 기본적인 Gateway와 HTTPRoute 구성입니다.  
`spec.gatewayClassName`: **istio**를 지정하여 Istio Gateway Controller가 이 Gateway를 관리하도록 합니다.
```
> kubectl create namespace istio-ingress
> kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  gatewayClassName: istio 
  listeners:
  - name: http
    hostname: "httpbin.example.com"
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin
spec:
  parentRefs:
  - name: httpbin-gateway
  hostnames: ["httpbin.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    - path:
        type: PathPrefix
        value: /headers
    backendRefs:
    - name: httpbin
      port: 8000
EOF
```
### 3. 테스트용 서비스 배포
샘플 서비스인 httpbin을 배포합니다.
```
> curl -O https://raw.githubusercontent.com/istio/istio/1.25.1/samples/httpbin/httpbin.yaml
> kubectl apply -f httpbin.yaml
```

### 4. 서비스로 요청 보내기
```
# 추후 외부 요청을 위해 NodePort로 변경해두기(기존 LoadBalancer)
> kubectl annotate gateway -n istio-ingress gateway networking.istio.io/service-type=NodePort --overwrite

# NodePort 30000번으로 직접 설정
> kubectl patch svc -n istio-ingress gateway-istio -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 80, "nodePort": 30000}]}}'

# 확인
> kubectl get svc -n istio-ingress gateway-istio
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                        AGE
gateway-istio   NodePort   10.200.1.202   <none>        15021:31191/TCP,80:30000/TCP   64m

# 트래픽 확인
> curl -s -I -HHost:httpbin.example.com "http://127.0.0.1:30000/get"
> curl -s -HHost:httpbin.example.com "http://127.0.0.1:30000/get"
```
### 5. kiali로 트래픽 확인
istio-ingress 네임스페이스에 있는 Gateway를 통해, default 네임스페이스에 있는 Service로 트래픽이 들어오는걸 확인할 수 있습니다.
![](/assets/img/posts/istio/istio-kubernetes-gateway-kiali.png)

## 마치며
다양한 Gateway Controller 사용 가능성을 고려하거나 Istio의 고급 기능(트래픽 분할/미러링 등)이 필요하지 않은 경우, Kubernetes Gateway API를 통해 표준화된 역할 기반 접근으로 인프라 관리 부담을 줄이면서도 세밀한 라우팅 제어가 가능한 방법이라고 생각됩니다.

## 참고문서
[Kubernetes Gateway API 공식](https://gateway-api.sigs.k8s.io/)  
[Istio Ingress Gateways 설치 가이드](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/)