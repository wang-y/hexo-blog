---
title: Elasticsearch IK+拼音分词配置
date: 2017-08-28 14:38:07
categories: 技术
tags:
    - ElasticSearch
---

# IK分词器安装

关于IK分词器的介绍不再多少，一言以蔽之，IK分词是目前使用非常广泛分词效果比较好的中文分词器。做ES开发的，中文分词十有八九使用的都是IK分词器。

<!-- more -->

下载地址：

[[IK](https://github.com/medcl/elasticsearch-analysis-ik)]

配置之前**关闭elasticsearch**，配置完成以后再重启。 

IK的版本要和当前ES的版本一致，README中有说明。

下载之后进入到elasticsearch-analysis-pinyin-master目录，mvn打包(没有安装maven的自行安装)，运行命令：

```
mvn package
```

打包成功以后，会生成一个target文件夹，在elasticsearch-analysis-ik-master/target/releases目录下，找到elasticsearch-analysis-ik-5.x.x.zip，这就是我们需要的安装文件。解压elasticsearch-analysis-ik-5.x.x.zip，得到下面内容：

```
config
commons-codec-1.9.jar
commons-logging-1.2.jar
elasticsearch-analysis-ik-5.x.x.jar
httpclient-4.5.2.jar
httpcore-4.4.4.jar
plugin-descriptor.properties
```

然后在elasticsearch-5.x.x/plugins目录下新建一个文件夹ik，把elasticsearch-analysis-ik-5.x.x.zip解压后的文件拷贝到elasticsearch-5.x.x/plugins/ik目录下.

# pinyin分词器安装

下载地址：

[[PINYIN](https://github.com/medcl/elasticsearch-analysis-pinyin)]

安装方式同IK分词器。

# 分词测试

## IK分词测试

创建一个索引:

```
curl -XPUT "http://localhost:9200/test"
```

测试分词效果:

```
curl -XPOST "http://localhost:9200/test/_analyze?analyzer=ik_max_word&text=中华人民共和国国歌"
```

分词结果:

```
{
    "tokens": [{
        "token": "中华人民共和国",
        "start_offset": 0,
        "end_offset": 7,
        "type": "CN_WORD",
        "position": 0
    }, {
        "token": "中华人民",
        "start_offset": 0,
        "end_offset": 4,
        "type": "CN_WORD",
        "position": 1
    }, {
        "token": "中华",
        "start_offset": 0,
        "end_offset": 2,
        "type": "CN_WORD",
        "position": 2
    }, {
        "token": "华人",
        "start_offset": 1,
        "end_offset": 3,
        "type": "CN_WORD",
        "position": 3
    }, {
        "token": "人民共和国",
        "start_offset": 2,
        "end_offset": 7,
        "type": "CN_WORD",
        "position": 4
    }, {
        "token": "人民",
        "start_offset": 2,
        "end_offset": 4,
        "type": "CN_WORD",
        "position": 5
    }, {
        "token": "共和国",
        "start_offset": 4,
        "end_offset": 7,
        "type": "CN_WORD",
        "position": 6
    }, {
        "token": "共和",
        "start_offset": 4,
        "end_offset": 6,
        "type": "CN_WORD",
        "position": 7
    }, {
        "token": "国",
        "start_offset": 6,
        "end_offset": 7,
        "type": "CN_CHAR",
        "position": 8
    }, {
        "token": "国歌",
        "start_offset": 7,
        "end_offset": 9,
        "type": "CN_WORD",
        "position": 9
    }]
}
```

使用ik_smart分词:

```
curl -XPOST "http://localhost:9200/test/_analyze?analyzer=ik_smart&text=中华人民共和国国歌"
```

分词结果:

```
{
    "tokens": [{
        "token": "中华人民共和国",
        "start_offset": 0,
        "end_offset": 7,
        "type": "CN_WORD",
        "position": 0
    }, {
        "token": "国歌",
        "start_offset": 7,
        "end_offset": 9,
        "type": "CN_WORD",
        "position": 1
    }]
}
```

## 拼音分词测试

测试拼音分词:

```
curl -XPOST "http://localhost:9200/test/_analyze?analyzer=pinyin&text=张学友"
```

分词结果:

```
{
    "tokens": [{
        "token": "zhang",
        "start_offset": 0,
        "end_offset": 1,
        "type": "word",
        "position": 0
    }, {
        "token": "xue",
        "start_offset": 1,
        "end_offset": 2,
        "type": "word",
        "position": 1
    }, {
        "token": "you",
        "start_offset": 2,
        "end_offset": 3,
        "type": "word",
        "position": 2
    }, {
        "token": "zxy",
        "start_offset": 0,
        "end_offset": 3,
        "type": "word",
        "position": 3
    }]
}
```

# IK+pinyin分词配置

## 创建索引与分析器设置

创建一个索引，并设置index分析器相关属性:

```
curl -XPUT "http://localhost:9200/test/" -d'
{
    "index": {
        "analysis": {
            "analyzer": {
                "ik_pinyin_analyzer": {
                    "type": "custom",
                    "tokenizer": "ik_smart",
                    "filter": ["my_pinyin", "word_delimiter"]
                }
            },
            "filter": {
                "my_pinyin": {
                    "type": "pinyin",
                    "first_letter": "prefix",
                    "padding_char": " "
                }
            }
        }
    }
}'
```

创建一个type并设置mapping:

```
curl -XPOST http://localhost:9200/test/folks/_mapping -d'
{
    "folks": {
        "properties": {
            "name": {
                "type": "keyword",
                "fields": {
                    "pinyin": {
                        "type": "text",
                        "store": "no",
                        "term_vector": "with_positions_offsets",
                        "analyzer": "ik_pinyin_analyzer",
                        "boost": 10
                    }
                }
            }
        }
    }
}'
```

## 索引测试文档

索引2份测试文档。

文档1:

```
curl -XPOST http://localhost:9200/test/folks/andy -d '
{
    "name":"刘德华"
}'
```

文档2:

```
curl -XPOST http://localhost:9200/test/folks/tina -d '
{
    "name":"中华人民共和国国歌"
}'
```

## 拼音分词测试

下面四条命命令都可以匹配”刘德华”

```
curl -XPOST "http://localhost:9200/test/folks/_search?q=name.pinyin:liu"

curl -XPOST "http://localhost:9200/test/folks/_search?q=name.pinyin:de"

curl -XPOST "http://localhost:9200/test/folks/_search?q=name.pinyin:hua"

curl -XPOST "http://localhost:9200/test/folks/_search?q=name.pinyin:ldh"
```

## IK分词测试

```
curl -XPOST "http://localhost:9200/test/_search?pretty" -d'
{
  "query": {
    "match": {
      "name.pinyin": "国歌"
    }
  },
  "highlight": {
    "fields": {
      "name.pinyin": {}
    }
  }
}'
```

返回结果:

```
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 16.698704,
    "hits" : [
      {
        "_index" : "medcl",
        "_type" : "folks",
        "_id" : "tina",
        "_score" : 16.698704,
        "_source" : {
          "name" : "中华人民共和国国歌"
        },
        "highlight" : {
          "name.pinyin" : [
            "<em>中华人民共和国</em><em>国歌</em>"
          ]
        }
      }
    ]
  }
}
```

说明IK分词器起到了效果。

## 拼音+IK分词测试

```
curl -XPOST "http://localhost:9200/test/_search?pretty" -d'
{
  "query": {
    "match": {
      "name.pinyin": "zhonghua"
    }
  },
  "highlight": {
    "fields": {
      "name.pinyin": {}
    }
  }
}'
```

返回结果:

```
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 5.9814634,
    "hits" : [
      {
        "_index" : "medcl",
        "_type" : "folks",
        "_id" : "tina",
        "_score" : 5.9814634,
        "_source" : {
          "name" : "中华人民共和国国歌"
        },
        "highlight" : {
          "name.pinyin" : [
            "<em>中华人民共和国</em>国歌"
          ]
        }
      },
      {
        "_index" : "medcl",
        "_type" : "folks",
        "_id" : "andy",
        "_score" : 2.2534127,
        "_source" : {
          "name" : "刘德华"
        },
        "highlight" : {
          "name.pinyin" : [
            "<em>刘德华</em>"
          ]
        }
      }
    ]
  }
}
```

**使用pinyin分词以后，原始的字段搜索要加上.pinyin后缀，搜索原始字段没有返回结果!**

