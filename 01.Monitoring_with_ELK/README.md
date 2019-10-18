# Monitoring Kafka JMX metrics using ELK stack
- Apache Kafka를 구성하는 Broker, Producer, Consumer, Zookeeper를 통합 모니터링하기 위한 방안 검토
- JMX metrics 기반으로, ELK stack을 이용하여 통합 모니터링 dashboard를 구성한다.

# Kafka 모니터링 아키텍처
- 우리가 구축할 Kafka 모니터링 아키텍처
-
 ![architecture evtns](https://github.com/freepsw/kafka-monitoring/blob/master/01.Monitoring_with_ELK/img_monitoing_stack.png?raw=true)

# [PART 1] Install and run Elasticsearch & Kibana
## STEP 1.  Elasticsearch-5.4.3 Install
```
> mkdir ~/apps
> cd ~/apps
> wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.6.1.tar.gz
> tar xvf elasticsearch-6.6.1.tar.gz
> cd elasticsearch-6.6.1
> vi config/elasticsearch.yml  (network.host: 0.0.0.0)
> bin/elasticsearch
```

### [Error 해결] max file descriptors [4096] for elasticsearch process is too low 오류 발생시
- file descriptors 수를 증가
- https://www.elastic.co/guide/en/elasticsearch/reference/current/setting-system-settings.html#limits.conf 참고
```
sudo vi /etc/security/limits.conf

username       -       nofile          65536
username       -       nproc           262144
```

### [Error 해결] max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

```
> sudo vi /etc/sysctl.conf
> sudo sysctl -a  # 설정값 확인
> sudo sysctl -w vm.max_map_count=262144
```


## STEP 2. Kibana Install

```
> mkdir ~/apps
> cd ~/apps
> wget https://artifacts.elastic.co/downloads/kibana/kibana-6.6.1-linux-x86_64.tar.gz
> tar xvf kibana-6.6.1-linux-x86_64.tar.gz
> cd kibana-6.6.1-linux-x86_64
> vi config/kibana.yml (server.host: "0.0.0.0")
> cd kibana-5.4.3-linux-x86_64
> bin/kibana

# backgroud 실행
> nohup bin/kibana &
> ps -ef | grep node
```


# [PART 2]. Collect Kafka JMX Metrics using logstash

## STEP 1. Collect Kafka Broker/Producer/Consumer jmx metrics using logstash (jmx_port 9999)
- https://www.datadoghq.com/blog/monitoring-kafka-performance-metrics/ 참고
- http://docs.confluent.io/current/kafka/monitoring.html#new-consumer-metrics 참고

```
> cd ~/apps
> wget https://artifacts.elastic.co/downloads/logstash/logstash-6.6.1.tar.gz
> tar xvf logstash-6.6.1.tar.gz
> cd logstash-6.6.1
> bin/logstash-plugin install logstash-input-jmx
```

### Logstash 수집을 위한 config 설정
- jmx input plugin을 이용하여 jmx metrics를 수집한다.
- jmx path 아래에 있는 jmx 설정파일 들을 수집한다. (broker, producer, consumer, zookeeper)
- logstash_jmx_kafka.yml

```yml
input {
 jmx {
  path => "/home/poc/jmx_conf"
  polling_frequency => 2
  type => "jmx"
 }
}

filter {
 if [type] == "jmx" {
   if ("ProcessCpuLoad" in [metric_path] or "SystemCpuLoad" in [metric_path]) {
     ruby {
       code => "n = event.get('[metric_value_number]')
                event.set('[metric_value_number]', n*100)
                event.set('cpu_load',n)
               "
     }
   }
   if ("TotalPhysicalMemorySize" in [metric_path] or "FreePhysicalMemorySize" in [metric_path]) {
     ruby {
       code => "n = event.get('[metric_value_number]')
                event.set('[metric_value_number]', n/1024/1024)
                event.set('physical_memory',n)
               "
     }
   }
   if ("HeapMemoryUsage" in [metric_path] or "HeapMemoryUsage" in [metric_path]) {
     ruby {
       code => "n = event.get('[metric_value_number]')
                event.set('[metric_value_number]', n/1024/1024)
               "
     }
   }
 }
}

output{
 stdout {
  codec => rubydebug
 }
 elasticsearch {
   hosts => "localhost:9200"
   index => "kafka_mon"
 }
}
```
- 각 event의 메시지 별로 별도의 로직을 추가하거나, 필드를 추가할 수 있다.
- 위의 예시에서 event api에서 제공하는 event.set을 통하여 값을 변경하거나, 필드를 추가함.
- logstash를 실행해 보면, 화면에 cpu_load라는 필드가 추가된 것을 볼 수 있다.
- https://www.elastic.co/guide/en/logstash/current/event-api.html 참고

### Xpack monitoring 을 위한 설정
- Kibana monitoring에서 logstash의 성능을 모니터링 하기 위한 설정 
- config/logstash.yml 에 xpack 모니터링을 활성화 하기 위한 설정 추가
```yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.username: logstash_system
xpack.monitoring.elasticsearch.password: password
xpack.monitoring.elasticsearch.url: ["http://localhost:9200"]
```

```yml
{
    "metric_value_number" => 0.020206998521439132,
                   "path" => "/home/rts/apps/logstash-5.4.1/jmx",
             "@timestamp" => 2017-06-22T06:26:49.297Z,
               "@version" => "1",
                   "host" => "localhost",
            "metric_path" => "kafkabroker1.OperatingSystem.SystemCpuLoad",
                   "type" => "jmx",
               "cpu_load" => 0.020206998521439132
}
```

### JMX Configuration for Broker (jmx_conf/jmx_kafka_broker.conf)
- https://developers.lightbend.com/docs/opsclarity/latest/articles/Kafka-Brokers.html 참고

```shell
{
  "host" : "broker06",
  "port" : 9999,
  "alias" : "kafkabroker1",
  "queries" : [
  {
    "object_name" : "kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec",
    "attributes" : [ "OneMinuteRate" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec",
    "attributes" : [ "OneMinuteRate" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec",
    "attributes" : [ "OneMinuteRate" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions",
    "attributes" : [ "Value" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce",
    "attributes" : [ "Mean" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer",
    "attributes" : [ "Mean" ],
    "object_alias" : "${type}.${name}"
  }
 ]
}
```

#### - Broker Kafka metric 이해 (평균 값)
- MeanRate, OneMinuteRate 등의 Attributes가 있는데, 어떤 차이가 있는지 이해가 필요함. (모니터링 시)
- 예를 들어 kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec, BytesOutPerSec가 있는 경우
- BytesOutPerSec 별로 다양한 Attributes가 존재한다.
  - MeanRate : producer가 구동한 이후부터 broker에 전송된 bytes/sec (이 경우, 왜곡된 값이 보일 수 있다.)
    * 예를 들어, producer가 매일 1시간 만 전송하는 경우,
    * 1시간 동안 전송된 bytes의 초당 평균 전송량을 봐야 하는데,
    * MeanRate를 사용하면, producer가 구동된 이후에 전송된 모든 bytes를
    * producer가 구동된 시간으로 평균을 구하게 됨. (따라서 낮은 값이 보임)
  - OneMinuteRate : 이전 60초 동안 전송된 평균 bytes
- https://stackoverflow.com/questions/43677710/kafka-jmx-metric-current-rate-of-messages-in/43679512 참고



### JMX Configuration for Broker Resource (jmx_conf/jmx_kafka_broekr_resource.conf)
- http://help.boomi.com/atomsphere/GUID-48F37A93-5BF9-488D-82C3-38E4E9D45A22.html 참고
- CPU 부하
  * "SystemCpuLoad", "ProcessCpuLoad"
  * SystemLoadAverage : 지난 1분간 서버의 CPU 사용량 (0: 사용하지 않음. 1: 1개의 CPU가 100% 사용, 2: 2개의 CPU가 100% 사용)
- Open File 현황
  * "OpenFileDescriptorCount" : Kafka Broker를 구동한 User의 open file 갯수
- Memory 사용량
  * TotalPhysicalMemorySize : 전체 물리 메모리 크기
  * FreePhysicalMemorySize : 사용가능한 물리 메모리 크기
- JVM Heap Memory 사용량

```shell
{
  "host" : "broker06",
  "port" : 9999,
  "alias" : "kafkabroker1",
  "queries" : [
  {
    "object_name" : "java.lang:type=OperatingSystem",
    "attributes" : ["SystemCpuLoad", "ProcessCpuLoad", "OpenFileDescriptorCount","FreePhysicalMemorySize", "TotalPhysicalMemorySize" ],
    "object_alias" : "${type}"
  },
  {
    "object_name" : "java.lang:type=Memory",
    "attributes" : ["HeapMemoryUsage", "NonHeapMemoryUsage"],
    "object_alias" : "${type}"
  }    
 ]
}
```

### JMX Configuration for Producer (jmx_conf/jmx_kafka_producer.conf)

```json
{
  "host" : "localhost",
  "port" : 9881,
  "alias" : "kafka-producer",
  "queries" : [
  {
    "object_name" : "kafka.producer:type=producer-metrics,client-id=*",
    "attributes" : ["outgoing-byte-rate"],
    "object_alias" : "Producer.BytesRate"
  },
  {
    "object_name" : "kafka.producer:type=producer-metrics,client-id=*",
    "attributes" : ["request-rate"],
    "object_alias" : "RequestRate"
  },
  {
    "object_name" : "kafka.producer:type=producer-metrics,client-id=*",
    "attributes" : ["response-rate"],
  },
  {
    "object_name" : "kafka.producer:type=producer-metrics,client-id=*",
    "attributes" : ["request-latency-avg"],
  },
  {
    "object_name" : "java.lang:type=OperatingSystem",
    "object_alias" : "OperatingSystem",
    "attributes" : ["SystemCpuLoad"]
   }    
 ]
}
```

### JMX Configuration for Consumer (jmx_conf/jmx_kafka_consumer.conf)
```json
{
  "host" : "localhost",
  "port" : 9882,
  "alias" : "kafka-consumer",
  "queries" : [
  {
    "object_name" : "kafka.consumer:type=consumer-metrics,client-id=*",
    "attributes" : ["incoming-byte-rate"],
    "object_alias" : "Consumer.InBytesRate"
  },
  {
    "object_name" : "kafka.consumer:type=consumer-fetch-manager-metrics,client-id=logstash-0",
    "attributes" : ["bytes-consumed-rate"],
    "object_alias" : "Consumer.ByteConsumed"
  },
  {
    "object_name" : "kafka.consumer:type=consumer-fetch-manager-metrics,client-id=logstash-0",
    "attributes" : ["records-consumed-rate"],
    "object_alias" : "Consumer.RecordsConsumed"
  },
  {
    "object_name" : "java.lang:type=OperatingSystem",
    "object_alias" : "OperatingSystem",
    "attributes" : ["SystemCpuLoad"]
   }    
 ]
}
```

### JMX Configuration for Consumer (jmx_conf/jmx_kafka_consumer_lag.conf)
- records-lag-max의 경우, 값이 없을 때 "-Infinity"라는 값이 저장된다
- 그런데, 이 값은 string이므로 jmx 값을 파싱할 때 오류가 발생하면서,
- 나머지 metrics도 수집못하게 되는 문제가 발생한다. (값이 있을 때는 정상동작)
- 그래서 일단 jmx config를 분리하여 처리함. (좀 더 근본적으로는 logstash event api로 처리 가능한지 확인 필요)
```json
{
  "host" : "localhost",
  "port" : 9882,
  "alias" : "kafka-consumer",
  "queries" : [
  {
    "object_name" : "kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*",
    "attributes" : ["records-lag-max"],
    "object_alias" : "Consumer.RecordsLagMax"
  }
 ]
}
```


### [참고] kafka 관련 jmx metrics 참고
 - https://github.com/wikimedia/puppet-kafka/blob/master/manifests/server/jmxtrans.pp


## STEP 2. Run logstash
- lgstash를 실행하면,
- 3개의 jmx설정에 따라서 각각의 jmx metrics를 수집하게 된다.

```
bin/logstash  -f config/jmx_kafka.yml
```
- 만약 jmx plugin이 없다는 오류가 발생하면, 직접 설치한다.

```
# online install
> bin/logstash-plugin install logstash-input-jmx

# offline install
# export plugins to zip
>  bin/logstash-plugin prepare-offline-pack logstash-input-jmx

# import zip to logstash plugins
> bin/logstash-plugin install file:///home/rts/logstash-offline-plugins-5.4.1.zip
```


### 수집된 jmx metrics 예시

```yml
{
    "metric_value_number" => 0.1642061100531988,
                   "path" => "/home/rts/apps/logstash-5.4.1/jmx",
             "@timestamp" => 2017-06-20T09:08:19.968Z,
               "@version" => "1",
                   "host" => "localhost",
            "metric_path" => "kafkabroker1.kafka.server:type=BrokerTopicMetrics,name=TotalFetchRequestsPerSec.FiveMinuteRate",
                   "type" => "jmx"
}
```


# [PART 3]. Monitorig JMX metrics using kibana

### STEP 1. Kibana Index 생성
- Menu : Management > Index Pattenrs > "+" 버튼 클릭 > "index name" 필드에 "kafka-mon" 입력

### STEP 2. Import  visualizations & dashboard object
- Menu : Management > Saved Object > "import" 버튼 클릭 > 아래 2개 파일 import
  - kibana/kibana_dashboard.json
  - kibana/kibana_visualizations.json

### STEP 3. Monitoring Kafka metrics using kibana dashboard
