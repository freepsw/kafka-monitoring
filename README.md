# Monitoring apache kafka using monitoring tools
- Apache Kafka를 구성하는 Broker, Producer, Consumer, Zookeeper를 통합 모니터링하기 위한 방안 검토
- Producer -> Kafka Clustr & Zookeeper Cluster -> Consumer


## [PART 1] Run Kafka Cluster & Zookeeper & Producer & Consumer with jmx option
 
## [PART 2] Monitoring Kafka using ELK stack
 - link (https://github.com/freepsw/kafka-monitoring/tree/master/01.Monitoring_with_ELK)

## [PART 3] Monitoring Kafka using datadog service
 - link (https://github.com/freepsw/kafka-monitoring/tree/master/10.Monitoring_with_Datadog)

## [PART 4] Implementing Kafka Monitoring System using docker_compose
 - link (https://github.com/freepsw/kafka-monitoring/blob/master/01.Monitoring_with_ELK/Kafka_Monitoring_using_docker.md)

# [ETC]

### 1. Clean system cache memory
```
sync
echo 1 > /proc/sys/vm/drop_caches
```


### 2. Test environment using logstash
#### run kafka_producer

```
> bin/logstash -e "input { stdin {} } output { kafka { bootstrap_servers => '<broker_ip>:9092' topic_id => 'test' } }"
```

### 3. logstash 실행시 고려사항
- path.data 옵션은 동일한 directory에서 logstash를 실행하면,
- logstash instance별로 사용하는 data directory를 별도로 지정해야 함

```
>  bin/logstash --path.data data1 -f config/kafka_consumer.yml
```

### 4. kafka-console-consumer

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```


### 4. Kafka ERROR filebeat logs에서 "dropping too large message of size" 해결
- https://discuss.elastic.co/t/filebeat-5-0beta1-with-kafka-output-multiplication-of-log-lines/62513/12 참고
### [Problem]
- compression: none 으로 압축을 하지 않고, 3,000 byte 메세지를 보낼때
- filebeat log에서 해당 오류가 발생함.
- 특이한 부분은 이 오류가 발생하면 retry를 시도하지 않고, 메세지가 유실됨.

### [Cause]
- 한번에 보내는 메세지 크기를 제한하는 Kafka Broker의 "message.max.bytes=1,000,000"을 초과하는 경우
#### 이상한 점 1.
  - 1개 메시지 크기는 3,000 byte인데,
  - 1,000,000 byte(1M)를 넘을 수 있을까?
  - 그렇다면 1개 메시지의 크기가 아니라 어떤 크기를 의미할까?

### [Solve]
- 아래 3개 설정 값을 함꼐 수정해야 정상적으로 메시지 처리가 가능하다.

- Broker (server.properties)

```
message.max.bytes=10000000 (10배 증가)
replica.fetch.max.bytes (이 부분은 변경하지 않았는데, 정상 동작)
```

- Producer

```
max.request.size=10000000
```

- Consumer

```
max.partition.fetch.bytes=10000000
```

#### 이상한 점 2.
  - producer의 설정은 변경하지 않고, 1,000,000으로 유지하고,
  - broker, consumer의 설정 값만 변경하였다.
  - 그런데 메시지가 정상적으로 처리되었다는 것은..
  - 결국 보낼때는 1,000,000 이하로 보내는데,
  - Broker에서 1,000,000 이상으로 인식한다는 의미가 되는데...
  - 음.... filebeat kafka output의 버그인가? (이상하다...)

#### [정확한 원인]
- 0.11.0.0 부터 max.message.bytes의 의미가 batch에 포함된 전체 메시지 사이즈로 변경됨!!!
  - https://kafka.apache.org/documentation/#upgrade_1100_notable
  - The broker configuration max.message.bytes now applies to the total size of a batch of messages


### 5. Kafka Error (Error: Executing consumer group command failed due to The consumer group command timed out while waiting for group to initialize)

#### [Problem]
##### kafka-consumer-groups.sh --describe 실행시 오류 발생

```
> /bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic __consumer_offsets
Error: Executing consumer group command failed due to The consumer group command timed out while waiting for group to initialize:
```

- ConsumerGroupCommand throws GroupCoordinatorNotAvailableException when describing a non-existent group before the offset topic is created
- http://mail-archives.apache.org/mod_mbox/kafka-dev/201606.mbox/%3CJIRA.12914176.1447867021000.57097.1465587741031@Atlassian.JIRA%3E

- The offsets.topic.replication.factor broker config is now enforced upon auto topic creation. Internal auto topic creation will fail with a GROUP_COORDINATOR_NOT_AVAILABLE error until the cluster size meets this replication factor requirement.
- https://kafka.apache.org/documentation/#upgrade_1100_notable



#### [Solve]

- 카프카 내부 토픽(consumer_offsets)을 여러 노드에 복제하도록 리밸런싱을 실행한다.
- http://www.chidoo.me/index.php/2017/05/30/kafka-monitoring-and-administration
- http://blog.leedohyun.pe.kr/2016/08/kafka-topic-replication-factor.html 참고

```
  > sudo pip install kafka-tools
  > /bin/kafka-assigner -z localhost:2181 --tools-path /home/poc/kafka/bin  -e set-replication-factor --topic __consumer_offsets --replication-factor 3
```

### 6. kafka & zookeeper restart

```
/home/poc/kafka/bin/kafka-server-stop.sh
/home/poc/zookeeper-3.4.10/bin/zkServer.sh stop

/home/poc/zookeeper-3.4.10/bin/zkServer.sh start
env JMX_PORT=9999 /home/poc/kafka/bin/kafka-server-start.sh config/server.properties &
```


#### 7. 추가 성능 테스트
##### 1) 1개 broker, topic 1개(AAA,) producer 10, lz4 처리 성능 : 100,000건 / 초
##### 2) 1개 broker, topic 3개(AAA, BBB, CCC), producer 10 * 3, lz4 성능 : 260,000건 / 초
 - 7/18 09:44
 - producer가 1대의 서버에서 구동되어 자원 부족현항으로 인하어 데이터 전송 속도가 늦음 (이로 인한 처리 건수가 낮아짐.)
 - 이후 producer를 3개의 다른서버에서 구동하면, 300,000건 /초 성능이 나올 것임.

```
>bin/kafka-console-producer.sh —broker-list broker01:9092,broker02:9092,broker03:9092 —topic DDD  —sync —compression-codec 'lz4' < 100m.dat
```
