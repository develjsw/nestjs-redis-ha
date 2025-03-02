### Redis Cluster 가용성 테스트

- Docker Redis Cluster 설정
  - docker.compose.yml 파일 생성
    ~~~
    version: '3.8'
    
    services:
      redis-node1:
        image: redis:latest
        container_name: redis-node1
        command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes
        ports:
          - "7001:6379"
        networks:
          - redis-cluster-network
      
      ..생략..
          
    networks:
      redis-cluster-network:
        driver: bridge
    ~~~
  - Redis Container 실행
    ~~~
    $ cd [docker-compose.yml 파일 위치]
    $ docker-compose up -d
    ~~~
  - Redis Cluster 생성 명령어
    ~~~
    # 1번 방식 (호스트에서 실행)
    $ docker exec -it redis-node1 redis-cli --cluster create \
      127.0.0.1:6379 127.0.0.1:6379 127.0.0.1:6379 127.0.0.1:6379 \
      --cluster-replicas 0


    # 2번 방식 (컨테이너 내부에서 실행)
    $ docker exec -it redis-node1 redis-cli --cluster create \
      redis-node1:6379 redis-node2:6379 redis-node3:6379 redis-node4:6379 \
      --cluster-replicas 0
    ~~~