---
title: Elasticsearch备份
date: 2018-02-06 15:58:29
categories: 技术
tags: 
    - ElasticSearch
---

**所需环境**

- NodeJS

```shell
 npm install elasticdump
```

**index to index**

```shell
./elasticdump --input=http://192.168.1.100:9200/source_demo --output=http://192.168.1.101:9200/target_demo --type=analyzer 
./elasticdump --input=http://192.168.1.100:9200/source_demo --output=http://192.168.1.101:9200/target_demo --type=mapping
./elasticdump --input=http://192.168.1.100:9200/source_demo --output=http://192.168.1.101:9200/target_demo --type=data
```

**index to file**

```shell
./elasticdump --input=http://192.168.1.100:9200/source_demo --output=/home/es-export/analyzer.log --type=analyzer 
./elasticdump --input=http://192.168.1.100:9200/source_demo --output=/home/es-export/mapping.log --type=mapping
./elasticdump --input=http://192.168.1.100:9200/source_demo --output=/home/es-export/data.log --type=data
```

**file to index**

```shell
./elasticdump --input=/home/es-export/analyzer.log --output=http://192.168.1.101:9200/target_demo --type=analyzer 
./elasticdump --input=/home/es-export/mapping.log  --output=http://192.168.1.101:9200/target_demo --type=mapping
./elasticdump --input=/home/es-export/data.log     --output=http://192.168.1.101:9200/target_demo --type=data
```

