---
title: "Upload CSV Data to Elasticsearch"
date: 2021-12-28
---

Recently, I've been working on understanding and detecting [Log4j](https://logging.apache.org/log4j/2.x/) vulnerability using Elasticsearch. If you want to know more about this vulnerability, I would suggest read the blog series <https://www.securitynik.com/2021/12/beginning-log4-shell-understanding.html> by Nik Alleyne on his blog [securitynik.com](https://www.securitynik.com/). 

To detect outbound traffic going to IOC's related to Log4j, needed to upload a csv data to Elasticsearch. To achieve that I followed the following steps:

### 1. First create a new index `logs-threat-intel`  
Using the `Dev Tools` in Kibana, isse the Create Index API
```
PUT /logs-threat-intel
{
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "properties": {
      "ioc.ip": { "type": "ip" },
      "ioc.url": { "type": "keyword" },
      "threatintel.indicator.signature": {"type": "keyword"},
      "@timestamp":{ "type" : "date", "format" : "strict_date_optional_time_nanos"},
      "event.ingested":{ "type" : "date", "format" : "strict_date_optional_time_nanos"},
      "event.module": {"type": "keyword"},
      "event.category": {"type": "keyword"},
      "event.type": {"type": "keyword"},
      "event.kind": {"type": "keyword"}
    }
  }
}
```

### 2. Create a new Ingest Node Pipeline `logs-threat-intel-pipeline`  
[Ingest pipelines](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html) let you perform common transformations on your data before indexing. For example, you can use pipelines to remove fields, extract values from text, and enrich your data.

```
PUT _ingest/pipeline/logs-threat-intel-pipeline
{
  "description": "Ingest Pipeline for the index logs-threat-intel",
  "processors": [
    {
      "set": {
        "field": "@timestamp",
        "value": "{{_ingest.timestamp}}"
      }
    },
        {
      "set": {
        "field": "event.ingested",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "set": {
         "field": "event.module",
         "value": "threatintel"
      }
    },
    {
      "set": {
      "field": "event.category",
      "value": "threat"
      }
    },
    {
      "set": {
        "field": "event.type",
        "value": "indicator"
      }
    },
    {
      "set": {
         "field": "event.kind",
         "value": "enrichment"
      }
    }
  ]
}
```

### 3. Next, map the Ingest Node Pipeline created in Step 2 with the index created in Step 1. 
```
PUT logs-threat-intel/_settings
{
  "index.default_pipeline": "logs-threat-intel-pipeline"
}
```

### 4. Example on how to ingest new IOC's

```
POST logs-threat-intel/_doc/1
{
    "ioc.ip": "<IPv4/IPv6>"
}
```
```
POST logs-threat-intel/_doc/2
{
    "ioc.url": "<URL>"
}
```
Replace the `IPv4/IPv6` and `URL` with respective IP or urls. Above technique works, but not quite scalable if we need ingest a big list of IOC's. As an example, we need to ingest [Log4j CSV](https://raw.githubusercontent.com/Azure/Azure-Sentinel/master/Sample%20Data/Feeds/Log4j_IOC_List.csv) downloaded from [Microsoft blog](https://www.microsoft.com/security/blog/2021/12/11/guidance-for-preventing-detecting-and-hunting-for-cve-2021-44228-log4j-2-exploitation/#IOC). To achieve that, wrote this 4 liner bash script:

```bash
while read f1
do        
   curl -k -X POST 'https://<ELASTICSEARCH_URL/ELASTICSEARCH_IP>:9200/logs-threat-intel/_doc/?pretty' -H "Content-Type: application/json" -u <USERNAME>:<PASSWORD> -d "{ \"ioc.ip\": \"$f1\", \"threatintel.indicator.signature\": \"log4j\"}"
   done > /dev/null 2>&1 & < log4j-ioc.csv
```

Above script reads the contents of the file `log4j-ioc.csv` one at a time and ingest to elasticsearch under the index `logs-threat-intel`.

### References:  
- Create index API: <https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html>  
- Create or update pipeline API: <https://www.elastic.co/guide/en/elasticsearch/reference/current/put-pipeline-api.html>