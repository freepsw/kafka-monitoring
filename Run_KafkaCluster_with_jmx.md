# Run Kafka Cluster & Zookeeper & Producer & Consumer with jmx option
- Apache Kafka와 관련된 컴퍼넌트들의 jmx metrics를 수집하기 위한 설정

## [PART 0] Run Apache Kafka & Zookeeper Cluster with jmx option

### STEP 1. Run Apache Zookeeper with JMX_PORT environment

#### Setting Zookeeper configuration
- conf/zoo.cfg

```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/poc/zookeeper-3.4.10/data
clientPort=2181
server.1=192.178.50.1:2888:3888
server.2=192.178.50.2:2888:3888
server.3=192.178.50.3:2888:3888
```

- data/myid

- conf/java.env 수정
```
export JVMFLAGS="-Xmx2048m"
```

- bin/zkServer.sh 수정
```
ZOOMAIN="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9998 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false org.apache.zookeeper.server.quorum.QuorumPeerMain"
```

#### Run Zookeeper cluster (Cluster)
```
> bin/zkServer.sh start
```

#### Run Zookeeper cluster (Standalone)

```
> env JMX_PORT=9998 bin/zookeeper-server-start.sh config/zookeeper.properties
```


#### check zookeeper status
```
> echo ruok | nc localhost 2181
imok (정상)
```

### STEP 2. Run Apache Kafka with JMX_PORT environment

#### Setting Kafka Configuration
- conf/server.properties
```
broker.id=1
log.dirs=/logdata/data/kafka  
zookeeper.connect=192.168.100.135:2181,192.xxx.xxx.136:2181,192.xxx.xxx.137:2181
```

#### Config JMS configuration
- bin/kafka-run-class.sh
- 외부 jmx client(jconsole 등)에서 접속 할 수 있도록, IP/hostname 정보를 등록한다.
```
# JMX settings
if [ -z "$KAFKA_JMX_OPTS" ]; then
  KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=34.85.37.182(kafka broker ip or hostname)  -Djava.net.preferIPv4Stack=true"

fi
```


#### Run Kafka Cluster

```
> env JMX_PORT=9999 bin/kafka-server-start.sh config/server.properties
```

#### create topic (test)
```
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```


## [PART 2] Run Kafka Producer
- file에서 메시지를 읽어와서, kafka topic으로 전송한다.
- 모니터링이 가능하도록 jmx port 설정을 추가하여 실행한다.

### Case 1. Run producer using logstash
#### logstash jmx configuration
```
> vi bin/logstash.lib.sh
-- 아래 JAVA_OPTS 항목을 추가
if [ "$LS_JAVA_OPTS" ] ; then
  # The client set the variable LS_JAVA_OPTS, choosing his own
  # set of java opts.
  JAVA_OPTS="$JAVA_OPTS $LS_JAVA_OPTS"
  JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote"
  JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.port=9881"
  JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.local.only=false"
  JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.authenticate=false"
  JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.ssl=false"
fi

또는 (logstash 7.4에서는 아래 옵션 필요 )
LS_JAVA_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=9881 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=34.84.47.120 -Dcom.sun.management.jmxremote.local.only=false"
```

- 또는 export를 이용
```
> export LS_JAVA_OPTS=" -Dcom.sun.management.jmxremote.authenticate=false \
                        -Dcom.sun.management.jmxremote.port=9010 \
                        -Dcom.sun.management.jmxremote.ssl=false \
                        -Dcom.sun.management.jmxremote.authenticate=false"
```  

- 또는 jmxremote.port를 제외한 값은 logstash.lib.sh에 추가하고,
- jmxremote.port 만 export로 실행
- logstash를 하나의 서버에서
```
> export LS_JAVA_OPTS=" -Dcom.sun.management.jmxremote.port=9881 "
```  

#### logstash config
- file을 읽어와서 kafka topic으로 메시지를 전송한다.
- config/logstash_producer.yml

```yml

input {
  file {
    path => ["$1/F1.dat"]
    start_position => "beginning"
  }
}

output {
   kafka {
     codec => plain { format => "%{message}" }
     bootstrap_servers => "broker06:9092"
     topic_id => "F1"
     acks => "all"
     retries => 20
     batch_size => 49152
     linger_ms => 2000
   }
}
```

#### Run logstash
```
> logstash -f config/logstash_producer.conf
```

### Case 2. Run producer using filebeat
- logstash의 경우 여러개 process를 구동하면 서버의 자원을 너무 많이 소모한다.
- 따라서 성능 테스트와 같이 여러개의 producer를 구동해야 할 경우에는
  filebeat과 같은 경량화된 도구를 사용하는 것이 효과적이다

#### filebeat producer config file

- filebeat_producer.yml
```yml
filebeat.prospectors:
- input_type: log
  document_type: F1
  paths:
    - /home/poc/data/F1.dat

- input_type: log
  document_type: test2
  paths:
    - /home/poc/data/F2.dat

output.kafka:
  # The Logstash hosts
  hosts: ["broker06:9092","broker07:9092","broker08:9092"]
  topic: '%{[type]}'
  codec.format:
    string: '%{[message]}'
  required_acks: -1
  max_retries: 3
  bulk_max_size: 2048
```

#### run flebeat
```
> filebeat  -c conf/filebeat_producer.yml -d "publish"  &
```

### Case 3. Run producer using kafka command line
- filebeat와 logstash의 경우 lz4 압축 옵션을 지원하지 못한다.
- 따라서 해당 압축 옵션을 이용하여 성능을 측정하는 경우 직접 producer를 구현하거나, kafka에서 제공하는 tool을 활용한다.
```
> kafka-console-producer.sh —broker-list broker06:9092,broker07:9092,broker08:9092 —topic F1  —sync —compression-codec 'lz4' < F1.dat
```


## [PART 3] Run Kafka Consumer
- Kafka Producer와 마찬가지로 jmx 설정

#### setting logstash config file

- logstash_consumer.conf
```yml
input {
  kafka {
    type => "F1"
    bootstrap_servers => "broker06:9092,broker07:9092,broker08:9092"
    topics => "F1"
    group_id => "logstash"
    max_poll_records => "5"
    receive_buffer_bytes => "10240"
  }
}

output {
    file {
       path => "F1.dat"
       codec => line { format => "%{message}" }
     }
}

```

#### Run logstash
```
> logstash -f config/logstash_consumer.conf
```
