---
title: Kubestronaut가 되기까지의 1년
author: rumi
date: 2025-01-19
categories: [Career, Certification]
tags: [Kubestronaut, Kubernetes, CKA, CKAD, CKS, KCNA, KCSA]
---

## Kubestronaut가 되었습니다
![kubestronaut-badge](/assets/img/posts/certification/kubestronaut-badge.png)

2023년에 쿠버네티스를 시작하면서 자격증의 존재를 알게 되었고, 사이버 먼데이에 CKA를 구매했습니다.
그리고 2024년 3월에 겨우 CKA를 취득했을 즈음, 팀원분이 Kubestronaut가 되셨다는 이야기를 듣고 저도 도전해 볼까 하는 마음으로 여기까지 오게 되었습니다.
그렇게 시작해서 5월에 CKAD, 6월에 KCNA, 10월에 KCSA, 그리고 12월에 CKS를 취득했습니다.

이 중에서 뭐가 제일 힘들었냐고 하면, 당연히 CKS였습니다.  
1년 안에 이 모든 걸 해낼 수 있을 거라고는 생각도 못 했는데, 다 끝내고 나니 정말 보람찬 1년이었습니다

## 자격증별 준비 방법

### CKA & CKAD
#### Udemy 강의 추천
[뭄샤드 강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/?gad_source=1&gclid=Cj0KCQiA4rK8BhD7ARIsAFe5LXJ1xNQgfcQy_BbgF4dmFWcrnDHzGoQb3TCK7lgkkwLTBmrUp5GPHL8aAoqDEALw_wcB&utm_campaign=WJ_conv_Pmax_individual_conversion_240805&utm_content=CKA&utm_medium=udemyads&utm_source=wj-gdn&utm_term=cka_241007&couponCode=KEEPLEARNING)  
-> 여러 번 듣고 실습을 진행하면 어렵지 않았습니다. 2025년 2월부터 시험 범위가 변경된다는 소식이 있으니, 강의 업데이트를 기다리거나 추가로 준비해야 할 수도 있습니다.

### KCNA & KCSA
#### Udemy 강의 추천
- 덤프 문제 4세트  
2025년 버전: [KCNA](https://www.udemy.com/course/kubernetes-and-cloud-native-associate-kcna-exam-questions/?couponCode=KEEPLEARNING), [KCSA](https://www.udemy.com/course/kubernetes-and-cloud-native-security-associate-cert-practice-exam-prep/?couponCode=KEEPLEARNING)  
-> KCNA는 강의 없이도 가능했지만, KCSA는 문제 유형 파악을 위해 유데미 강의를 구매해서 활용했습니다.

깃헙 레포 추천: https://paulyu.dev/article/kcsa-study-guide/, https://infrasec.sh/post/cncf-kcsa/

### CKS
> 검색하면 많은 덤프와 강의들이 나오지만, 제가 시험에서 마주했던 내용 중 앞에서 다루지 않은 항목을 정리했습니다.
#### 1. kubeadm-reconfigure
- kubeadm을 이용해 kubelet 설정을 변경하는 문제
  - [공식문서-Reflecting the kubelet changes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure/#applying-cluster-configuration-changes)
```
kubectl edit cm -n kube-system kubelet-config // only control-plane
kubeadm upgrade node phase kubelet-config
systemctl restart kubelet
```

#### 2. SBOM 탐지 - bom & trivy 사용
- [bom 공식레포](https://github.com/kubernetes-sigs/bom)
- [trivy 공식레포](https://github.com/aquasecurity/trivy)
```
# bom으로 SBOM 생성
bom generate --format json -o target.json -i nginx

# trivy로 SBOM 생성
trivy image --format cyclonedx --output result.json alpine:3.15
trivy image --format spdx --output result.spdx alpine:3.15

# trivy로 SBOM 탐지
trivy sbom /path/to/sbom_file

# trivy로 취약점 탐지 - 특정 등급과 코드 포함 확인
trivy image --severity HIGH,CRITICAL nginx | grep -E 'CVE-...|CVE-...'
```

#### 3. CiliumNetworkPolicy 사용
- [CiliumNetworkPolicy 공식문서](https://docs.cilium.io/en/latest/security/policy/language/#limit-ingress-egress-ports)
- [Cilium mTLS](https://docs.cilium.io/en/latest/network/servicemesh/mutual-authentication/mutual-authentication-example/)
  - mTLS 옵션에 대한 설명을 공식문서에서 찾기가 힘들기 때문에 알아두는게 좋습니다.
``` 
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l4-rule"
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints: # k8s Netpol과 비슷한 사용법
    - matchLabels:
        role: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
  ingressDeny: {} # Deny 옵션 이해
  egress:
    - toPorts:
      - ports:
        - port: "80"
          protocol: TCP
  authentication: # mTLS 옵션 이해
	mode: "required"
```
#### 4. ProjectedVolume 사용
- [공식문서](https://kubernetes.io/ko/docs/concepts/storage/projected-volumes/)
- `automountServiceAccountToken: false` 옵션이 생기면서, ServiceAccountToken 사용법에 대한 부분이 출제 됩니다.
```
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    volumeMounts:
    - name: token-vol
      mountPath: "/service-account"
      readOnly: true
  serviceAccountName: default
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```

#### 5. api-server & etcd tls 설정
- 특정 버전과 알고리즘 세트에 대한 매니페스트 수정 문제가 나왔습니다.
```
# apiserver
- --tls-min-version=VersionTLS12
- --tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256

# etcd
 - --cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
 - --tls-min-version=VersionTLS12
 ```

#### 6. Pod-Security-Standards
- PSS 조건에 걸려 오류가 발생한 워크로드가 속한 네임스페이스의 프로필을 확인하고, 조건을 만족하도록 수정해야 합니다. 저는 Restricted 프로필로 문제들을 푸는 경우가 많았습니다.
- [공식문서](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
```
# profile=Restricted 일경우 세팅
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
	containers:
	  - name: example
	    image: gcr.io/google-samples/node-hello:1.0
	    securityContext:
		    privileged: false
		    runAsNonRoot: true
	      runAsUser: 4000
	      capabilities:
            drop:
                - ALL
            add: ["CHOWN"]
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
```

## 추천하나요?
투자했던 금액, 시간, 그리고 노력을 생각하면 정말 포기하고 싶었던 순간도 많았습니다. 하지만 쿠버네티스를 실제로 다루며 자격증을 준비했기 때문에, 이 과정이 서로 상호작용하며 큰 도움이 되었습니다.
5개 모두를 취득하는 것을 굳이 추천하지는 않지만, 쿠버네티스에 조금이라도 관심이 있거나 이미 시작하신 분들께는 꼭 **포기하지 말고 1개라도 취득해 보시라고** 말씀드리고 싶습니다.

이 과정은 단순히 시험을 준비하기 위한 공부가 아니라, 쿠버네티스를 깊이 이해하고 실무에 활용할 수 있는 다양한 내용들을 다루고 있습니다. 후회하지 않을 경험이 될 것이라고 생각합니다.
행운을 빕니다!

