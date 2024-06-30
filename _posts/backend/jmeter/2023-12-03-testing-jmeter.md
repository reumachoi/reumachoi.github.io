---
title: 서버의 성능을 JMeter로 테스트해보자
author: rumi
date: 2023-12-03
categories: [Backend, Test]
tags: [Jmeter, Testing]
---

## 소개

풀네임은 APACHE JMeter이며 자바 앱으로 만들어진 오픈소스 툴이다. 그렇기 때문에 JMeter를 실행하는 환경에는 Java와 JVM이 설치 되어있어야 한다.

---

## 설치 (방법 선택)

1. 홈페이지 소스 다운로드
   https://jmeter.apache.org/download_jmeter.cgi
   자바8 이상의 환경에서 Binaries / Source 를 받아 설치합니다
2. brew 이용

```
brew install jmeter
```

---

## 실행방법

- GUI 모드와 Non-GUI 모드 두가지 방식이 있습니다
- 테스트 실행계획은 GUI에서 생성 후 실제 테스팅은 Non-GUI 모드로 하는것을 공식사이트에서는 권장하는 것 같습니다.
  (https://jmeter.apache.org/usermanual/get-started.html)
- 타 블로그에서는 GUI모드가 높은 부하테스트를 실행하도록 설계되지 않아 과부하가 올 수 있다고 합니다. 실제로 저도 많은 양의 테스트를 GUI모드로 돌릴때 실행이 제대로 되지않고 화면이 멈추는 일이 여러번 발생했습니다.
  (https://www.blazemeter.com/blog/jmeter-non-gui-mode)

### GUI 모드

- jmeter를 설치한 폴더의 bin 디렉터리에서 jmeter를 실행시키면 GUI 창이 나타납니다

1. 마우스 우클릭으로 Thread Group을 생성해줍니다
2. Thread Group 하위에 Sampler와 Listener를 추가합니다
   (Sampler => HTTP Request, Listener => Assertion Results, View Results Tree)

- Smapler : 테스트를 할 대상의 타입
- Listener : 테스트 결과물 시각화 (Plugin Manager로 다양하게 선택 가능)
  ![이미지1]({{site.url}}{{site.baseurl}}/assets/img/2023-12-03-testing-jmeter/jmeter_1.png)
  ![이미지2]({{site.url}}{{site.baseurl}}/assets/img/2023-12-03-testing-jmeter/jmeter_2.png)

1. 설정파일 저장 (.jmx)

- 상단에 디스크 모양을 클릭해 위에서 세팅한 설정들을 저장합니다.

> .jmx 파일 본문
> ![이미지3]({{site.url}}{{site.baseurl}}/assets/img/2023-12-03-testing-jmeter/jmeter_3.png)

### Non-GUI 모드 (CLI)

- 저는 EC2 리눅스 서버에서 테스팅을 진행하여 설치과정 부터 실행 방법까지 정리해서 소개하겠습니다
- Java 설치, JMeter 설치, Plugins Manager 설치까지 진행합니다

```
sudo yum update -y
# 1. JMeter 설치전 사전준비
sudo yum install java-17-amazon-corretto -y

java -version

# 2. JMeter 설치
wget https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.6.2.tgz
tar -xf apache-jmeter-5.6.2.tgz
cd apache-jmeter-5.6.2/bin/
./jmeter -v

# 3. Plugins Manager 설치
# jmeter/lib/ext 폴더내에 설치
curl -O https://repo1.maven.org/maven2/kg/apc/jmeter-plugins-manager/1.6/jmeter-plugins-manager-1.6.jar

# jmeter/lib 폴더내에 cmdrunner-2.0.jar가 없는 경우 아래 실행후 저 위치로 이동
wget http://search.maven.org/remotecontent?filepath=kg/apc/cmdrunner/2.0/cmdrunner-2.0.jar

# jmeter/bin 폴더내에 PluginsManagerCMD.sh 또는 PluginsManagerCMD.bat가 없다면
java -cp jmeter/lib/ext/jmeter-plugins-manager-x.x.jar org.jmeterplugins.repository.PluginManagerCMDInstaller

# jmeter/bin 위치에 PluginsManagerCMD.sh 확인시 완료

# 4. jmx 파일에서 필요한 플러그인 추가 다운로드
./PluginsManagerCMD.sh install-for-jmx test.jmx
```

### 실행방법

- '-n' : Non-GUI 모드
- '-t' : 테스트 플랜이 담긴 jmx 파일명
- '-l' : 샘플 결과들을 저장할 jtl 파일명
- '-e' : 대시보드 보고서 생성
- '-o' : 대시보드 보고서 폴더명

```
# */jmeter/bin/jmeter.sh 위치에서 실행
./jmeter -n -t test.jmx -l <jtl-file-name> -e -o <folder-name>

```

---

## 분산환경 테스팅

- 대규모 트래픽정도의 조건에서 테스트를 로컬 PC에서 하려고하면 요청을 보내고 처리하는데에 한계가 있습니다. 위에서는 JMeter 서버를 한대로만 대상서버로 요청을 보냈지만 '분산환경 테스팅'은 JMeter 서버를 여러대를 두어 Test Plan을 실행하도록 합니다.

![이미지4]({{site.url}}{{site.baseurl}}/assets/img/2023-12-03-testing-jmeter/jmeter_4.png)

- 이미지 가운데에 '192.168.0.2' 서버가 Controller Node (=Master) 입니다
- 나머지 외곽에 있는 서버들은 전부 Worker Nodes (=Slave) 입니다
- 모든 서버에는 JMeter가 설치되어 있어야 합니다

### 서버 설정 (jmeter.properties 파일 수정)

- Java RMI 방식으로 통신하기 때문에 서로 인식하기 위한 포트 설정이 필요합니다
- SSL 설정을 사용하려면 **Controller** 서버에서 '/bin/create-rmi-keystore.sh'로 설정을 하고 만들어진 rmi_keystore.jks 파일을 **Worker** 서버들로 넣어서 준비해야합니다 (저는 비활성화 방식으로 진행했습니다)

**1. Controller(Master) 서버**

- remote_hosts = [172.xxx](http://172.xxx).xxx.xxx
- server.rmi.ssl.keystore.file (SSL 설정 활성화 시)
  - server.rmi.ssl.disable = true (SSL 설정 비활성화 시)
- client.rmi.localport = 4000 (b)
- server.rmi.port = 4000 (a)

**2. Worker(Slave) 서버**

- server.rmi.ssl.keystore.file
  - server.rmi.ssl.disable = true
- server.rmi.localport = 4000 (a)
- server.port = 4000 (b)

### 테스트 실행

1. 테스트를 하려는 모든 Worker 서버에서 jmeter-server를 실행

```
cd jmeter/bin
./jmeter-server

# 정상 연결 확인을 위해 jmeter.log 확인 (아래처럼 뜨면 성공)
Writing log file to: /XXXX/XXXXX/bin/jmeter-server.log
Created remote object: UnicastServerRef [liveRef: [endpoint:[192.X.X.X:XXXXX](local),objID:[-6a665beb:15a2c8b9419:-7fff, 3180474504933847586]]]

```

2. Controller 서버에서 Test Plan 실행

2-1. Worker 서버들을 설정하는 두가지 방법
a) Worker 서버들을 METER_HOME/bin/jmeter.properties 파일에 'remote_hosts' 표기

```
remote_hosts=172.0.0.1, 172.0.0.2...

```

b) '-R' 옵션으로 CLI 직접 표기

```
-R 172.0.0.1, 172.0.0.2
```

2-2. 테스트 실행

- '-r' : 2-1번의 a 방식으로 jmeter.properties 설정한 주소들을 그대로 사용하는 경우
- '-R' : 2-1번의 b 방식으로 주소들을 직접 명시하는 경우
- '-X' : 테스트 종료 후 Worker 서버들에서 jmeter-server 종료

```
./jmeter -n -t test.jmx -l log.jtl -e -o testing-results -r -X

./jmeter -n -t test.jmx -l log.jtl -e -o testing-results -R 172.0.0.1, 172.0.0.2 -X

```

### java.lang.OutOfMemoryError: Java heap space 트러블 슈팅

-> 기본 heap 사이즈 설정이 1GB로 되어있기 때문에 **JVM_ARGS** 설정을 통해 조정 가능

> Increase the Java Heap size. By default JMeter runs with a heap of 1 GB, this might not be enough for your test and depends on your test plan and number of threads you want to run

```
JVM_ARGS="-Xms2048m -Xmx2048m" ./jmeter -n -t test.jmx -l log.jtl -e -o testing-results -r -X
```

---

## 결과확인

- TroughPut이 우리가 흔히 알고있는 TPS(초당처리량)으로 확인하면 된다.

TPS를 높이기 위해서 Thread Group 설정을 여러번 바꿨으나 계속 비슷하게 나오는 경우가 있었다.
https://clarkshim.tistory.com/169 해당 블로그를 확인했을때 우리의 경우에는 '클라이언트 수가 너무 많다' 였던 것 같다.

결과를 기준으로 말하자면
Thread(users) 수를 높게 잡고(200정도) loop 수를 낮게(100) 설정했을때는 Troughput의 결과가 겨우 3000정도가 나왔다. Thread가 너무 많다보면 요청을 보내려는 **_수많은 스레드들간의 경쟁_**으로 인해 오히려 처리시간이 길어질 수 있어서 문제가 된 듯 했다.

일단 JMeter의 서버 1대와 대상서버 1대로 최소한의 상황에서 진행했다. **_대상서버의 CPU가 85% 내외정도_**로 도달하는 수치를 찾기위해 이전보다 **thread 수는 50**정도로 낮추고 **loop 수를 대폭늘려 1000**정도로 설정했다. 그리고 **ramp-up period 설정을 10**으로 설정해서 스레드가 10초 간격으로 하나씩 늘어나며 테스트하도록 설정했다(ramp-up period는 총실행시간에 포함되지 않는다). 이렇게 한 결과 Throughput이 이전 결과의 2배는 더 높게 나왔다!!

대상서버의 성능을 테스트하기 위해서는 적절한 thread, loop, ramp-up period 설정을 찾는게 관건인 듯 하다.

---

Artillery는 써봤지만 JMeter를 이렇게 다뤄본건 처음인것 같다. 요즘 좋은 성능테스트 툴도 많이 나오던데 JMeter는 문서도 친절하지 않은 편에 Distributed Testing 환경도 직접 구축해야하는게 너무 번거로웠다.
혹시나 저처럼 JMeter를 사용해야하는 분들께 조금이라도 도움이 되기를 바랍니다.

## 참고문서

[JMeter 공식문서](https://jmeter.apache.org/usermanual/get-started.html)
[Plugins Manager CLI 설치방법](https://jmeter-plugins.org/wiki/PluginsManagerAutomated/)
[분산테스팅 설명서](https://jmeter.apache.org/usermanual/jmeter_distributed_testing_step_by_step.html)
[분산테스팅 설명서2](https://jmeter.apache.org/usermanual/remote-test.html)
