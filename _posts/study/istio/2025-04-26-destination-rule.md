---
title: Istio에서 DestinationRule로 트래픽 제어 및 복원성 확보하기
author: rumi
date: 2025-04-26
categories: [Study, Istio]
tags: [Kubernetes, Istio, Service Mesh, Destination Rule]
---

# 개요
Istio에서 트래픽 제어와 복원성 강화를 위해 반드시 이해해야 할 핵심 리소스가 바로 DestinationRule입니다.
이 글에서는 DestinationRule의 주요 기능 설명과 매니페스트 예시까지 함께 소개하도록 하겠습니다.

## DestinationRule 소개
매니페스트 스펙에서 관리되는 필드 5개를 소개합니다.
1. `host`
   - example.com 같은 일반 도메인 도는 Kubernetes svc 도메인을 사용합니다.
   - 서비스명만 사용시 기본 네임스페이스(default)로 해석되기 때문에, 완전한 FQDN을 명시적으로 기재하는 것이 권장됩니다. (예: 'review' → 'review.default.svc.cluster.local')
2. `trafficPolicy`
   -  부하 분산, 커넥션 풀 조정, 서킷브레이커 등을 설정합니다. 
3. `subsets`
    - 서비스별 레이블을 사용해서 버전별 트래픽 정책을 관리할 수 있습니다. 
4. `exportTo`
    - 트래픽을 전송할 네임스페이스를 지정합니다.
5. `workloadSelector`
    - 동일 네임스페이스 내 일치하는 레이블을 가진 워크로드로 제한해서 설정합니다.

### subsets
버전별로 다른 트래픽 정책을 설정하는 경우 아래와 같이(`.spec.subsets[*].trafficPolicy`) 설정 가능합니다. 
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
  subsets:
  - name: testversion-v1
    labels:
      version: v1
    trafficPolicy: 
      simple: ROUND_ROBIN
  - name: testversion-v2
    labels:
      version: v2
```

## TrafficPolicy 상세 설명
하위 6개의 설정들로 정책을 관리합니다.
1. `loadBalancer`: 부하분산 알고리즘을 설정합니다.
2. `connectionPool`: 업스트림으로의 커넥션 수를 조정합니다.
3. `outlierDetection`: 비정상 호스트를 제외시킵니다.
4. `tls`: 업스트림 연결 과정에서 TLS설정을 합니다.
5. `portLevelSettings`: 포트 개별 설정을 합니다.
6. `tunnel`: TCP, TLS 터널링을 구성합니다.

### LoadBalancer
**필드**
- `simple`: 부하분산 알고리즘 방식을 설정합니다. 기본값은 'ROUND_ROBIN'으로 설정에서 생략됩니다.
  - RANDOM, LEAST_REQUEST 등의 방식들이 있고 워크로드에 맞는 방식을 선택할 수 있습니다.
- `consistentHash`: 특정 키(예: 헤더, 쿠키, 소스IP)에 기반하여 요청을 일관된 업스트림으로 라우팅합니다. 스티키 세션과 유사한 동작을 구현할 수 있습니다.
- `localityLbSetting`: 요청하는 곳의 가용영역과 같은 가용영역에 위치하는 호스트로 라우팅 되도록 합니다.
- `warmup`: 추가된 신규 서비스가 처음 트래픽을 받기까지의 워밍업 기간을 설정합니다. ('warmupDurationSecs'가 대체되었습니다.)


**예시**
```
spec:
  trafficPolicy:
    loadBalancer:
        simple: LEAST_REQUEST
        consistentHash:
            httpCookie:
              name: user
              ttl: 0s
        localityLbSetting:
            distribute:
            - from: us-west/zone1/* # 가중치 미설정시 zone1로 100% 적용
              to:
                "us-west/zone1/*": 80
                "us-west/zone2/*": 20
            - from: us-west/zone2/* 
              to:
                "us-west/zone1/*": 20
                "us-west/zone2/*": 80
        warmup: ...
```


### connectionPool
**필드**
- `tcp`: 커넥션 풀을 설정합니다. HTTP, TCP 모두 공통 설정.
- `http`: HTTP에만 커넥션 풀을 설정합니다.

**예시**
```
spec:
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100 
        connectTimeout: 30ms 
        tcpKeepalive:
          time: 7200s
          interval: 75s
```

### OutlierDetection
'서킷 브레이커'를 설정하기 위한 조건들을 설정합니다.

**필드**  
- `interval`: 상태 감지 후 호스트를 정리하는 간격
- `consecutive5xxErrors`: 호스트를 풀에서 제거하기 위한 5xx 오류 수
...


**예시**
```
spec:
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 7
      interval: 5m
```


### TLS
**필드**  
- `mode`: TLS 적용방식 설정(필수 설정 항목)

**예시**
```
spec:
  trafficPolicy:
    tls:
      mode: MUTUAL # MTLS 설정
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

### TrafficPolicy 실습
#### 1. locality 기반 동일 가용영역 우선 라우팅 검증(localityLbSetting)
**locality 설정 확인**

환경은 아래 코드를 통해 virtualservice, service, gateway, destinationrule을 설정을 확인해주세요.  
현재 web에서 backend로 향하는 DestinationRule에 가용영역 a:b=70:30 비율로 설정을 해둔 상태입니다.
```
> k get vs,svc,gw
NAME                                                           GATEWAYS                 HOSTS                             AGE
virtualservice.networking.istio.io/simple-web-vs-for-gateway   ["simple-web-gateway"]   ["simple-web.istioinaction.io"]   16m

NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/simple-backend   ClusterIP   10.200.1.179   <none>        80/TCP    16m
service/simple-web       ClusterIP   10.200.1.24    <none>        80/TCP    16m

NAME                                             AGE
gateway.networking.istio.io/simple-web-gateway   16m     ClusterIP   10.200.1.24    <none>        80/TCP    14m

> k get dr
NAME                HOST                                             AGE
simple-backend-dr   simple-backend.istioinaction.svc.cluster.local   17m

> k get dr -o jsonpath='{.items[*].spec.trafficPolicy.loadBalancer}'
{"localityLbSetting":{"distribute":[{"from":"us-west1/us-west1-a/*","to":{"us-west1/us-west1-a/*":70,"us-west1/us-west1-b/*":30}}]}}
```

**kiali 확인**  
위쪽 70% 비율로 트래픽이 전달되는 워크로드가 가용영역 'us-west1-a'인 것을 알 수 있습니다.
![kiali-실습-locality 사진](/assets/img/posts/istio/dr-example-locality.png)

#### 2. 비정상 호스트 자동 차단 검증(OutlierDetection)
**이상탐지 설정 및 호스트 제외 확인**  
아래 DestinationRule에 설정한 outlierDetection 조건에 걸리는 경우, 엔드포인트 'OUTLIER CHECK=FAILED'로 설정됩니다. 그리고 호스트 풀에서 제외됩니다.
```
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: simple-backend-dr
spec:
  host: simple-backend.istioinaction.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      baseEjectionTime: 5s # 호스트 풀에서 제거하는데 소요되는 시간.
      consecutive5xxErrors: 5 # 기본값 5. 5개 이후부터 이상탐지.
      interval: 10s # 기본값 10s. 10초마다 확인.
      maxEjectionPercent: 10 # 기본값 10. 제거가능 호스트 비율.
```
```
> docker exec -it myk8s-control-plane istioctl proxy-config endpoint deploy/simple-web.istioinaction --cluster 'outbound|80||simple-backend.istioinaction.svc.cluster.local'
ENDPOINT            STATUS      OUTLIER CHECK     CLUSTER
10.10.0.23:8080     HEALTHY     OK                outbound|80||simple-backend.istioinaction.svc.cluster.local
10.10.0.26:8080     HEALTHY     FAILED            outbound|80||simple-backend.istioinaction.svc.cluster.local
```

**kiali 확인**  
에러비율이 높았던 호스트가 제외되고는 정상 호스트로 라우팅 되었습니다.
![kiali-실습-outlier-detection 사진](/assets/img/posts/istio/dr-example-outlier-detection-error-rate.png)


#### 3. 커넥션 풀 제어 검증(connectionPool)

**커넥션 풀 설정 및 요청**  
1개의 연결만 허용하고, 재시도나 대기 요청도 1개만 허용하는 조건입니다.  
여기서 조건을 초과하는 초당 2개의 요청을 보내는 경우를 확인합니다.
```
> fortio load -quiet -jitter -t 30s -c 2 -qps 2 --allow-initial-errors http://simple-web.istioinaction.io:30000

---
# destinationrule.yaml
  spec:
    host: simple-backend.istioinaction.svc.cluster.local
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 1
          http2MaxRequests: 1
          maxRequestsPerConnection: 1
          maxRetries: 1
        tcp:
          maxConnections: 1
```

**요청 및 커넥션풀 결과**  
요청과 통계의 오차는 약간 있지만, 커넥션 풀 조건을 초과했다는 것을 알 수 있습니다.
```
# 클라이언트 측
Code 200 : 31 (51.7 %)
Code 500 : 29 (48.3 %)

# istio 통계
cluster.outbound|80||simple-backend.istioinaction.svc.cluster.local.upstream_cx_active: 1
cluster.outbound|80||simple-backend.istioinaction.svc.cluster.local.upstream_cx_total: 73
cluster.outbound|80||simple-backend.istioinaction.svc.cluster.local.upstream_cx_max_requests: 73
cluster.outbound|80||simple-backend.istioinaction.svc.cluster.local.upstream_cx_overflow: 45
```

## 마치며
기존에 익히 들었던 Istio의 기능들이 DestinationRule을 통해 제어된다는 사실을 알게되었습니다.
특히 실습을 진행한 3가지에 대해서는, 운영환경에서 가용영역 간 트래픽 비용 증가 문제나 서버별 커넥션&부하 한계를 관리할 때, Istio 없이 인프라만으로 해결하기 어려웠던 경험이 있었습니다.
이번에 DestinationRule의 주요 기능을 이해해서, 향후 비슷한 상황이 예상되는 경우 도입을 고려하거나 이미 구축된 환경에서 더욱 효과적으로 활용할 수 있을 것이라고 기대됩니다.

## 참고문서
[Istio 공식문서-DestinationRule](https://istio.io/v1.17/docs/reference/config/networking/destination-rule/#TrafficPolicy)