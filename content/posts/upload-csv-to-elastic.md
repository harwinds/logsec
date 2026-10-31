---
title: "Upload CSV Data to Elasticsearch"
date: 2021-12-28
description: "Load a CSV of Log4j IOCs into Elasticsearch using a custom index, an ingest pipeline and a short bash loop, to detect outbound traffic to them."
tags: [elastic, kibana, threat-intel, log4j, detection]
---

Recently, I've been working on understanding and detecting the [Log4j](https://logging.apache.org/log4j/2.x/) vulnerability using Elasticsearch. If you want to know more about this vulnerability, I would suggest reading the blog series <https://www.securitynik.com/2021/12/beginning-log4-shell-understanding.html> by Nik Alleyne on his blog [securitynik.com](https://www.securitynik.com/). 

To detect outbound traffic going to IOCs related to Log4j, I needed to upload CSV data to Elasticsearch. To achieve that, I followed these steps:

### 1. First create a new index `logs-threat-intel`  
Using `Dev Tools` in Kibana, issue the Create Index API call
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

### 4. Example of how to ingest new IOCs

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
Replace `IPv4/IPv6` and `URL` with the respective IPs or URLs. This technique works, but it doesn't scale if we need to ingest a big list of IOCs. As an example, we need to ingest the [Log4j CSV](https://raw.githubusercontent.com/Azure/Azure-Sentinel/master/Sample%20Data/Feeds/Log4j_IOC_List.csv) downloaded from [Microsoft blog](https://www.microsoft.com/security/blog/2021/12/11/guidance-for-preventing-detecting-and-hunting-for-cve-2021-44228-log4j-2-exploitation/#IOC). To achieve that, I wrote this 4-line bash script:

```bash
while read f1
do        
   curl -k -X POST 'https://<ELASTICSEARCH_URL/ELASTICSEARCH_IP>:9200/logs-threat-intel/_doc/?pretty' -H "Content-Type: application/json" -u <USERNAME>:<PASSWORD> -d "{ \"ioc.ip\": \"$f1\", \"threatintel.indicator.signature\": \"log4j\"}"
done < log4j-ioc.csv > /dev/null 2>&1 &
```

The script reads the file `log4j-ioc.csv` one line at a time and ingests each IOC into Elasticsearch under the index `logs-threat-intel`.

### References:  
- Create index API: <https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html>  
- Create or update pipeline API: <https://www.elastic.co/guide/en/elasticsearch/reference/current/put-pipeline-api.html>