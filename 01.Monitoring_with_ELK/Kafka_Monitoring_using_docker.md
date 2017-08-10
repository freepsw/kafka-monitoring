# Implementing Kafka Monitoring System using docker_compose
- docker_compose를 이용하여 kafka broker, zookeeper, producer, consumer에 대한 정보를 수집하는 시스템 구성
- 사용자가 별도의 오픈소스 설치없이, logstash config 파일만 수정하여 모니터링 가능한 구조
- kibana의 시각화는 미리 정의된 visualization & dashboard file을 import하여 한번에 구성 가능



# [PART 0]. Prerequisite
- Kafka Monitoring 시스템 구축을 위해 사전에 필요한 소프트웨어 및 설정
- Software environment
```
Apache Kafka : 0.11.0
elasticsearch : 5.5.1
kibana : 5.5.1
logstash : 5.5.1
```

## STEP 1. Software 설치 및 실행 (docker-compose 실행에 필요)
- docker-ce 설치 및 실행
- docker-compose 설치 및 실행

- https://github.com/freepsw/simple-log-monitoring-using-docker/tree/master/04.elastic#step-01-install-necessary-sw 참고

## STEP 2. Check Kafka Environment (IP, JMX Port)
- 모니터링할 서버의 IP 목록과 JMX Port를 사전에 확인
  - Kafka Broker
  - Zookeeper
  - Kafka Producer
  - Kafka Consumer

- 만약 JMX port가 설정되지 않았다면, 모니터링할 jmx metrics를 수집할 수 없다.


# [PART 1]. Configure docker-dompose for Kafka Monitoring
- 아래 링크에서 elk stack을 docker-compse로 구성하기 위한 가이드를 자세히 제공한다.
- https://github.com/deviantony/docker-elk 참고

## STEP 1. Download ELK project
- logstash -> elasticsearch --> kibana 실행을 위한 기본 docker-compose를 다운로드
- github에 공개된 docker-compose project를 활용한다.
- https://github.com/deviantony/docker-elk

```
> cd ~
> git clone https://github.com/deviantony/docker-elk.git
> cd dokeer-elk
```

## STEP 2. Configure logstash pipeline setting
- elasticsearch에서 제공하는 기본 Dockerfile에 logstash 실행과 관련된 내용이 작성됨.
- 또한 docker-compose에서 logstash pipiline을 정의한 config파일의 경로를
- docker run 시에 volume을 이용하여 참고하도록 설정하였다
- docker-compose.yml
```
logstash:
  build: logstash/
  volumes:
    - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    - ./logstash/pipeline:/usr/share/logstash/pipeline
  ports:
    - "5000:5000"
  environment:
    LS_JAVA_OPTS: "-Xmx256m -Xms256m"
  networks:
    - elk
  depends_on:
    - elasticsearch
```
### pipeline 디렉토리 아래에 jmx 수집을 위한 설정 파일 복사
```
> cp config/logstash_jmx_kafka.yml docker-elk/logstash/pipeline
> cp -R config/jmx_conf docker-elk/logstash/pipeline
```
### logstash에서 jmx를 사용할 수 있도록 plugin을 설치
- 아래의 plugin 설치 스크립트 추가
- vi docker-elk/logstash/Dockerfile
```
RUN logstash-plugin install logstash-input-jmx
```
- docker image 가 새롭게 build 되어야 하므로, 기존 image가 존재한다면 docker rmi로 삭제한 후 실행


## STEP 3. Run ELK Stack using docker-compose
- 아래의 명령어 만으로 logstash -> elasticsearch --> kibana 프로세스 구동이 완료된다.
```
> cd ~
> git clone https://github.com/deviantony/docker-elk.git
> cd dokeer-elk
> docker-compose up
```

- 정상적으로 jmx metrics를 수집한다면, 아래와 같은 로그가 출력되어야 함.
```
logstash_1       | {
logstash_1       |     "metric_value_number" => -1,
logstash_1       |                    "path" => "/usr/share/logstash/pipeline/jmx",
logstash_1       |              "@timestamp" => 2017-08-10T01:26:36.021Z,
logstash_1       |                "@version" => "1",
logstash_1       |                    "host" => "10.178.50.105",
logstash_1       |             "metric_path" => "kafkabroker2.Memory.NonHeapMemoryUsage.max",
logstash_1       |                    "type" => "jmx"
logstash_1       | }
```


# [PART 3]. Monitoring Kafka jmx metrics
## STEP 1. Open Kibana Web UI
- http://<ip>:5601 로 kibana로 접속

## STEP 2. Create Kibana index
- Management > Index Patterns > Create Index Pattern 클릭
- "Index name or pattern"의 텍스트 박스에 "kafka_mon"을 입력하고,
- "Time Filter field name"의 select box에서 "@timestamp"를 선택
- "Create" 버튼 클릭

## STEP 3. Import Kibana dashboard object
- Management > Saved Objects > Import 클릭
- 현재 폴더 아래의 "kibana/kibana_export_everything.json" 파일 선택

## STEP 4. View imported dashboard
- Dashboard > KafkaMon-Dash(Resource) 클릭
- Kafka Broker 프로세스의 cpu, memory 사용량을 모니터링 가능



# [ETC]

## Create logstash Dockerfile using elastic official repository
- elastic 공식 github에서 제공하는 코드에는 Dockerfile이 없다.
- 대신 Makefile을 이용해서 Dockerfile을 생성하고,
- 이를 기준으로 image를 생성하도록 코드를 제공한다.
- 그래서 Dockerfile을 보려면 Makefile을 아래와 같이 실행해야 한다.

### Install python3.5 with virtualenv
- 기존 python2.x에 영향을 주지 않고, 별도의 virtualenv로 python3.5 환경을 구성
- https://www.atlantic.net/community/howto/installing-python-3/

```
wget https://www.python.org/ftp/python/3.5.4/Python-3.5.4.tgz
tar xvf Python-3.5.4.tgz
sudo yum groupinstall "Development Tools" "Development Libraries"
sudo yum -y install openssl-devel
sudo yum -y install readline-devel

cd Python-3.5.4
./configure --prefix=/home/rts/apps/Python-3.5.4/
make && make install

# virtualenv로 python3.5 환경 생성
virtualenv venv --python=/home/rts/apps/Python-3.5.4/bin/python3.5
source venv/bin/activate
python -V
```

### Create Dockerfile and build image
```
git clone https://github.com/elastic/logstash-docker.git
cd logstash-docker

```
#### - virtualenv로 설치한 python3.5의 경로를 지정해 준다.
- 기존 코드에서 virtualenv --python="python3.5.4경로지정"
```
vi Makefile
# The tests are written in Python. Make a virtualenv to handle the dependencies.
venv: requirements.txt
        test -d venv || virtualenv --python=/home/rts/apps/Python-3.5.4/bin/python3.5 venv
        pip install -r requirements.txt
        touch venv
```

#### - logstash 용 Dockerfile 확인하기

```
make test
```
- Dockerfile은 정상적으로 생성됨 (build/logstash/Dockerfile)
- docker build 시에 아래와 같은 오류가 발생함. (나중에 확인 필요)
```
docker run --rm -i \
  -v /home/rts/apps/logstash-docker/build/logstash/env2yaml:/usr/local/src/env2yaml \
  golang:env2yaml
/usr/bin/env: python3: No such file or directory
docker build --pull -t docker.elastic.co/logstash/logstash: build/logstash
invalid argument "docker.elastic.co/logstash/logstash:" for t: Error parsing reference: "docker.elastic.co/logstash/logstash:" is not a valid repository/tag
See 'docker build --help'.
make: *** [build] Error 125
```

### - 생성된 Dockerfile 참고
```
# This Dockerfile was generated from templates/Dockerfile.j2
FROM centos:7
LABEL maintainer "Elastic Docker Team <docker@elastic.co>"

# Install Java and the "which" command, which is needed by Logstash's shell
# scripts.
RUN yum update -y && yum install -y java-1.8.0-openjdk-devel which && \
    yum clean all

# Provide a non-root user to run the process.
RUN groupadd --gid 1000 logstash && \
    adduser --uid 1000 --gid 1000 \
      --home-dir /usr/share/logstash --no-create-home \
      logstash

# Add Logstash itself.
RUN curl -Lo - https://artifacts.elastic.co/downloads/logstash/logstash-.tar.gz | \
    tar zxf - -C /usr/share && \
    mv /usr/share/logstash- /usr/share/logstash && \
    chown --recursive logstash:logstash /usr/share/logstash/ && \
    ln -s /usr/share/logstash /opt/logstash

ENV ELASTIC_CONTAINER true
ENV PATH=/usr/share/logstash/bin:$PATH

# Provide a minimal configuration, so that simple invocations will provide
# a good experience.
ADD config/logstash.yml config/log4j2.properties /usr/share/logstash/config/
ADD pipeline/default.conf /usr/share/logstash/pipeline/logstash.conf
RUN chown --recursive logstash:logstash /usr/share/logstash/config/ /usr/share/logstash/pipeline/

# Ensure Logstash gets a UTF-8 locale by default.
ENV LANG='en_US.UTF-8' LC_ALL='en_US.UTF-8'

# Place the startup wrapper script.
ADD bin/docker-entrypoint /usr/local/bin/
RUN chmod 0755 /usr/local/bin/docker-entrypoint

USER logstash

RUN cd /usr/share/logstash && LOGSTASH_PACK_URL=https://artifacts.elastic.co/downloads/logstash-plugins logstash-plugin install x-pack

ADD env2yaml/env2yaml /usr/local/bin/

EXPOSE 9600 5044

ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
```
