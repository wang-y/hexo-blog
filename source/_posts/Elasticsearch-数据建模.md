---
title: Elasticsearch 数据建模
date: 2017-08-29 15:32:47
categories: 设计
tags:
    - ElasticSearch
---

# 数据建模(Modeling Your Data)

ES是一头不同寻常的野兽，尤其是当你来自SQL的世界时。它拥有很多优势：性能，可扩展性，准实时的搜索，以及对大数据的分析能力。并且，它很容易上手！只需要下载就能够开始使用它了。

但是它也不是魔法。为了更好的利用ES，你需要了解它从而让它能够满足你的需求。

<!--more-->

在ES中，处理实体之间的关系并不像关系型存储那样明显。在关系数据库中的黄金准则 - 数据规范化，在ES中并不适用。在处理关联关系，嵌套对象和父子关联关系中，我们会讨论几种可行方案的优点和缺点。

紧接着在为可扩展性而设计中，我们会讨论ES提供的一些用来快速灵活实现扩展的特性。对于扩展，并没有一个可以适用于所有场景的解决方案。你需要考虑数据是如何在你的系统中流转的，从而恰当地对你的数据进行建模。针对基于时间的数据比如日志事件或者社交数据流的方案比相对静态的文档集合的方案是十分不同的。

最后，我们会讨论一样在ES中不会扩展的东西。

# 处理关联关系(Handling Relationships)

在真实的世界中，关联关系很重要：博客文章有评论，银行账户有交易，客户有银行账户，订单有行项目，目录也拥有文件和子目录。

在关系数据库中，处理关联关系的方式让你不会感到意外：

- 每个实体(或者行，在关系世界中)可以通过一个主键唯一标识
- 实体是规范化了的。对于一个唯一的实体，它的数据仅被存储一次，而与之关联的实体则仅仅保存它的主键。改变一个实体的数据只能发生在一个地方
- 在查询期间，实体可以被联接(Join)，它让跨实体查询成为可能
- 对于单个实体的修改是原子性，一致性，隔离性和持久性的(参考ACID事务获取更多相关信息。)
- 绝大多数的关系型数据库都支持针对多个实体的ACID事务

但是关系型数据库也有它们的局限，除了在全文搜索领域它们拙劣的表现外。在查询期间联接实体是昂贵的 - 联接的实体越多，那么查询的代价就越大。对不同硬件上的实体执行联接操作的代价太大以至于它甚至是不切实际的。这就为在单个服务器上能够存储的数据量设下了一个限制。

ES，像多数NoSQL数据库那样，将世界看作是平的。一个索引就是一系列独立文档的扁平集合。一个单一的文档应该包括用来判断它是否符合一个搜索请求的所有信息。

虽然在ES中改变一份文档的数据是符合[ACID事务](http://en.wikipedia.org/wiki/ACID_transactions)的，涉及到多份文档的事务就不然了。在ES中，当事务失败后是没有办法将索引回滚到它之前的状态的。

这个扁平化的世界有它的优势：

- 索引是迅速且不需要上锁的
- 搜索是迅速且不需要上锁的
- 大规模的数据可以被分布到多个节点上，因为每份文档之间是独立的

但是关联关系很重要。我们需要以某种方式将扁平化的世界和真实的世界连接起来。在ES中，有4中常用的技术来管理关联数据：

- [应用端联接(Application-side joins)](https://www.elastic.co/guide/en/elasticsearch/guide/current/application-joins.html)
- [数据非规范化(Data denormalization)](https://www.elastic.co/guide/en/elasticsearch/guide/current/denormalization.html)
- [嵌套对象(Nested objects)](https://www.elastic.co/guide/en/elasticsearch/guide/current/nested-objects.html)
- [父子关联关系(Parent/child relationships)](https://www.elastic.co/guide/en/elasticsearch/guide/current/parent-child.html)

通常最终的解决方案会结合这些方案的几种。

## 应用端联接(Application-side Joins)

我们可以通过在应用中实现联接来(部分)模拟一个关系型数据库。比如，当我们想要索引用户和他们的博客文章时。在关系型的世界中，我们可以这样做:

```
PUT /my_index/user/1  (1)
{
  "name":     "John Smith",
  "email":    "john@smith.com",
  "dob":      "1970/10/24"
}

PUT /my_index/blogpost/2   (2)
{
  "title":    "Relationships",
  "body":     "It's complicated...",
  "user":     1   (3)
}
```
> (1)(2) 索引，类型以及每份文档的ID一起构成了主键。
> (3) 博文通过保存了用户的ID来联接到用户。由于索引和类型是被硬编码到了应用中的，所以这里并不需要。

通过用户ID等于1来找到对应的博文很容易：

```
GET /my_index/blogpost/_search
{
  "query": {
    "filtered": {
      "filter": {
        "term": { "user": 1 }
      }
    }
  }
}
```

为了找到用户John的博文，我们可以执行两条查询：第一条查询用来得到所有名为John的用户的IDs，第二条查询通过这些IDs来得到对应文章:

```
GET /my_index/user/_search
{
  "query": {
    "match": {
      "name": "John"
    }
  }
}

GET /my_index/blogpost/_search
{
  "query": {
    "filtered": {
      "filter": {
        "terms": { "user": [1] }   (1)
      }
    }
  }
}
```
> (1) 传入到terms过滤器的值是第一条查询的结果。

应用端联接最大的优势在于数据是规范化了的。改变用户的名字只需要在一个地方操作：用户对应的文档。劣势在于你需要在搜索期间运行额外的查询来联接文档。

在这个例子中，只有一位用户匹配了第一条查询，但是在实际应用中可能轻易就得到了数以百万计的名为John的用户。将所有的IDs传入到第二个查询中会让该查询非常巨大，它需要执行百万计的term查询。

这种方法在第一个实体的文档数量较小并且它们很少改变时合适(这个例子中实体指的是用户)。这就使得通过缓存结果来避免频繁查询成为可能。

## 反规范化你的数据(Denormalizing Your Data)

让ES达到最好的搜索性能的方法是采用更直接的办法，通过在索引期间[反规范化](http://en.wikipedia.org/wiki/Denormalization)你的数据。通过在每份文档中包含冗余数据来避免联接。

如果我们需要通过作者的名字来搜索博文，可以在博文对应的文档中直接包含该作者的名字:

```
PUT /my_index/user/1
{
  "name":     "John Smith",
  "email":    "john@smith.com",
  "dob":      "1970/10/24"
}

PUT /my_index/blogpost/2
{
  "title":    "Relationships",
  "body":     "It's complicated...",
  "user":     {
    "id":       1,
    "name":     "John Smith" 
  }
}
```

现在，我们可以通过一条查询来得到用户名为John的博文了:

```
GET /my_index/blogpost/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title":     "relationships" }},
        { "match": { "user.name": "John"          }}
      ]
    }
  }
}
```

对数据的反规范化的优势在于速度。因为每份文档包含了用于判断是否匹配查询的所有数据，不需要执行代价高昂的联接操作。

## 字段折叠(Field Collapsing)

一个常见的需求是通过对某个特定的字段分组来展现搜索结果。我们或许希望通过对用户名分组来返回最相关的博文。对用户名分组意味着我们需要使用到terms聚合。为了对用户的全名进行分组，name字段需要有not_analyzed的原始值，如聚合和分析中解释的那样。

```
PUT /my_index/_mapping/blogpost
{
  "properties": {
    "user": {
      "properties": {
        "name": {   (1)
          "type": "string",
          "fields": {
            "raw": {   (2)
              "type":  "string",
              "index": "not_analyzed"
            }
          }
        }
      }
    }
  }
}
```
> (1) 字段用来支持全文搜索。 
> (2) 字段用来支持terms聚合来完成分组。

然后添加一些数据:

```
PUT /my_index/user/1
{
  "name": "John Smith",
  "email": "john@smith.com",
  "dob": "1970/10/24"
}

PUT /my_index/blogpost/2
{
  "title": "Relationships",
  "body": "It's complicated...",
  "user": {
    "id": 1,
    "name": "John Smith"
  }
}

PUT /my_index/user/3
{
  "name": "Alice John",
  "email": "alice@john.com",
  "dob": "1979/01/04"
}

PUT /my_index/blogpost/4
{
  "title": "Relationships are cool",
  "body": "It's not complicated at all...",
  "user": {
    "id": 3,
    "name": "Alice John"
  }
}
```

现在我们可以运行一个查询来获取关于relationships的博文，通过用户名为John对结果进行分组。这都要感谢top_hits聚合:

```
GET /my_index/blogpost/_search?search_type=count (1)
{
  "query": { (2)
    "bool": {
      "must": [
        { "match": { "title":     "relationships" }},
        { "match": { "user.name": "John"          }}
      ]
    }
  },
  "aggs": {
    "users": {
      "terms": {
        "field":   "user.name.raw",      (3) 
        "order": { "top_score": "desc" } (4)
      },
      "aggs": {
        "top_score": { "max":      { "script":  "_score"           }},  (5)
        "blogposts": { "top_hits": { "_source": "title", "size": 5 }}   (6)
      }
    }
  }
}
```
> (1) 我们感兴趣的博文在blogposts聚合中被返回了，因此我们可以通过设置search_type=count来禁用通常的搜索结果。
> (2) 该查询返回用户名为John，title匹配relationships的博文。
> (3) terms聚合为每个user.name.raw值创建一个桶。
> (4)(5) 在users聚合中，使用top_score聚合通过每个桶中拥有最高分值的文档进行排序。
> (6) top_hits聚合只返回每个用户的5篇最相关博文的title字段。

部分响应如下所示:

```
...
"hits": {
  "total":     2,
  "max_score": 0,
  "hits":      []    (1)
},
"aggregations": {
  "users": {
     "buckets": [
        {
           "key":       "John Smith",    (2)
           "doc_count": 1,
           "blogposts": {
              "hits": {    (3)
                 "total":     1,
                 "max_score": 0.35258877,
                 "hits": [
                    {
                       "_index": "my_index",
                       "_type":  "blogpost",
                       "_id":    "2",
                       "_score": 0.35258877,
                       "_source": {
                          "title": "Relationships"
                       }
                    }
                 ]
              }
           },
           "top_score": {    (4)
              "value": 0.3525887727737427
           }
        },
...
```
> (1) hits数组为空因为我们设置了search_type=count。
> (2) 针对每个用户都有一个对应的桶。
> (3) 在每个用户的桶中，有一个blogposts.hits数组，它包含了该用户的相关度最高的搜索结果。
> (4) 用户桶通过用户最相关的博文进行排序。

使用top_hits聚合等效于运行获取最相关博文及其对应用户的查询，然后针对每个用户运行同样的查询，来得到每个用户最相关的博文。可见使用top_hits更高效。

每个桶中返回的top hits是通过运行一个基于原始主查询的迷你查询而来。该迷你查询也同样支持高亮(Highlighting)以及分页(Pagination)。

## 反规范化和并发(Denormalization and Concurrency)

当然，数据反规范化也有弊端。首先，它会让索引变大，因为每篇博文的_source都变大了，与此同时需要索引的字段也变多了。通常这并不是一个大问题。写入到磁盘的数据会被高度压缩，而且磁盘存储空间也不贵。ES能够很从容地处理这些多出来的数据。

更重要的弊端在于，如果用户修改了他的名字，他名下的所有博文都需要被更新。即使用户真的这么做了，一位用户也不太可能写了上千篇博文，因此通过scroll和bulk APIs，更新也不会超过1秒。

然而，让我们来考虑一个变化更常见，影响更深远和重要的复杂情景 - 并发。

在这个例子中，我们通过ES来模拟一个拥有目录树的文件系统，就像Linux上的文件系统那样：根目录是/，每个目录都能包含文件和子目录。

我们希望能够搜索某个目录下的文件，就像下面这样:

```
grep "some text" /clinton/projects/elasticsearch/*
```

它需要我们对文件的路径进行索引:

```
PUT /fs/file/1
{
  "name":     "README.txt", 
  "path":     "/clinton/projects/elasticsearch", 
  "contents": "Starting a new Elasticsearch project is easy..."
}
```

> **NOTE**
> 说实在的，我们同时也应该一个目录下所有的文件和子目录进行索引，但是为了简洁起见，这里忽略了这一需求。

我们还需要能够搜索某个目录下任意深度的文件，就像下面这样:

```
grep -r "some text" /clinton
```

为了支持这一需求，需要对路径层次进行索引:

- /clinton
- /clinton/projects
- /clinton/projects/elasticsearch

该层次结构可以通过对path字段使用path_hierarchy tokenizer来自动生成:

```
PUT /fs
{
  "settings": {
    "analysis": {
      "analyzer": {
        "paths": {   (1)
          "tokenizer": "path_hierarchy"
        }
      }
    }
  }
}
```
> (1) 以上的自定义的分析器使用了path_hierarchy分词器，使用其默认设置

文件类型的映射则像下面这样:

```
PUT /fs/_mapping/file
{
  "properties": {
    "name": {   (1)
      "type":  "string",
      "index": "not_analyzed"
    },
    "path": {   (2)
      "type":  "string",
      "index": "not_analyzed",
      "fields": {
        "tree": {   (3)
          "type":     "string",
          "analyzer": "paths"
        }
      }
    }
  }
}
```
> (1) name字段会包含完整的名字。
> (2)(3) path字段会包含完整的目录名，而path.tree字段会包含路径层次结构。

一旦建立了该索引并完成了文件的索引，我们就能够执行如下查询，它搜索/client/projects/elasticsearch目录下包含有elasticsearch的文件:

```
GET /fs/file/_search
{
  "query": {
    "filtered": {
      "query": {
        "match": {
          "contents": "elasticsearch"
        }
      },
      "filter": {
        "term": {   (1)
          "path": "/clinton/projects/elasticsearch"
        }
      }
    }
  }
}
```
> (1) 只在该目录下搜索。

任何存储于/clinton目录下的文件都会在path.tree字段中包含/clinton。因此我们可以通过下面的搜索来得到/clinton目录下的所有文件:

```
GET /fs/file/_search
{
  "query": {
    "filtered": {
      "query": {
        "match": {
          "contents": "elasticsearch"
        }
      },
      "filter": {
        "term": {   (1)
          "path.tree": "/clinton"
        }
      }
    }
  }
}
```
> (1) 寻找该目录或其下任意子目录中的文件。

### 重命名文件和目录(Renaming Files and Directories)

到目前为止还不错。文件重命挺简单 - 只需要一个简单的更新或者索引请求就行了。你甚至可以使用[乐观并发控制(Optimistic Concurrency Control)](https://www.elastic.co/guide/en/elasticsearch/guide/current/optimistic-concurrency-control.html)来确保你的更改不会和另一个用户的更改发生冲突。

```
PUT /fs/file/1?version=2    (1)
{
  "name":     "README.asciidoc",
  "path":     "/clinton/projects/elasticsearch",
  "contents": "Starting a new Elasticsearch project is easy..."
}
```
> (1) version值能够确保只有当索引中的文档拥有相同的version值时，更改才会生效。

我们还可以对目录重命名，但是这意味着对该目录下的所有文件执行更新。这个操作或快或慢，取决于有多少文件需要被更新。我们需要做的是使用scan-and-scroll来获取所有的文件，然后使用bulk API完成更新。这个过程并不是原子性的，但是所有的文件都能够被迅速地更新。

## 解决并发问题(Solving Concurrency Issues)

当我们允许多个用户同时对文件和目录进行重命名时，问题就来了。假设你对/clinton目录进行重命名，它包含了成百上千个文件。同时，另外一个用户重命名了/clinton/projects/elasticsearch/README.txt这个文件。该用户的更改虽然在你的操作之后，但是它完成的也许更快。

下面两种情况中的一种会发生:

- 你决定使用version值，这意味着你对文件README.txt的重命名操作会因为version冲突而失败。
- 你不使用versioning，那么你的更改会直接覆盖掉另一用户的更改。

问题的根源在ES并不支持ACID事务。对单个文档的更新虽然是符合ACID原则的，但是对多个文档的更新则不然。

如果你使用的主要数据源是关系行数据库，而ES只是简单地被用来当作搜索引擎或者提升性能的方法，那么首先更新数据库，然后待这些更新操作成功后再将这些变更复制到ES中。这样的话，你就能受益于数据库对于ACID事务的支持了，它能保证ES中的所有更新顺序都是正确的。并发的问题在关系型数据库中被处理了。

如果你没有使用关系型数据源，那么这些并发问题就需要在ES中完成。下面有三种可行的方案，这些方案都涉及到了某种形式的锁:

- 全局锁(Global Locking)
- 文档锁(Document Locking)
- 树锁(Tree Locking)

> **TIP**
> 以上提到的方案都可以通过应用了相同原则的外部系统实现。

### 全局锁(Global Locking)

我们可以通过在任何时候只允许一个进程执行更新操作来完全避免并发性的问题。大多数的修改只设计很少的几个文件，完成的也相当快。对于顶层目录的重命名也许会阻塞其它变更操作更久一点，但是这种情况发生的频率也会更低一些。

因为在ES中文档级别的变更是满足ACID的，我们可以将一份文档是否存在作为一个全局锁。为了获取一个锁，我们尝试去创建一个全局锁文档:

```
PUT /fs/lock/global/_create
{}
```

如果上述的创建请求由于发生了冲突而失败了，就表明另一个进程已经被授权得到了全局锁，我们只能稍后重试。如果上述请求成功了，我们就拥有了全局锁从而可以进行后续的变更操作。一旦这些操作完成了，必须通过删除全局锁文档来释放它:

```
DELETE /fs/lock/global
```

取决于变更的频繁程度和它们需要消耗的时间，全局锁对系统的性能或许会有相当程度的限制。可以通过实现更加细粒度的锁来增加并行性。

### 文档锁(Document Locking)

相比于对整个文件系统上锁，我们可以通过上面提到的技术来对单个文档完成锁定。一个进程可以使用[scan-and-scroll](https://www.elastic.co/guide/en/elasticsearch/guide/current/scan-scroll.html)请求来获取到变更会影响到的所有文档的IDs，然后对每份文档创建一个锁文件:

```
PUT /fs/lock/_bulk
{ "create": { "_id": 1}}   (1)
{ "process_id": 123    }   (2)
{ "create": { "_id": 2}}
{ "process_id": 123    }
...
```
> (1) 锁文档的ID需要和被锁定的文档的ID一致。
> (2) process_id是将要执行变更操作进程的唯一ID。

如果某些文档已经被锁了，那么部分bulk请求会失败，只好重试。

当然，如果我们试图再次去锁定所有的文档，对于那些已经被我们锁定的文档，前面使用的创建语句会失败！相比一个简单的创建语句，我们需要使用带有upsert参数的update请求:

```
if ( ctx._source.process_id != process_id ) {   (1)
  assert false;   (2)
}
ctx.op = 'noop';  (3)
```
> (1) process_id是我们传入到脚本中的参数。
> (2) assert false会抛出一个异常，它导致更新失败。
> (3) 将op从update修改为noop能够防止update请求真的执行变更操作，而仍然返回成功。

完整的update请求如下所示:

```
POST /fs/lock/1/_update
{
  "upsert": { "process_id": 123 },
  "script": "if ( ctx._source.process_id != process_id )
  { assert false }; ctx.op = 'noop';"
  "params": {
    "process_id": 123
  }
}
```
如果文档并不存在，upsert会执行insert操作 - 就和之前使用的create请求一样。但是，如果文档存在，脚本会查看文档中保存的process_id。如果它和我们的相同，就会放弃update(noop)并返回成功。如果它和我们的不同，那么assert false就会抛出一个异常告诉我们锁定失败了。

一旦所有的锁都别成功创建了，重命名操作就开始了。在这之后，我们必须释放所有的锁，通过delete-by-query请求来完成:

```
POST /fs/_refresh   (1)

DELETE /fs/lock/_query
{
  "query": {
    "term": {
      "process_id": 123
    }
  }
}
```
> (1) refresh调用保证了所有的锁文档对delete-by-query请求是可见的。

文档级别的锁拥有更加细粒度的访问控制，但是为百万计的文档创建锁是非常昂贵的。在某些场合下，比如前面例子中出现的目录树，可以通过更少的工作来达到细粒度的锁定。

### 树锁(Tree Locking)

相比像前面那样对每份文档上锁，也可以支队目录树的部分上锁。我们需要对重命名的文档或者目录拥有独占性访问，可以通过独占性锁文档实现:

```
{ "lock_type": "exclusive" }
```

同时我们也需要对上层目录使用分享锁:

```
{
  "lock_type":  "shared",
  "lock_count": 1   (1)
}
```
> (1) lock_count记录了拥有该分享锁的进程数量。

对/clinton/projects/elasticsearch/README.txt重命名的进程需要该文件的独占锁，以及针对目录/clinton，/clinton/projects和/clinton/projects/elasticsearch的分享锁。

对于独占锁，可以通过简单的创建请求来实现，但是分享锁需要带有脚本的update请求来实现:

```
if (ctx._source.lock_type == 'exclusive') {
  assert false;   (1)
}
ctx._source.lock_count++   (2)
```
> (1) 如果lock_type是独占性的，那么assert语句会抛出一个异常，导致update请求失败。
> (2) 否则，增加lock_count。

该脚本能够处理当锁文档已经存在的情况，但是我们仍然需要使用upsert来处理它不存在时的情况。完整的update请求如下所示:

```
POST /fs/lock/%2Fclinton/_update   (1)
{
  "upsert": {   (2)
    "lock_type":  "shared",
    "lock_count": 1
  },
  "script": "if (ctx._source.lock_type == 'exclusive')
  { assert false }; ctx._source.lock_count++"
}
```
> (1) 文档的ID是/clinton，再进行URL编码后变成了%2fclinton。
> (2) 当文档不存在时，upsert代表的文档会被插入。

一旦我们成功获取了文件所在目录的所有上级目录的分享锁后，就可以尝试去创建一个针对该文件的独占锁了:

```
PUT /fs/lock/%2Fclinton%2fprojects%2felasticsearch%2fREADME.txt/_create
{ "lock_type": "exclusive" }
```

现在，如果另外的某个人想要对/clinton目录重命名，他们就需要获取该目录的独占锁:

```
PUT /fs/lock/%2Fclinton/_create
{ "lock_type": "exclusive" }
```

该请求会失败，因为一个拥有相同ID的锁文档已经存在了。该用户只好等待我们的操作完成并释放锁。独占锁可以被删除:

```
if (--ctx._source.lock_count == 0) {
  ctx.op = 'delete'   (1)
}
```
> (1) 一旦lock_count为0，ctx.op就从update变成delete。

对每个上级目录都需要以相反的顺序执行update请求，从最长的目录到最短的目录:

```
POST /fs/lock/%2Fclinton%2fprojects%2felasticsearch/_update
{
  "script": "if (--ctx._source.lock_count == 0) { ctx.op = 'delete' } "
}
```

树锁通过最少的代价实现了细粒度的并发控制。当然，它并不能适用于每个场景 - 数据模型必须要有类似目录树这种结构才行。

> **NOTE**
> 以上的三种方案 - 全局锁，文档锁和树锁 - 都没有处理关于锁的最棘手的问题：如果持有锁的进程挂了怎么办？
> 一个进程的意外挂掉给我们留下了两个问题：
> - 我们如何知道我们可以释放被挂掉的进程持有的锁？
> - 我们如何清理挂掉的进程没有完成的工作？

尽管反规范化对于很多项目而言都是一个好的选择，由于需要锁的支持，会导致较为复杂的实现。相比之下，ES提供了另外两种模型来处理关联的实体：[嵌套对象(Nested Objects)](https://elasticsearch.cn/book/elasticsearch_definitive_guide_2.x/nested-objects.html)和[父子关系(Parent-Child Relationship)](https://elasticsearch.cn/book/elasticsearch_definitive_guide_2.x/parent-child.html)。
