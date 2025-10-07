```yaml
services:

caldera:

build:

context: ./caldera

args:

- VARIANT=slim

network_mode: host

ports:

- "8888:8888"

volumes:

- ./caldera/conf/local.yml:/usr/src/app/conf/local.yml:ro

entrypoint: ["python3"]

command: ["server.py", "--insecure", "--build"]

depends_on:

- wildcat

  

wildcat:

build: ./wildcatdam

network_mode: host

ports:

- "502:502"

volumes:

- ./wildcatdam/config/wildcat_dam.yaml:/app/config/wildcat_dam.yaml:ro

command: ["python3", "/app/wildcat_dam.py"]

  

zeek:

image: zeek/zeek:latest

network_mode: host

cap_add:

- NET_RAW

- NET_ADMIN

command: ["zeek", "-i", "lo", "-b", "/opt/zeek/share/zeek/site/local.zeek"]

volumes:

- ./zeek/local.zeek:/opt/zeek/share/zeek/site/local.zeek:ro

- ./zeek/networks.cfg:/opt/zeek/etc/networks.cfg:ro

- ./zeek_logs:/opt/zeek/logs/current

  

elasticsearch:

image: elasticsearch:9.1.4

network_mode: host

ports:

- "9200:9200"

- "9300:9300"

volumes:

- ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro

- es_data:/usr/share/elasticsearch/data

environment:

- ES_JAVA_OPTS=-Xms512m -Xmx512m

- bootstrap.memory_lock=true

- discovery.type=single-node

- xpack.security.enabled=false

ulimits:

memlock: -1

  

logstash:

image: focker.ir/logstash/logstash:9.1.4

network_mode: host

ports:

- "5044:5044"

- "9600:9600"

volumes:

- ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro

- ./logstash/pipeline:/usr/share/logstash/pipeline:ro

environment:

- LS_JAVA_OPTS=-Xms256m -Xmx256m

depends_on:

- elasticsearch

  

kibana:

image: focker.ir/kibana/kibana:9.1.4

network_mode: host

ports:

- "5601:5601"

volumes:

- ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml:ro

depends_on:

- elasticsearch

  

filebeat:

image: docker.elastic.co/beats/filebeat:9.1.4

network_mode: host

user: root

volumes:

- ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro

- ./zeek_logs:/zeek_logs:ro

depends_on:

- zeek

- elasticsearch

  

volumes:

es_data:
```