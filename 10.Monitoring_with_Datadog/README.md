## PART 1. Monitoring using datadog
- datadog는 cloud기반의 상용 모니터링 서비스
- 즉, kafka broker가 구동하고 있는 서버에서, datadog 서버로 접속이 가능해야 한다.
- https://www.datadoghq.com/blog/monitor-kafka-with-datadog/ 참고

### STEP 1. Start with free-trial (30 days)
- connect to https://www.datadoghq.com/
- get API Key
- copy printed command line on the web site
- install data-dog agent

```
# centos
> DD_API_KEY=xxxxxxxxx bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/dd-agent/master/packaging/datadog-agent/source/install_agent.sh)"
```

### STEP 2. Setting apache kafka integrations for Monitoring on Datadog web site
- Menu > integrations 선택
- Apache kafka click > install
- Apache zookeeper click > install
- monitoring을 위한 환경구성 까지 약 5분 정도 소요됨.


### STEP 3. Setting datadog configuration for apache kafka
- 수집할 metrics 정보를 설정 (default 사용)
```
> cd /etc/dd-agent/conf.d
> sudo cp kafka.yaml.example kafka.yaml

instances:
  - host: localhost
    port: 9999 # This is the JMX port on which Kafka exposes its metrics (usually 9999)
    tags:
      kafka: broker

> sudo cp kafka_consumer.yaml.example kafka_consumer.yaml

instances:
   - kafka_connect_str: localhost:9092
     zk_connect_str: localhost:2181
     zk_prefix: /0.8
     consumer_groups:
       my_consumer:  --> "consumer_group의 이름을 명시, console_consumer는 조회되지 않음"
         my_topic: [test]


# datadog agent restart
# http://docs.datadoghq.com/guides/basic_agent_usage/centos/ 참고
> sudo /etc/init.d/datadog-agent restart

# check datadog agent status
> sudo /etc/init.d/datadog-agent info
```


### STEP 4. Monitoring on datadog Dashboard
- https://app.datadoghq.com/dash/list 접속
- Custom Kafka 를 클릭히면, 상세 metrics 정보가 제공됨
