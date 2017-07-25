# Monitoring Server resource status using metricbeats
- https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-getting-started.html

## STEP 1. Install on the server which you want to monitor
### Download metricbeats
- 설치파일을 다운받고 압축을 해제한다. (설치완료!)
```
> cd ~
> wget https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-5.4.3-linux-x86_64.tar.gz
> tar xvf metricbeat-5.4.3-linux-x86_64.tar.gz
```

## STEP 2. Configuring metricbeats
```yml
metricbeat.modules:
#------------------------------- System Module -------------------------------
- module: system
  metricsets:
    # CPU stats
    - cpu

    # System Load stats
    - load

    # Per CPU core stats
    - core

    # IO stats
    - diskio

    # Per filesystem stats
    - filesystem

    # File system summary stats
    - fsstat

    # Memory stats
    - memory

    # Network stats
    - network

    # Per process stats
    - process

    # Sockets (linux only)
  enabled: true
  period: 10s
  processes: ['.*']
  cpu_ticks: false

#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]
  # index를 지정할 수도 있으나, Default값인 "metricbeat-*"으로 저장되어야
  # metricbeat에서 제공하는 kiban dashboard를 사용할 수 있다.
#  index: "broker1-metric"
```


## STEP 3. Run metricbeats
```
> ./metricbeat -e -c metricbeat.yml -d "publish"
```

### STEP 4. Import Metricbeat dashboard to elasticsearch
- metricbeat에서 기본으로 제공하는 dashboard를 이용하여 Kibana에서 시각화 가능
```
> ./scripts/import_dashboards -es http://169.56.124.19:9200
```

### STEP 4. Visualiza metricbeat dashboard on the kibana dashboard ui
- Dashboard를 클릭하면, import된 다양한 dashboard가 목록으로 표시된다.
- 전체를 개략적으로 보려면 "Metricbeat system overview"를 선택
