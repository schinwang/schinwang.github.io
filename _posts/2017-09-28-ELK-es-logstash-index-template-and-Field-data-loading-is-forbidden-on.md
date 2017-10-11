---
bg: "elastic_stack.jpeg"
layout: post
title: "troubleshooting:'Field data loading is forbidden on [field]' with es-logstash index template"
summary: ""
tags: ['ELK']
categories: ELK
---

​        我们的日活等数据除了友盟等三方服务提供外，还通过nginx日志来统计，但是最近数据统计总是不准确，通过kibana聚合时也会报`Field data loading is forbidden on [uuid]`错误，detail as the following:

```shell
Visualize: Field data loading is forbidden on [uuid]

Error: Request to Elasticsearch failed: {"error":{"root_cause":[{"type":"illegal_state_exception","reason":"Field data loading is forbidden on [uuid]"}],"type":"search_phase_execution_exception","reason":"all shards failed","phase":"query","grouped":true,"failed_shards":[{"shard":0,"index":"logstash-nginx-log-2017.09.27","node":"uTHxVRMCQhWwlA5uZzMAVw","reason":{"type":"illegal_state_exception","reason":"Field data loading is forbidden on [uuid]"}}]}}
KbnError@http://kibana.qyvideo.net/bundles/commons.bundle.js:61164:30
RequestFailure@http://kibana.qyvideo.net/bundles/commons.bundle.js:61197:19
http://kibana.qyvideo.net/bundles/kibana.bundle.js:88304:57
http://kibana.qyvideo.net/bundles/commons.bundle.js:63691:28
http://kibana.qyvideo.net/bundles/commons.bundle.js:63660:31
map@[native code]
map@http://kibana.qyvideo.net/bundles/commons.bundle.js:63659:34
callResponseHandlers@http://kibana.qyvideo.net/bundles/kibana.bundle.js:88276:26
http://kibana.qyvideo.net/bundles/kibana.bundle.js:87783:37
processQueue@http://kibana.qyvideo.net/bundles/commons.bundle.js:41809:31
http://kibana.qyvideo.net/bundles/commons.bundle.js:41825:40
$digest@http://kibana.qyvideo.net/bundles/commons.bundle.js:42864:37
$apply@http://kibana.qyvideo.net/bundles/commons.bundle.js:43161:32
done@http://kibana.qyvideo.net/bundles/commons.bundle.js:37610:54
completeRequest@http://kibana.qyvideo.net/bundles/commons.bundle.js:37808:16
requestLoaded@http://kibana.qyvideo.net/bundles/commons.bundle.js:37749:25
```

### **分析**

这个问题在github [Field data loading is forbidden on [FIELDNAME] #15267](https://github.com/elastic/elasticsearch/issues/15267)被讨论过，引用**[clintongormley](https://github.com/clintongormley) **commented [on 11 Dec 2015](https://github.com/elastic/elasticsearch/issues/15267#issuecomment-163894259)

```
This is not a bug. It is a safeguard. The logstash template now disables fielddata loading where it makes sense, eg see https://github.com/logstash-plugins/logstash-output-elasticsearch/blob/master/lib/logstash/outputs/elasticsearch/elasticsearch-template.json#L15

You get this message when you try to sort or run aggregations or scripts on analyzed fields. Fulfilling this request would cause massive amounts of memory usage on your cluster, and it almost certainly isn't what you want anyway, eg Field data loading is forbidden on path"... You don't want to aggregate on the analyzed field path, you want to aggregate on the not analyzed field path.raw, which uses doc values not heap memory.
```

所以说出现这个问题的原因是因为聚合的某个字段被analyse了, ES为了提高搜索效率，会根据自己的分词逻辑(比如按空白符自动分割)将字符串进行分割索引

​        未被分词的字段以doc values的方式存储，这样在es进行查询的时候可以通过自己的doc search逻辑进行高效的索引，而被分词的字段则需要进行全局查找，这可能会占用大量内存，为了防止因为对分词字段的查找而导致的性能下降或崩溃，es引入了一种保护机制，即拒绝在聚合查询的时候对分词字段进行索引，同时，对于分词字段，es会自动生成field.raw 字段来采用doc values的方式存储，这样用户可以用filed.raw 代替field字段用于聚合分析

​        通常情况我们都是使用ELK来进行日志分析，而出现这种问题的一般场景也是因为logstash往es灌入日志时，mapping对某个字段设置为了analyzed

```shell
$curl http://localhost:9200/_mapping
```

找到出现问题的索引，会发现对应field字段mapping如下

```shell
"uuid":{"type":"string","norms":{"enabled":false},"fielddata":{"format":"disabled"},"fields":{"raw":{"type":"string","index":"not_analyzed","ignore_above":256}}}
```

### **原因**

​        出现这种问题是因为elasticsearch对logstash有一套默认的模板

```shell
$curl http://localhost:9200/_template
```

通常string type的数据默认是被分词的，

### **解决**

可以重新编辑一个es-template.json

```json
{
    "template" : "logstash-*",
    "settings" : {
        "index.refresh_interval" : "5s"
    },
    "mappings": {
        "_default_": {
            "_all": {
                "enabled": true, 
                "omit_norms": true
            }, 
            "dynamic_templates": [
                {
                    "message_field": {
                        "mapping": {
                            "doc_values": true, 
                            "fielddata": {
                                "format": "disabled"
                            }, 
                            "index": "not_analyzed", 
                            "omit_norms": true, 
                            "type": "string"
                        }, 
                        "match": "message", 
                        "match_mapping_type": "string"
                    }
                }, 
                {
                    "string_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "index": "not_analyzed", 
                            "omit_norms": true, 
                            "type": "string"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "string"
                    }
                }, 
                {
                    "float_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "type": "float"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "float"
                    }
                }, 
                {
                    "double_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "type": "double"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "double"
                    }
                }, 
                {
                    "byte_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "type": "byte"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "byte"
                    }
                }, 
                {
                    "short_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "type": "short"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "short"
                    }
                }, 
                {
                    "integer_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "type": "integer"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "integer"
                    }
                }, 
                {
                    "long_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "type": "long"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "long"
                    }
                }, 
                {
                    "date_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "type": "date"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "date"
                    }
                }, 
                {
                    "geo_point_fields": {
                        "mapping": {
                            "doc_values": true, 
                            "type": "geo_point"
                        }, 
                        "match": "*", 
                        "match_mapping_type": "geo_point"
                    }
                }
            ], 
            "properties": {
                "@timestamp": {
                    "format": "strict_date_optional_time||epoch_millis", 
                    "type": "date",
                    "doc_values": true
                }, 
                "@version": {
                    "index": "not_analyzed", 
                    "type": "string",
                    "doc_values" : true
                }, 
                "geoip": {
                    "type": "object",
                    "dynamic": "true",
                    "properties": {
                        "ip": {
                            "type": "ip",
                            "doc_values" : true
                        }, 
                        "latitude": {
                            "type": "float",
                            "doc_values" : true
                        }, 
                        "location": {
                            "type": "geo_point",
                            "doc_values" : true
                        }, 
                        "longitude": {
                            "type": "float",
                            "doc_values" : true
                        }
                    }
                }
            }
        }
    }
}
```

然后更新ES的logstash index模板文件

```shell
curl -XPUT http://localhost:9200/_template/template-name?pretty -d @es-template.json
```

此时再去查询时，会发现已经更新为最新的template了

```shell
curl http://lcoalhost:9200/_template
```

### **解决+**

虽然我们更新了默认的index template，但是要注意的是logstash的配置文件中 template_overwrite 不能设置为true（可以注释掉，默认是false）, 否则更新的模板还是有可能被覆盖的

```shell
output{
    elasticsearch{
        hosts => ["10.19.24.94:9200", "10.19.24.100:9200", "10.19.24.91:9200"]
        #template_overwrite => true
        index => "logstash-%{type}-%{+YYYY.MM.dd}"
    }
    #stdout{codec => rubydebug}
}
```

或者干脆把模板加到配置文件中，保证每次建的index都是自己想要的

```shell
output{
    elasticsearch{
        hosts => ["10.19.24.94:9200", "10.19.24.100:9200", "10.19.24.91:9200"]
        template => "/data/deploy/logstash/es-template.json"
        template_overwrite => true
        index => "logstash-%{type}-%{+YYYY.MM.dd}"
    }
    #stdout{codec => rubydebug}
}
```



references:

[github-issue: Field data loading is forbidden on [FIELDNAME] #15267](https://github.com/elastic/elasticsearch/issues/15267)

[ELKstack中文指南-保存进 Elasticsearch](https://kibana.logstash.es/content/logstash/plugins/output/elasticsearch.html)

[Little Logstash Lessons: Using Logstash to help create an Elasticsearch mapping template](https://www.elastic.co/blog/logstash_lesson_elasticsearch_mapping)

[elasticsearch-analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/analysis-analyzers.html)