
# 환경
* `t2.medium` 7대
    * ansible x 1
    * zookeeper x 3
    * kafka x 3
* `172.16.0.0 / 16`

# 카프카 구성 요소
## ZooKeeper
카프카의 메타데이터 관리 및 `브로커`의 헬스 체크를 담당
## Kafka 또는 Kafka cluster
여러 대의 `브로커`를 구성한 클러스터를 의미
## Broker
카프카 애플리케이션이 설치된 서버 또는 노드를 의미
## Producer
카프카로 `메시지`를 보내는 역할을 하는 클라이언트
* 배치 전송을 위해 레코드를 파티션별로 잠시 모아둔다.
* 파티션을 지정하지 않은 경우 
    * (기본값) 라운드로빈 방식으로 전송

* 전송 방법 세 가지
    - 메시지를 보내고 확인하지 않기
        ```java
        Producer<String, String> producer = new KafkaProducer<>(props);
        ProducerRecord<String, String> record = ProducerRecord<>("key", "value");
        producer.send(record);
        ```
    - 동기 전송
        ```java
        ProducerRecord<String, String> record = ProducerRecord<>("key", "value");
        RecordMetadata metadata = producer.send(record).get();
        ```
    - 비동기 전송
        ```java
        // org.apache.kafka.clients.producer.Callback 구현이 되어있어야 한다.
        // 처리되면 onCompletion(RecordMetadata metadata, Exception e)가 처리

        ProducerRecord<String, String> record = ProducerRecord<>("key", "value");
        producer.send(record, new ProducerCallback(record));
        ```
### Producer 주요 옵션
* `bootstrap.servers`
    - 클러스터 마스터 개념이 없어 클러스터 내 모든 서버가 클라이언트의 요청을 받을 수 있다.
    - 카프카 클러스터에 처음 연결하기 위한 홓스트와 포트 정보를 나타낸다.
* `client.dns.lookup`
    - 클라이언트가 서버의 IP와 연결하지 못할 경우에 다른 IP로 시도하는 설정
* `acks`
    - 0: 빠른 전송을 의미하지만 메시지 손실 가능성이 있다.
    - 1: 리더가 메시지를 받았는지 확인하지만 모든 팔로워를 전부 확인하지는 않는다.
    - all(-1): 팔로워가 메시지를 받았는지 여부를 확인하므로 느리지만 메시지 손실이 없다.
* `buffer.memory`
    - 데이터를 보내기 전 대기할 수 있는 전체 메모리 byte 값
* `compression.type`
    - none, gzip, snappy, lz4, zstd 중 원하는 타입 설정
* `enable.idempotence`
    - true: 중복 없는 전송이 가능
        - `max.in.flight.requests.per.connection` 값은 5 이하
        - `retries`는 0 이상
        - `acks`는 all로 설정해야 한다.
* `max.in.flight.requests.per.connection`
    - 하나의 커넥션에서 프로듀서가 최대한 ACK 없이 전송할 수 있는 요청 수
    - 메시지 순서가 중요하다면 1로 설정을 권장하지만, 성능은 다소 떨어진다.
* `retries`
    - 전송에 실패한 데이터를 다시 보내는 횟수
* `batch.size`
    - 배치 크기 설정 (적절하게)
* `linger.ms`
    - 배치 메시지를 보내기 전에 추가적인 메시지를 위해 기다리는 시간을 조정, 배치 크기에 도달하지 못한 상황에서 `linger.ms` 제한 시간에 도달했을 때 메시지를 전송
* `transactional.id`
    - '정확히 한 번 전송'
    - `enable.idempotence`를 true로 설정한다.
## Consumer
카프카에서 `메시지`를 꺼내가는 역할을 하는 클라이언트

* Consumer Group
    - Consumer가 속한 그룹
    - Consumer는 반드시 Consumer Group에 속하게 된다.
    - 각 Partition의 리더에게 카프카 토픽에 저장된 메시지를 가져오기 위한 요청을 보낸다.
    - Partition 수와 Consumer 수는 일대일로 매핑되는 것이 이상적
    - Partition 수 `<` Consumer 수는 올바르지 않다.
        - 더 존재하는 Consumer들이 그냥 대기 상태로만 존재하기 때문이다.

* 가져오는 방법 세 가지
    - 오토 커밋
        - 컨슈머 종료 등이 빈번히 일어나는 경우 일부 메시지를 못 가져오거나 중복으로 가져오는 경우가 있다.
        - 종료되는 현상이 거의 없으므로 오토 커밋을 사용하는 경우가 많음
        ```java
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("basic01")); // 구독할 토픽 지정
        
        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(1000);
                for (ConsumerRecords<String, String> record : records) {
                    System.out.printf("%s, %s, %d, %s, %s", 
                        record.topic(), record.partition(), record.offset(), record.key(), record.value()));
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            consumer.close();
        }
        ```
    - 동기 가져오기
        - 속도는 느리지만 메시지 손실이 거의 발생하지 않는다.
        - 하지만 이 방법도 메시지 중복 이슈는 피할 수 없다.
        ```java
        ...

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(1000);
            for (ConsumerRecords<String, String> record : records) {
                System.out.printf("%s, %s, %d, %s, %s", 
                    record.topic(), record.partition(), record.offset(), record.key(), record.value()));
            }
            consumer.commitSync(); // 현재 배치를 통해 읽은 모든 메시지를 처리한 후, 추가 메시지를 폴링하기 전 현재의 오프셋을 동기 커밋
        }
        
        ...
        ```
    - 비동기 가져오기
        - commitSync()와 달리 오프셋 커밋을 실패하더라도 재시도하지 않는다.
            - 이전 실패 작업이 재시도되는 경우 이전 오프셋부터 메시지를 가져오게 되고 중복이 발생
            - 이러한 이유로 비동기인 경우 커밋 재시도를 하지 않는다.
            - 비동기 커밋이 계속 실패하더라도 마지막의 비동기 커밋만 성공하면 안정적으로 오프셋을 커밋한다.
        - 
        ```java
        ...
        props.put("auto.offset.reset", "latest");
        props.put("enable.auto.commit", "false");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("basic01"));


        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(1000);
            for (ConsumerRecords<String, String> record : records) {
                System.out.printf("%s, %s, %d, %s, %s", 
                    record.topic(), record.partition(), record.offset(), record.key(), record.value()));
            }
            consumer.commitAsync(); // 현재 배치를 통해 읽은 모든 메시지를 처리한 후, 추가 메시지를 폴링하기 전 현재의 오프셋을 동기 커밋
        }
        
        ...
        ```
### Consumer 주요 옵션
* `bootstrap.servers`
* `fetch.min.bytes`
    - 한 번에 가져올 수 있는 최소 데이터 크기
    - 지정한 크기보다 작은 경우, 요청에 응답하지 않고 데이터가 누적될 때까지 기다린다.
* `group.id`
    - 컨슈머가 속한 컨슈머 그룹을 식별하는 식별자.
    - 동일한 그룹 내의 컨슈머 정보는 모두 공유된다.
* `heartbeat.interval.ms`
    - 컨슈머의 상태가 active임을 의미
    - `session.timeout.ms`와 밀접한 관계
        - `session.timeout.ms`보다 낮은 값으로 설정
        - 일반적으로 `session.timeout.ms`의 1/3로 설정
* `session.timeout.ms`
    - 컨슈머가 종료된 것인지를 판단.
    - 컨슈머는 주기적으로 하트비트를 보낸다.
    - 설정한 시간 전까지 하트비트를 보내지 않았다면 해당 컨슈머는 종료된 것으로 간주한다.
    - 종료된 것으로 판단하면 그룹에서 제외하고 리밸런싱을 시작
* `max.partition.fetch.bytes`
    - 파티션당 가져올 수 있는 최대 크기를 의미
* `enable.auto.commit`
    - 백그라운드로 주기적 오프셋을 커밋한다.
* `auto.offset.reset`
    - 카프카에서 초기 오프셋이 없거나 현재 오프셋이 더 이상 존재하지 않는 경우 다음 옵션으로 reset
        - earlist: 가장 초기의 오프셋값으로 설정
        - latest: 가장 마지막의 오프셋값으로 설정
        - none: 이전 오프셋값을 찾지 못하면 에러를 나타냄
* `fetch.max.bytes`
    - 한 번의 가져오기 요청으로 가져올 수 있는 최대 크기
* `group.instance.id`
    - 컨슈머의 고유한 식별자.
    - static 멤버로 간주되어, 불필요한 리밸런싱을 하지 않는다.
* `isolation.level`
    - 트랜잭션 컨슈머에서 사용되는 옵션
        - (기본값) read_uncommitted: 모든 메시지를 읽음
        - read_committed: 트랜잭션이 완료된 메시지만 읽음
* `max.poll.records`
    - 한 번의 poll() 요청으로 가져오는 최대 메시지 수
* `partition.assignment.strategy`
    - 파티션 할당 전략, 기본값은 range
* `fetch.max.wait.ms`
    - `fetch.min.bytes`에 의해 설정된 값보다 적은 경우 요청에 대한 응답을 기다리는 최대 시간
## Topic
`메시지` 피드들을 토픽으로 구분하고 `토픽`의 이름은 카프카 내에서 고유하다
### Replication
각 메시지들을 여러 개로 복제해서 카프카 클러스터 내 브로커들에게 분산시킨다.

> 정확히는 토픽의 파티션이 리플리케이션된다.

**권장 리플리케이션 팩터 값**
* 테스트 또는 개발 환경: 1
* 운영 환경 (로그성 메시지로서 약간의 유실 허용): 2
* 운영 환경 (유실 허용 X): 3
> 4 또는 5 이상인 경우 디스크 공간을 더 많이 사용한다.

### Leader, Follower
* 리더는 리플리케이션 중 하나가 선정
* 모든 읽기와 쓰기는 리더를 통해서만 가능하다.
* 또한 컨슈머와 프로듀서도 오직 리더로 부터 메시지를 가져오고 리더에게 메시지를 보낸다.

### ISR (InSyncReplica)
ISR 내 팔로워들은 리더와의 데이터 일치를 유지하기 위해 지속적으로 리더의 데이터를 따라가고 리더는 ISR 내 모든 팔로워가 메시지를 받을 때까지 기다린다.

팔로워가 특정 주기의 시간만큼 복제 요청을 하지 않는다면, 리더는 해당 팔로워가 리플리케이션 기능에 문제가 있다고 판단해 ISR 그룹에서 추방시킨다.

모든 팔로워의 복제가 완료되면, 내부적으로 커밋되었다는 표시를 하는데 마지막 커밋 오프셋 위치는 `high water mark`라고 부른다.
* 커밋되었다는 의미는 모든 리플리케이션이 전부 메시지를 저장했다라는 뜻이다.
* 커밋된 메시지만 컨슈머가 읽어갈 수 있다.
* `/data/kafka-logs/replication-offset-checkpoint` 파일을 읽어보면 해당 토픽의 커밋 번호를 알 수 있다.
* 리더와 팔로워 사이에 ACK 통신을 하지않고 팔로워가 pull 하는 방식을 채택
## Partition
병렬 처리 및 고성능을 얻기 위해 하나의 `토픽`을 여러 개로 나눈 것
* 파티션 번호는 0번 부터 시작한다.
* 초기 생성 후 언제든지 늘릴 수 있지만, 반대로 줄일 수 없기 때문에 LAG 등을 모니터링하면서 LAG을 통해 컨슈머에 지연이 없는지 확인하고 조금씩 늘려가는 방법을 추천한다. 
> LAG = (프로듀서가 보낸 메시지 수) - (컨슈머가 가져간 메시지 수)
- 파티션의 메시지가 저장되는 위치를 `오프셋(offset)`이라고 부른다.
- 각 파티션에서의 오프셋은 고유한 숫자로, 오프셋을 통해 메시지의 순서를 보장하고 컨슈머에서는 마지막까지 읽은 위치를 알 수 있다.

### Partition 복구 : 리더에포크 (LeaderEpoch)
1. 장애 처리가 완료되고 카프카 프로세스가 시작되면서 복구 동작을 통해 자신의 [high water mark](###ISR)앞에 있는 메시지를 무조건 삭제하는 것이 아니라 리더에게 리더에포크 요청을 보낸다.
1. 리더는 리더에포크의 응답으로 오프셋의 어디까지인지 팔로워에게 보낸다.
1. 팔로워는 자신의 high water mark보다 높은 오프셋의 메시지를 삭제하지 않고, 리더의 응답을 확인한 후 메시지까지 자신의 high water mark를 상향 조정한다.

즉, 리더가 예상치 못한 장애로 다운되면서 팔로워가 새로운 리더로 승격되어도 하리더에포크 요청과 응답 과정을 통해 팔로워의 하이워터마크를 올릴 수 있어, 메시지 손실이 발생하지 않는다.


## Segment
`프로듀서`가 전송한 실제 `메시지`가 `브로커`의 로컬 디스크에 저장되는 파일
### 저장 과정
파티션이 1개인 test 토픽이 있다고 가정한다.
* 카프카 서버의 `/data/kafka-logs/{토픽 명}-{파티션 번호}`에서 로그 파일을 확인할 수 있다.
1. 프로듀서는 카프카의 test 토픽으로 메시지를 전송
1. test 토픽은 파티션0의 세그먼트 로그 파일에 저장
1. 로그 파일에 저장된 메시지는 컨슈머가 읽어갈 수 있다.
## Message 또는 Record
`프로듀서`가 `브로커`로 전송하거나 `컨슈머`가 읽어가는 데이터 조각

# 높은 처리량과 안전성을 지니게 된 카프카의 특성
## 분산 시스템
* 하나의 서버 또는 노드 등에 장애가 발생할 때 다른 서버 또는 노드가 대신 처리한다.
* 부하가 높은 경우에는 시스템 확장이 용이하다.
* 브로커는 온라인 상태에서 매우 간단하게 추가가 가능

## 페이지 캐시
* 직접 디스크에 읽고 쓰는 방식 대신 물리 메모리 중 애플리케이션이 사용하지 않는 일부 잔여 메모리를 활용
* 즉, 디스크 I/O에 대한 접근이 줄어들어 성능을 높일 수 있다.
## 배치 전송 처리
* 수많은 통신을 묶어서 처리 (배치 전송)
* 단건으로 통신할 때에 비해 네트워크 오버헤드를 줄일 수 있다.
## 압축 전송
지원 압축 타입
* 압축률이 높은 타입
    - gzip
    - zstd
- 빠른 응답 속도
    - lz4
    - snappy
## 고가용성 보장
고가용성을 보장하기 위해 `리플리케이션` 기능을 제공한다.

|리플리케이션 팩터 수|리더 수|팔로워 수|
|---|---|---|
|2|1|1|
|3|1|2|
|4|1|3|

팔로워의 수만큼 브로커의 디스크 공간도 소비되므로 이상적인 [리플리케이션 팩터 수](#replication)를 유지해야한다.
