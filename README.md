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
  - 참고 사항 :
    - 아래 명령어에서 redis-node1 컨테이너 내부에서 실행하는 것이지만, 이는 단순히 명령어 실행을 위한 것이며 redis-node1이 특별한 역할을 하지는 않음
    - 즉, docker exec -it redis-node1 대신 docker exec -it redis-node2 또는 docker exec -it redis-node3에서 실행해도 클러스터 동작 자체에는 차이가 없음
    - 단, redis-cli --cluster create 명령어는 클러스터 초기 설정을 수행하는 것이므로, 한 번만 실행하면 됨. (어느 노드에서 실행하든 관계없음)
  - Redis Container 실행
    ~~~
    $ cd [docker-compose.yml 파일 위치]
    $ docker-compose up -d
    ~~~
  - Redis Cluster 생성 명령어 ( Master Node만 생성 )
    ~~~
    # 컨테이너 내부에서 실행
    $ docker exec -it redis-node1 redis-cli --cluster create \
      redis-node1:6379 redis-node2:6379 redis-node3:6379 \
      --cluster-replicas 0
    ~~~
  - Redis Cluster 생성 명령어 ( Master/Replica Node 같이 생성 )
    ~~~
    # 각각의 Master에 Replica Node 1개씩 할당 (Redis가 자동으로 6개의 Node 중에서 3개를 마스터로 선택하고, 나머지를 슬레이브로 자동 배정함)
    # 슬레이브는 반드시 특정 마스터의 레플리카로 설정되지 않을 수도 있음 (자동 배정 방식이므로 결과가 달라질 수 있음)
    $ docker exec -it redis-node1 redis-cli --cluster create \
      redis-node1:6379 redis-node2:6379 redis-node3:6379 redis-node4:6379 redis-node5:6379 redis-node6:6379 \
      --cluster-replicas 1
    ~~~
  - Redis Cluster 생성 명령어 ( Master Node로 전부 생성 후, Replica Node로 일부 변경하여 연결 )  
    ~~~
    # 초기에는 모든 노드를 Master로 설정하고, 이후에 특정 노드를 Replica로 변경함
    # 이 방식은 Replica를 자동 할당하지 않고 사용자가 직접 Replica를 설정할 수 있도록 하기 위함
    $ docker exec -it redis-node1 redis-cli --cluster create \
      redis-node1:6379 redis-node2:6379 redis-node3:6379 redis-node4:6379 redis-node5:6379 redis-node6:6379 \
      --cluster-replicas 0
    
    # 주의: 이 상태에서는 모든 노드가 Master이므로, 장애 발생 시 자동 복구 기능이 없음
    # 따라서 아래 명령어를 통해 Replica를 수동으로 설정해야 함
    
    # redis-node1의 replica node를 redis-node4로 설정
    # redis-node2의 replica node를 redis-node5로 설정
    # redis-node3의 replica node를 redis-node6로 설정
    $ docker exec -it redis-node1 redis-cli --cluster add-node redis-node4:6379 redis-node1:6379 --cluster-slave
    $ docker exec -it redis-node1 redis-cli --cluster add-node redis-node5:6379 redis-node2:6379 --cluster-slave
    $ docker exec -it redis-node1 redis-cli --cluster add-node redis-node6:6379 redis-node3:6379 --cluster-slave
    ~~~
  - Redis Cluster Node 목록 확인
    ~~~
    $ docker exec -it redis-node1 redis-cli cluster nodes
    
    # (명령어 실행 결과) Master 노드만 3개 존재하는 경우
    ID값1 172.20.0.4:6379@16379 myself,master - 0 0 1 connected 0-5460
    ID값2 172.20.0.3:6379@16379 master - 0 1740961873072 3 connected 10923-16383
    ID값3 172.20.0.5:6379@16379 master - 0 1740961874078 2 connected 5461-10922
    ~~~