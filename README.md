elasticsearch-cdc plugin supports asynchronously synchronizing elasticsearch cdc data to kafka
# elasticsearch-cdc plugin design

![image](https://user-images.githubusercontent.com/18043146/130311896-6dfafd65-ab19-4727-9b90-750edbd7f350.png)
To achieve higher throughput, messages are sent to kafka asynchronously

# Installation
1. Compile and package
```bash
mvn clean package
```
2. Remove previous version (if installed before)
```bash
${elasticsearch_home}/bin/elasticsearch-plugin remove elasticsearch-cdc
```
3. Install
```bash
${elasticsearch_home}/bin/elasticsearch-plugin install file:///${elasticsearch_cdc_home_dir}/target/releases/elasticsearch-cdc-1.0-SNAPSHOT.zip
```
4. Configure ES's javax related permissions. Add related java configuration to `/etc/elasticsearch/jvm.options` file
```bash
-Djava.security.policy=/your_elasticsearch_home/plugins/elasticsearch-cdc/plugin-security.policy
```
5. Restart elasticsearch
```bash
service restart elasticsearch
```
Note: For ES distributed clusters, the plugin should be installed on every node

# Usage
## Set cluster properties
```json
PUT _cluster/settings
{
  "persistent": {
    "indices.cdc.request.timeout.ms": 30000,
    "indices.cdc.send.buffer.bytes": 131072,
    "indices.cdc.acks": "all",
    "indices.cdc.compression.type": "none",
    "indices.cdc.receive.buffer.bytes": 32768,
    "indices.cdc.batch.size": 16384,
    "indices.cdc.linger.ms": 1000,
    "indices.cdc.buffer.memory": 33554432,
    "indices.cdc.bootstrap.servers": "your_kafka_server1:9092,your_kafka_server2:9092,your_kafka_server3:9092",
    "indices.cdc.max.request.size": 1048576,
    "indices.cdc.max.in.flight.requests.per.connection": 10,
    "indices.cdc.retry.backoff.ms": 100,
    "indices.cdc.retries": 100,
    "indices.cdc.max.block.ms": 86400000
  }
}
```
## When creating an index, enable CDC and initialize related properties
```json
PUT /index
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1,
    "index.cdc.enabled": true,
    "index.cdc.topic": "cdc_test",
    "index.cdc.pk.column": "doc_id",
    "index.refresh_interval": "100s"
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text"
      },
      "doc_id": {
        "type": "integer"
      },
      "age": {
        "type": "integer"
      },
      "address": {
        "type": "keyword"
      },
      "keywords": {
        "type": "nested",
        "properties": {
          "keyword": {
            "type": "keyword"
          },
          "frequency": {
            "type": "integer"
          }
        }
      }
    }
  }
}
```


# Index Level Property Description

| Property Value            | Default | Description                                      |
| ------------------------- | ------- | ------------------------------------------------ |
| index.cdc.enabled         | false   | Whether to enable CDC monitoring                 |
| index.cdc.topic           | ""      | Specifies the topic name for writing CDC data    |
| index.cdc.pk.column       | ""      | Primary key column                               |
| index.cdc.exclude.columns | ""      | Exclude columns, use commas to separate multiple columns |
| index.cdc.alias           | ""      | Alias of the index, when set, CDC will use this alias



# Cluster Level Configuration Properties

| Property Value                                    | Default           | Description                                       |
| ------------------------------------------------- | ----------------- | ------------------------------------------------- |
| indices.cdc.bootstrap.servers                     | ""                | Same as producer bootstrap.servers                |
| indices.cdc.batch.size                            | 16384             | Same as producer batch.size                       |
| indices.cdc.acks                                  | all               | Same as producer acks                             |
| indices.cdc.buffer.memory                         | 32 * 1024 * 1024  | Same as producer buffer.memory                    |
| indices.cdc.compression.type                      | none              | Same as producer compression.type                 |
| indices.cdc.request.timeout.ms                    | 30 * 1000         | Same as producer request.timeout.ms               |
| indices.cdc.max.request.size                      | 1024 * 1024       | Same as producer max.request.size                 |
| indices.cdc.retries                               | 0                 | Same as producer retries                          |
| indices.cdc.retry.backoff.ms                      | 100               | Same as producer retry.backoff.ms                 |
| indices.cdc.send.buffer.bytes                     | 131072            | Same as producer send.buffer.bytes                |
| indices.cdc.receive.buffer.bytes                  | 32768             | Same as producer receive.buffer.bytes             |
| indices.cdc.linger.ms                             | 0                 | Same as producer linger.ms                        |
| indices.cdc.max.in.flight.requests.per.connection | 5                 | Same as producer max.in.flight.requests.per.connection |
| indices.cdc.max.block.ms                          | 0                 | Same as producer max.block.ms                     |
| indices.cdc.producer.nums                         | core count of ES server | Number of kafka producer threads per node                  |



## Note
action values: delete - 0, insert - 1, update - 2



# Testing

| Operation                                        | Request Data                                                     | kafka key              | kafka value                                                  |
| ------------------------------------------------ | ------------------------------------------------------------ | ---------------------- | ------------------------------------------------------------ |
| insert with POST                                 | POST /index/_create/1<br/>{"doc_id":1,"content":"peace and hope","age":12,"address":"bj","keywords":[{"keyword":"peace","frequency":10},{"keyword":"hope","frequency":5}]} | 1                      | {"op":1,"index":"index","content":{"address":"bj","keywords":[{"keyword":"peace","frequency":10},{"keyword":"hope","frequency":5}],"doc_id":1,"content":"peace and hope","age":12},"ts":1629530725380} |
| insert with PUT                                  | PUT /index/_doc/2<br/>{"doc_id": 2,"content":"peace and hope 2", "age": 12, "address": "bj", "keywords": [{"keyword": "peace", "frequency": 1}, {"keyword": "hope", "frequency": 1}]} | 2                      | {"op":1,"index":"index","content":{"address":"bj","keywords":[{"keyword":"peace","frequency":1},{"keyword":"hope","frequency":1}],"doc_id":2,"content":"peace and hope 2","age":12},"ts":1629531058056} |
| bulk insert                                      | PUT /_bulk<br/>{"index":{"_index":"index","_id":"3"}}<br/>{"doc_id": 3,"content":"peace and hope 3", "age": 13, "address": "bj", "keywords": [{"keyword": "peace", "frequency": 1}, {"keyword": "hope", "frequency": 1}]}<br/>{"index":{"_index":"index","_id":"4"}}<br/>{"doc_id": 4,"content":"peace and hope 4", "age": 14, "address": "bj", "keywords": [{"keyword": "peace", "frequency": 1}, {"keyword": "hope", "frequency": 1}]}<br/>{"index":{"_index":"index","_id":"5"}}<br/>{"doc_id": 5,"content":"peace and hope 5", "age": 15, "address": "bj", "keywords": [{"keyword": "peace", "frequency": 1}, {"keyword": "hope", "frequency": 1}]} | The keys for the three records are 5, 4, 3 respectively. | The corresponding values for the keys are respectively：<br/>{"op":1,"index":"index","content":{"address":"bj","keywords":[{"keyword":"peace","frequency":1},{"keyword":"hope","frequency":1}],"doc_id":5,"content":"peace and hope 5","age":15},"ts":1629531206042}<br/>{"op":1,"index":"index","content":{"address":"bj","keywords":[{"keyword":"peace","frequency":1},{"keyword":"hope","frequency":1}],"doc_id":4,"content":"peace and hope 4","age":14},"ts":1629531205887}<br/>{"op":1,"index":"index","content":{"address":"bj","keywords":[{"keyword":"peace","frequency":1},{"keyword":"hope","frequency":1}],"doc_id":3,"content":"peace and hope 3","age":13},"ts":1629531205885}<br/> |
| simple update                                    | POST /index/_update/1<br/>{"doc":{"doc_id":1,"address":"sjz"}} | 1                      | {"op":2,"index":"index","content":{"address":"sjz","keywords":[{"keyword":"peace","frequency":10},{"keyword":"hope","frequency":5}],"doc_id":1,"content":"peace and hope","age":12},"ts":1629531453903} |
| update by query and just update non-nested field | POST /index/_update_by_query<br/>{"script":{"source":"ctx._source.age++","lang":"painless"},"query":{"term":{"doc_id":1}}} | 1                      | {"op":2,"index":"index","content":{"address":"sjz","keywords":[{"keyword":"peace","frequency":10},{"keyword":"hope","frequency":5}],"doc_id":1,"content":"peace and hope","age":13},"ts":1629531523775} |
| update by query --update nested field            | POST /index/_update/1<br/>{"script":{"source":"for(e in ctx._source.keywords){if (e.frequency == 5) {e.frequency = 10;}}"}} | 1                      | {"op":2,"index":"index","content":{"address":"sjz","keywords":[{"keyword":"peace","frequency":10},{"keyword":"hope","frequency":10}],"doc_id":1,"content":"peace and hope","age":13},"ts":1629531711169} |
| update by query -- delete nested field           | POST /index/_update/1<br/>{"script":{"source":"ctx._source.keywords.removeIf(it -> it.frequency == 10);"}} | 1                      | {"op":2,"index":"index","content":{"address":"sjz","keywords":[],"doc_id":1,"content":"peace and hope","age":13},"ts":1629531763492} |
| simple delete                                    | DELETE /index/_doc/1<br/>                                    | 1                      | {"op":0,"index":"index","content":{"doc_id":"1"},"ts":1629531859497} |
| delete by query                                  | POST /index/_delete_by_query<br/>{"query":{"match":{"doc_id":2}}} | 2                      | {"op":0,"index":"index","content":{"doc_id":"2"},"ts":1629531944820} |


# Change CDC Configuration Test

```json
PUT index/_settings
{
  "index.cdc.enabled":false
}

PUT index/_settings
{
  "index.cdc.enabled":true,
  "index.cdc.exclude.columns": "age,content"
}

POST /index/_create/6
{"doc_id": 6,"content":"peace and hope", "age": 12, "address": "bj", "keywords": [{"keyword": "peace", "frequency": 10}, {"keyword": "hope", "frequency": 5}]}


The generated CDC message, in cdc_test, is as follows:
{"op":1,"index":"index","content":{"address":"bj","keywords":[{"keyword":"peace","frequency":10},{"keyword":"hope","frequency":5}],"doc_id":6},"ts":1629532594468}
The information for content and age has been excluded
```
