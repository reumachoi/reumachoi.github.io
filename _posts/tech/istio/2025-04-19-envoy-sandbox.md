---
title: Envoy Sandbox로 손쉽게 필터 기능 확인해보기
author: rumi
date: 2025-04-19
categories: [Tech, Istio]
tags: [Istio, Envoy, Sandbox, Filter, CORS, Redis, MySQL]
---

# 개요
- Envoy에서 제공하는 샌드박스 Docker Compose를 사용해서 특정 기능들을 확인하는 환경을 쉽게 설정합니다.
- 해당 포스팅에서는 CORS Filter, Redis Filter, MySQL Filter 기능을 샌드박스 환경에서 확인하고자 합니다.

## Sandbox 환경 구성방법
> [공식문서 설정방법](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/setup#start-sandboxes-setup)
1. Docker 설치
2. Docker Compose 설치
3. Git 설치
4. Envoy 예제 레파지토리 클론
- `/envoy/examples` 경로에서 아래 실습을 시작합니다.

## a) CORS Filter

### 0. envoy 설정 확인
- `/envoy/examples/cors/backend/envoy.yaml`
  - "/cors/disabled"는 CORS 비활성화
  - "/cors/restricted"는 envoyproxy.io 오리진 확인  

```
route_config:
  name: local_route
  virtual_hosts:
  - name: www
    domains:
    - "*"
    ...
    routes:
    - match:
        prefix: "/cors/open"
      route:
        cluster: backend_service
    - match:
        prefix: "/cors/disabled"
      route:
        cluster: backend_service
      typed_per_filter_config:
        envoy.filters.http.cors:
          "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.CorsPolicy
          filter_enabled:
            default_value:
              numerator: 0
              denominator: HUNDRED
    - match:
        prefix: "/cors/restricted"
      route:
        cluster: backend_service
      typed_per_filter_config:
        envoy.filters.http.cors:
          "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.CorsPolicy
          allow_origin_string_match:
          - safe_regex:
              regex: .*\.envoyproxy\.io
          allow_methods: "GET"
    - match:
        prefix: "/"
      route:
        cluster: backend_service
http_filters:
- name: envoy.filters.http.cors
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors   
```

### 1. frontend 실행
```
# /envoy/examples/cors/frontend
> docker compose pull
> docker compose up --build -d
> docker compose ps
```

### 2. backend 실행
- 여기까지 총 4개의 컨테이너가 실행됩니다.

```
# /envoy/examples/cors/backend
> docker compose pull
> docker compose up --build -d
> docker compose ps
```

### 3. 기능 테스트
- `http://localhost:8000` 경로별 테스트
- CORS Enforcement를 변경하며 경로별 CORS 필터를 확인합니다.
  - Disabled, Restricted는 의도대로 실패합니다.
![envoy-cors](/assets/img/posts/istio/envoy-cors.png)


### 4. 결과 확인
- `http://localhost:8003/stats` 통계 확인  

```
http.ingress_http.cors.origin_invalid: 7
http.ingress_http.cors.origin_valid: 6
```  


## b) Redis Filter

### 0. envoy 설정 확인

- `/envoy/examples/redis/envoy.yaml` 

```
filter_chains:
- filters:
  - name: envoy.filters.network.redis_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.network.redis_proxy.v3.RedisProxy
      stat_prefix: egress_redis
      settings:
        op_timeout: 5s
      prefix_routes:
        catch_all_route:
          cluster: redis_cluster   
```

### 1. Redis 컨테이너 실행
```
# /envoy/examples/redis
> docker compose pull
> docker compose up --build -d
> docker compose ps
```
### 2. Redis 커맨드 실행
```
> docker run --rm --network host redis:latest redis-cli -h localhost -p 1999 set foo foo
> docker run --rm --network host redis:latest redis-cli -h localhost -p 1999 set bar bar
> docker run --rm --network host redis:latest redis-cli -h localhost -p 1999 get foo
> docker run --rm --network host redis:latest redis-cli -h localhost -p 1999 get bar
```
### 3. egress 통계 확인  
- `http://localhost:8001/stats?usedonly&filter=redis.egress_redis.command`  
- 필터에서 설정한 'stat_prefix=egress_redis' 이름으로 통계를 확인합니다.  

```
redis.egress_redis.command.get.success: 2
redis.egress_redis.command.get.total: 2
redis.egress_redis.command.set.success: 2
redis.egress_redis.command.set.total: 2
redis.egress_redis.command.get.latency: P0(nan,0) P25(nan,0) P50(nan,0) P75(nan,0) P90(nan,0) P95(nan,0) P99(nan,0) P99.5(nan,0) P99.9(nan,0) P100(nan,0)
redis.egress_redis.command.set.latency: P0(nan,0) P25(nan,0) P50(nan,0) P75(nan,0) P90(nan,0) P95(nan,0) P99(nan,0) P99.5(nan,0) P99.9(nan,0) P100(nan,0)
```


## c) MySQL Filter
### 0. envoy 설정 확인
- `/envoy/examples/mysql/envoy.yaml`
- 'mysql_proxy', 'tcp_proxy' 2개를 같이 사용하고, 'egress_mysql', 'mysql_tcp' 통계를 얻습니다.

```
filter_chains:
- filters:
  - name: envoy.filters.network.mysql_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.network.mysql_proxy.v3.MySQLProxy
      stat_prefix: egress_mysql
  - name: envoy.filters.network.tcp_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
      stat_prefix: mysql_tcp
      cluster: mysql_cluster   
```


### 1. MySQL 컨테이너 실행
```
# /envoy/examples/redis
> docker compose pull
> docker compose up --build -d
> docker compose ps
```

### 2. MySQL 커맨드 실행
```
> docker run --rm -it --network mysql_default mysql:5.7 mysql -h proxy -P 1999 -u root --skip-ssl

mysql> CREATE DATABASE test;
mysql> USE test;
mysql> CREATE TABLE test ( text VARCHAR(255) );
mysql> SELECT COUNT(*) FROM test;
+----------+
| COUNT(*) |
+----------+
|        0 |
+----------+
mysql> INSERT INTO test VALUES ('hello, world!');
mysql> SELECT COUNT(*) FROM test;
+----------+
| COUNT(*) |
+----------+
|        1 |
+----------+
mysql> exit

```

### 3. egress 통계 확인
- 'queries_parse_error'로 실패한 쿼리 개수도 알 수 있습니다.  

```
curl -s "http://localhost:8001/stats?filter=egress_mysql"
mysql.egress_mysql.auth_switch_request: 1
mysql.egress_mysql.decoder_errors: 0
mysql.egress_mysql.login_attempts: 1
mysql.egress_mysql.login_failures: 0
mysql.egress_mysql.protocol_errors: 0
mysql.egress_mysql.queries_parse_error: 2
mysql.egress_mysql.queries_parsed: 9
mysql.egress_mysql.sessions: 1
mysql.egress_mysql.upgraded_to_ssl: 0
```

### 4. TCP 통계 확인
```
tcp.mysql_tcp.downstream_cx_no_route: 0
tcp.mysql_tcp.downstream_cx_rx_bytes_buffered: 0
tcp.mysql_tcp.downstream_cx_rx_bytes_total: 537
tcp.mysql_tcp.downstream_cx_total: 1
tcp.mysql_tcp.downstream_cx_tx_bytes_buffered: 0
tcp.mysql_tcp.downstream_cx_tx_bytes_total: 889
tcp.mysql_tcp.downstream_flow_control_paused_reading_total: 0
tcp.mysql_tcp.downstream_flow_control_resumed_reading_total: 0
tcp.mysql_tcp.early_data_received_count_total: 0
tcp.mysql_tcp.idle_timeout: 0
tcp.mysql_tcp.max_downstream_connection_duration: 0
tcp.mysql_tcp.upstream_flush_active: 0
tcp.mysql_tcp.upstream_flush_total: 0
```

## 마치며
- Envoy의 샌드박스 환경을 통해 테스트하려는 필터를 간편하게 구성하고 검증할 수 있었습니다. 특히, 자주 마주치는 CORS 오류를 Envoy에서 직접 설정해보며, 별도의 API Gateway 없이도 애플리케이션 단에서 유연하게 제어할 수 있다는 가능성을 확인할 수 있었습니다.

## 참고문서
[Envoy Sandbox](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/)  
[HTTP Filter - CORS](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/cors/v3/cors.proto#envoy-v3-api-msg-extensions-filters-http-cors-v3-corspolicy)  
[Network Filter - Redis proxy](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/redis_proxy_filter#config-network-filters-redis-proxy)  
[Network Filter - MySQL proxy](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/mysql_proxy_filter#config-network-filters-mysql-proxy)  
[Network Filter - TCP proxy](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/tcp_proxy_filter)