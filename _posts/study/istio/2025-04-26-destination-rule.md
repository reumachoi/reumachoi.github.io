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
1. host
   - example.com 같은 일반 도메인 도는 Kubernetes svc 도메인을 사용합니다.
   - 서비스명만 사용시 기본 네임스페이스(default)로 해석되기 때문에, 완전한 FQDN을 명시적으로 기재하는 것이 권장됩니다. (예: 'review' → 'review.default.svc.cluster.local')
2. trafficPolicy
   -  부하 분산, 커넥션 풀 조정, 서킷브레이커 등을 설정합니다. 
3. subsets
    - 서비스별 레이블을 사용해서 버전별 트래픽 정책을 관리할 수 있습니다. 
4. exportTo
    - 트래픽을 전송할 네임스페이스를 지정합니다.
5. workloadSelector
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

### TrafficPolicy 설명
하위 6개의 설정들로 정책을 관리합니다.
1. loadBalancer: 부하분산 알고리즘을 설정합니다.
2. connectionPool: 업스트림으로의 커넥션 수를 조정합니다.
3. outlierDetection: 비정상 호스트를 제외시킵니다.
4. tls: 업스트림 연결 과정에서 TLS설정을 합니다.
5. portLevelSettings: 포트 개별 설정을 합니다.
6. tunnel: TCP, TLS 터널링을 구성합니다.

#### LoadBalancer
**필드**
- simple: 부하분산 알고리즘 방식을 설정합니다. 기본값은 'ROUND_ROBIN'으로 설정에서 생략됩니다.
  - RANDOM, LEAST_REQUEST 등의 방식들이 있고 워크로드에 맞는 방식을 선택할 수 있습니다.
- consistentHash: 특정 키(예: 헤더, 쿠키, 소스IP)에 기반하여 요청을 일관된 업스트림으로 라우팅합니다. 스티키 세션과 유사한 동작을 구현할 수 있습니다.
- localityLbSetting: 요청하는 곳의 가용영역과 같은 가용영역에 위치하는 호스트로 라우팅 되도록 합니다.
- warmup: 추가된 신규 서비스가 처음 트래픽을 받기까지의 워밍업 기간을 설정합니다. ('warmupDurationSecs'가 대체되었습니다.)


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


#### connectionPool
**필드**
- tcp: 커넥션 풀을 설정합니다. HTTP, TCP 모두 공통 설정.
- http: HTTP에만 커넥션 풀을 설정합니다.

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

#### OutlierDetection
'서킷 브레이커'를 설정하기 위한 조건들을 설정합니다.

**필드**  
- interval: 상태 감지 후 호스트를 정리하는 간격
- consecutive5xxErrors: 호스트를 풀에서 제거하기 위한 5xx 오류 수
...


**예시**
```
spec:
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 7
      interval: 5m
```


#### TLS
**필드**  
- mode: TLS 적용방식 설정(필수 설정 항목)

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

### TrafficPolicy 실습(TODO)
#### 1. locality 기반 동일 가용영역 우선 라우팅 검증(localityLbSetting)
#### 2. 비정상 호스트 자동 차단 검증(OutlierDetection)
#### 3. 커넥션 풀 제어 검증(connectionPool)

## 마치며
기존에 익히 들었던 Istio의 기능들이 DestinationRule을 통해 제어된다는 사실을 알게되었습니다.
특히 실습을 진행한 3가지에 대해서는, 운영환경에서 가용영역 간 트래픽 비용 증가 문제나 서버별 커넥션&부하 한계를 관리할 때, Istio 없이 인프라만으로 해결하기 어려웠던 경험이 있었습니다.
이번에 DestinationRule의 주요 기능을 이해해서, 향후 비슷한 상황이 예상되는 경우 도입을 고려하거나 이미 구축된 환경에서 더욱 효과적으로 활용할 수 있을 것이라고 기대됩니다.

## 참고문서
[Istio 공식문서-DestinationRule](https://istio.io/v1.17/docs/reference/config/networking/destination-rule/#TrafficPolicy)