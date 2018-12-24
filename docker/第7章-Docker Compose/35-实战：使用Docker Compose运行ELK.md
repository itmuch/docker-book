# 实战：使用Docker Compose运行ELK

* ElasticSearch【存储】
* Logtash【日志聚合器】
* Kibana【界面】

答案：

```yaml
version: '2'
services:
 elasticsearch:
  image: elasticsearch
  # command: elasticsearch
  ports:
   - "9200:9200"   # REST API端口
   - "9300:9300"   # RPC端口
 logstash:
  image: logstash
  command: logstash -f /etc/logstash/conf.d/logstash.conf
  volumes:
   - ./config:/etc/logstash/conf.d
   - /opt/build:/opt/build
  ports:
   - "5000:5000"
 kibana:
  image: kibana
  environment:
   - ELASTICSEARCH_URL=http://elasticsearch:9200
  ports:
   - "5601:5601"
```

`logstash.conf` 参考示例：

```json
input {
  file {
    codec => json
    path => "/opt/build/*.json"
  }
}
filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp}\s+%{LOGLEVEL:severity}\s+\[%{DATA:service},%{DATA:trace},%{DATA:span},%{DATA:exportable}\]\s+%{DATA:pid}---\s+\[%{DATA:thread}\]\s+%{DATA:class}\s+:\s+%{GREEDYDATA:rest}" }
  }
}
output {
  elasticsearch {
    hosts => "elasticsearch:9200"
  }
}
```





## 参考文档

<https://docs.docker.com/compose/samples-for-compose/#samples-tailored-to-demo-compose>