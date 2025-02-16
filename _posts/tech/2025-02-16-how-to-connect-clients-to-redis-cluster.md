---
title: Redis Cluster를 이해하고 Client 연결하기
author: rumi
date: 2025-02-16
categories: [Tech]
tags: [redis, redis-client, redis-cluster, go-redis, redis-py, ioredis]
---

## Redis Cluster의 데이터 분배 방식
-  **초기 배정**
   -  Redis Cluster에서 데이터는 키의 해시값을 16384개의 슬롯 중 하나로 매핑하여 저장하고, 각 슬롯은 하나의 샤드에 할당됩니다. 이를 통해 키가 고르게 분산될 수 있습니다.
      - CRC16 알고리즘: 키의 CRC16 해시 값을 16,384로 나눈 나머지 값으로 슬롯값 결정합니다.
- **업데이트**
  - 새로운 샤드가 추가되거나 기존 샤드가 제거될 경우, 슬롯은 기존/ 신규 샤드로 자동으로 재배치됩니다.
  - 슬롯 재배치는 점진적으로 수행되며, 클러스터는 요청을 계속 처리할 수 있습니다.

## Redis Cluster에서 클라이언트 요청 처리 과정
- **신규 연결**
  - 클라이언트는 연결 시 `CLUSTER SLOTS` 커맨드를 사용하여 슬롯과 샤드의 맵핑 정보를 가져옵니다. 각 슬롯 범위와 이를 관리하는 샤드의 IP와 포트를 반환하며, 클라이언트는 해당 정보를 캐싱하여 요청 시 활용합니다.
- **슬롯 업데이트 중**
  - 기존 키의 슬롯이 이동 중일 경우 `(error) ASK` 응답이 발생합니다. 클라이언트는 해당 키를 새로운 노드에 임시로 요청합니다. 이 과정에서 클러스터는 요청을 처리할 준비가 되어 있습니다.
- **슬롯 업데이트 완료**
  - 클라이언트가 요청한 키가 기존과 다른 슬롯으로 완전히 이동되었을 경우, 클러스터는 `(error) MOVED` 응답과 함께 새로운 샤드 주소를 반환합니다. 클라이언트는 이를 수신한 후 새 주소로 요청을 재전송합니다.
- **클러스터 지원 라이브러리**
  - 클러스터를 지원하는 라이브러리는 'MOVED' 응답을 처리하여 자동으로 새로운 샤드로 요청을 재전송하고, 슬롯 맵 정보를 업데이트합니다. 반면, 클러스터를 지원하지 않는 라이브러리는 이를 수동으로 처리해야 하므로 비효율적일 수 있습니다.
  




## Redis 클러스터 상태 처리 코드 예제 (언어별)
###  언어별 클러스터 지원 라이브러리
- Java: Jedis
- .NET: StackExchange.Redis
- Go: Radix, go-redis/redis
- Node.js: node-redis, ioredis
- Python: redis-py

### [redis-py] 로직 알아보기
```
 def _execute_command(self, target_node, *args, **kwargs):
        """
        Send a command to a node in the cluster
        """
        command = args[0]
        redis_node = None
        connection = None
        redirect_addr = None
        asking = False
        moved = False
        ttl = int(self.RedisClusterRequestTTL)

        while ttl > 0:
            ttl -= 1
            try:
                if asking:
                    target_node = self.get_node(node_name=redirect_addr)
                elif moved:
                    # MOVED occurred and the slots cache was updated,
                    # refresh the target node
                    slot = self.determine_slot(*args)
                    target_node = self.nodes_manager.get_node_from_slot(
                        slot, self.read_from_replicas and command in READ_COMMANDS
                    )
                    moved = False

                redis_node = self.get_redis_connection(target_node)
                ...
```
### [go-redis] 로직 알아보기
- MOVED 발생시 해당 함수 호출로 맵 갱신
```
func (c cmdable) ClusterSlots(ctx context.Context) *ClusterSlotsCmd {
	cmd := NewClusterSlotsCmd(ctx, "cluster", "slots")
	_ = c(ctx, cmd)
	return cmd
}
```
### [ioredis] 로직 알아보기
```
this.refreshSlotsCache((err) => {
            if (err && err.message === ClusterAllFailedError.defaultMessage) {
              Redis.prototype.silentEmit.call(this, "error", err);
              this.connectionPool.reset([]);
            }
});
```


## 운영환경 유의점
'MOVED'와 'ASK' 에러 이해하기
- Redis 클러스터에서 자주 발생하는 'MOVED'와 'ASK' 에러는 클러스터 내에서 슬롯 이동이나 상태 변경으로 인해 발생합니다. 클러스터와 클라이언트 간의 호환성 문제가 발생할 수 있으므로, 이러한 에러와 클러스터 처리 방식을 충분히 이해하고 이를 처리하는 로직을 구현하는 것이 중요합니다.

레디스 라이브러리 선택 시 클러스터 지원 고려하기
- Redis 클러스터를 사용하려면 클러스터를 지원하는 라이브러리로 전환이 필요합니다. 클러스터 환경에서의 데이터 처리 및 슬롯 관리 기능을 갖춘 라이브러리를 선택하여, 시스템의 확장성과 안정성을 높여야 합니다.

레디스 구성 교체 시 애플리케이션 호환성 확인
- 단일 Redis에서 Redis Cluster로의 교체 시, 기존 클라이언트 애플리케이션들이 클러스터 환경과 호환되는지 점검해야 합니다. 클러스터로의 전환을 준비하며 기존 코드에서의 클러스터 지원 여부와 데이터 분배 및 관리 로직을 재검토가 필요합니다.
  

  
---

### 참고문서
[공식문서 redis-cluster](https://redis.io/learn/operate/redis-at-scale/scalability/redis-cluster-and-client-libraries)  
[redis-py](https://redis.readthedocs.io/en/stable/_modules/redis/cluster.html#RedisCluster)  
[go-redis](https://github.com/redis/go-redis/blob/master/cluster_commands.gos)  
[ioredis](https://github.com/redis/ioredis/blob/main/lib/cluster/index.ts)
