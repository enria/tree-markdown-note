# 数据中心Elasticsearch使用指南

我们主要会介绍一些常用的DSL语句，整个文档会作为一个模板来提供参考。如果有时间的还是建议查看官方的文档。

在下面的DSL中，我们会使用占位符`${...}`来指代需要修改的地方。

## 索引和映射

### 新建索引

```http
PUT ${index}
{
  "settings": {
    "number_of_shards": ${shards_num},
    "number_of_replicas": ${replicas_num}
  },
  "aliases": {
    "${aliases_name}": {}
  },
  "mappings": {
    "${type}": {
      "dynamic": ${dynamic},
      "properties": {
        "${field_keyword}": {
          "type": "keyword"
        },
        "${field_text}": {
          "type": "text",
          "fields": {
            "${extra_filed_type}": {
              "type": "keyword",
              "ignore_above": 256
            }
          },
          "analyzer": "${analyzer}"
        },
        "${field_date}": {
          "type": "date",
          "format": "${date_format}"
        },
        "${field_number}": {
          "type": "integer"
        },
        "${field_nested}": {
          "type": "nested",
          "properties": {
            "${nested_field}": {
              "type": "keyword"
            }
          }
        }
      }
    }
  }
}
```
+ 索引
    + `${index}`:索引名（数据库名）
+ 分片
    + `${shards_num}`：主分片数，代表整个库会分成几个独立的存储单元。
    + `${replicas_num}`：副本数，代表数据会复制几份。
+ 别名
    + `${aliases_name}`：别名，在重建索引的时候会详细介绍它的作用。
+ 映射
    + `${type}`:类型名（表名）
    + `${dynamic}`:是否自动映射。有3种选择：`true`，`false`，`"strict"`。`true`代表使用自动映射，意思是在创建新文档时存在没有映射字段，ES会自动给它一种类型。`false`跟`"strict"`都是不自动映射，但是机制不同，`false`会忽略这个字段，而`"strict"`会直接放弃这个文档。
    + `${field_keyword}`:关键字字段，文本，不会被分词。
    + `${field_text}`:文本字段，会被分词。
    + `${analyzer}`:分词器，作为文本字段的一个属性。如果不传会使用标准分词器（`standard`）,这个分词器对于英文使用空格、标点等符号进行分词，对于中文也会使用空格、标点切分，每个汉字作为一个词进行索引。后面会介绍其它的分词器。
    + `${extra_filed_type}`:额外类型。一个字段可以使有多种索引方式来支持各种查询，在使用的时候以`${filed_name}`.`${extra_filed_type}`来声明使用哪种索引。如果不指定的话，就是使用最外层的索引方式。
    + `${filed_date}`:日期字段，通过`${date_format}`指明日期的格式。
    + `${field_number}`:数字型字段，这里只写了`integer`，但其实支持的类型特别多，具体可以看[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/5.3/number.html)。
    + `${field_nested}`:嵌套字段，可以把它理解为是一个列表，定义映射的时候其实定义的是列表中存储的对象的映射，`${nested_field}`是列表中对象的字段名。

### 修改映射

```http
GET ${index}/${type}/_mapping
{
  "${type}":{
    "properties":{
      "${add_filed}":{
        "type": "${type}"
      }
    }
  }
}
```

因为ES的映射字段一旦创建就不能修改，所以我们说的修改映射只是添加字段。声明要修改的索引`${index}`和类型`${type}`，要加的字段`${add_filed}`和字段的类型`${type}`即可。

### 重建索引

```http
POST _reindex
{
  "source": {
    "index": "${source_index}"
  },
  "dest": {
    "index": "${desc_index}"
  }
}
```

前面说ES的映射字段不可能修改，但有时候可能因为需求的变化等等因素——如关键字字段要支持模糊查询——，我们确实需要修改映射，这时候就要重建索引了。另外，主分片数也一旦确定也无法修改，随着数据的增长，太少的副本数可能会成为性能的瓶颈，为了提高索引和搜索的性能，也可能会重建索引。

重建索引有两个步骤：首先创建新的索引，然后使用重建索引的API进行数据迁移，在DSL需要指定源索引`${source_index}`和目标索引`${desc_index}`。

很明显源索引和目标索引名称不能一样，如果我们使用了新的索引名，那对于ES的使用者来说，岂不是每重建一次索引就要在业务代码中一次更改索引名称？情况并不是这样，这时候索引的别名就开始发挥它的作用。我们在创建索引的时候，索引名称可以带上额外的信息，可以是索引的版本，比如`my_index_v1`，再给索引一个没有额外信息的别名，这个例子中就是`my_index`。在使用的时候，我们只使用别名来操作索引。重建索引时新索引名`my_index_v2`，为当我们重建完索引，删除旧索引的别名`my_index`——也可直接删除旧的索引`my_index_v1`——，再给新的索引`my_index_v2`添加别名`my_index`。这样就可以完成重建索引的过程。这样的操作方式，对于业务系统来说，数据一直都是可用的（除了删除旧索引的别名到给新索引添加别名之间极短暂的时间），而且不需要任何修改。

### 添加别名

```http
PUT ${index}/_alias/${aliases}
```

给指定的索引`${index}`添加别名`${aliases}`，这一步也可以直接在创建索引的时候进行。

### 删除别名

```http
DELETE ${index}/_alias/${aliases}
```

删除索引`${index}`的别名`${aliases}`。

## 分词器

分词器作为插件放置在ES服务器内部。

### 测试分词器

```http
GET _analyze
{
  "tokenizer" : "${analyzer}",
  "text" : "${test_text}"
}
```

指定使用的分词器`${analyzer}`和测试的文本`${test_text}`，可以得到对于测试文本的分词结果。

### IK分词器

[IK分词器](https://github.com/medcl/elasticsearch-analysis-ik)是一个中文的分词器，目前数据中心使用它来过行中文分词。

IK分词器提供了两种分词方式：`ik_smart`和`ik_max_word`。我们使用测试接口来直观的感受它们的区别。

#### ik_smart

```http
GET _analyze
{
  "tokenizer" : "ik_smart",
  "text" : "中华人民共和国国歌"
}
```

结果:

```json
{
  "tokens": [
    {
      "token": "中华人民共和国",
      "start_offset": 0,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 0
    },
    {
      "token": "国歌",
      "start_offset": 7,
      "end_offset": 9,
      "type": "CN_WORD",
      "position": 1
    }
  ]
}
```

#### ik_max_word

```http
GET _analyze
{
  "tokenizer" : "ik_max_word",
  "text" : "中华人民共和国国歌"
}
```

结果：

```json
{
  "tokens": [
    {
      "token": "中华人民共和国",
      "start_offset": 0,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 0
    },
    {
      "token": "中华人民",
      "start_offset": 0,
      "end_offset": 4,
      "type": "CN_WORD",
      "position": 1
    },
    {
      "token": "中华",
      "start_offset": 0,
      "end_offset": 2,
      "type": "CN_WORD",
      "position": 2
    },
    {
      "token": "华人",
      "start_offset": 1,
      "end_offset": 3,
      "type": "CN_WORD",
      "position": 3
    },
    {
      "token": "人民共和国",
      "start_offset": 2,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 4
    },
    {
      "token": "人民",
      "start_offset": 2,
      "end_offset": 4,
      "type": "CN_WORD",
      "position": 5
    },
    {
      "token": "共和国",
      "start_offset": 4,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 6
    },
    {
      "token": "共和",
      "start_offset": 4,
      "end_offset": 6,
      "type": "CN_WORD",
      "position": 7
    },
    {
      "token": "国",
      "start_offset": 6,
      "end_offset": 7,
      "type": "CN_CHAR",
      "position": 8
    },
    {
      "token": "国歌",
      "start_offset": 7,
      "end_offset": 9,
      "type": "CN_WORD",
      "position": 9
    }
  ]
}
```

对比两个分词器我们可以发现，`ik_max_word`会将文本做最细粒度的拆分，`ik_smart`会将文本做最粗粒度的拆分。二者都各有优缺点，比如`ik_max_word`对于测试文本“中华人民共和国国歌”共分出了10个词，而`ik_smart`只分出了2个词，我们可以认为对于这个测试文本，`ik_max_word`存储需要的空间是`ik_smart`的5倍。但是因为`ik_smart`分出的词只有2个，对于一些词，如“共和国”没有索引，所以如果使用`ik_smart`进行分词，存在“中华人民共和国国歌”短语的文档将无法命中关词词“共和国”的查询。

上面的这个例子只是说`ik_max_word`占用空间大，`ik_smart`召回率低（文档中存在短语，但无法命中）这些缺点，但在实际的使用过程中，情况要比这个复杂。上面的分词结果中有个`position`字段，代表分词的位置，后面的查询部分会用到它。

## 搜索

搜索的基本模板是：

```http
GET ${index}/${type}/_search
{
  "query": ${query_body},
  "sort": ${sort_array},
  "from": ${from_index},
  "size": ${page_size},
  "_source": {
    "includes": ${fetch_fileds_array}
  }
}
```

+ query放置查询的条件，我们称它为检索
+ sort对数据进行排序
+ 使用from和size来控制分页
+ _source指定我们抽取的字段，类似于SQL中的`select ${fetch_fileds_array} from table`

### 检索

ES提供了丰富的检索接口来让我们从海量的文档中拿到自己想要的数据，我把它们分成3类：全文检索、精准检索、复合检索。检索的内容放在`${query_body}`里面。

#### 全文检索

分词查询是指对分词字段（即`text`）的查询，实现原理主要是将查询的文本分词，与倒排索引表进行比较并计算匹配度，匹配度达到一个阈值后将命中文档。

下面介绍几种常用的全文查询。

##### match

```json
{
  "match": {
    "${text_filed}": {
      "query": "${text}",
      "analyzer": "${analyzer}",
      "minimum_should_match": "${minimum_match_percent}"
    }
  }
}
```
指定查询的文本`${text}`进行查询。另外还有一些高级的参数，我们在使用ES的过程中可能会用到：分词器`${analyzer}`和最小匹配度`${minimum_match_percent}`。

我们前面说到，全文检索在查询前会将查询的文本进行分词，分词就需要分词器。如果我们不指定分词器，默认会使用查询字段在创建索引时的分词器。

此外还有一个匹配度的概念。我们在使用互联网搜索引擎的时候会发现，当查询的文本比较小众或者文本比较长时，搜索引擎很少会直接响应0条命中，很多时候是给你一些只命中查询文本中部分关键词的文档。对于ES来说，`match`的策略跟互联网搜索引擎比较相似。在默认情况下，`match`认为即使命中一个关键词，这个文档也可以作为命中的，如果我们的需求希望把这个阈值提高，就需要使用最小匹配度这个参数。它的计算规则也值得注意，算法是`floor(len(tokens)*minimum_match_percent)`，假设查询的文本分词后的关键词数量是`7`,`minimum_match_percent`为`60%`，那根据这个算法那就是只要4个关键词命中就认为是命中的文档。

##### match_phrase

```json
{
  "match_phrase": {
    "${text_filed}": {
      "query": "${text}",
      "slop": ${slop_num}
    }
  }
}

```

上面说到`match`查询判定依据是分词后查询命中的词的数量是否达到阈值，但是这种查询有一个缺点是没有考虑分词后的词之间的位置关系。比如查询“中华人民”，分词后是`["中华","人民"]`，极端情况下“人民”在文章的开头，“中华”在文章的结尾，这样也是会判定命中，但是我们可能并不需要这样的文档。

针对这个问题我们可以使用`match_phrase`查询。它要求分词后的关键词与文档索引时的关键具有相同的position偏移量。举个例子，“中华人民共和国”分词后是

```json
{
  "中华": {
    "position": 0
  },
  "人民": {
    "position": 1
  },
  "共和国": {
    "position": 2
  }
}
```

我们假设倒排索引表中的数据是

```json
{
  "中华": [
    {
      "id": "doc_1",
      "position": 1
    },
    {
      "id": "doc_2",
      "position": 5
    }
  ],
  "人民": [
    {
      "id": "doc_1",
      "position": 2
    },
    {
      "id": "doc_2",
      "position": 6
    }
  ],
  "共和国": [
    {
      "id": "doc_1",
      "position": 3
    },
    {
      "id": "doc_2",
      "position": 10
    }
  ],
  ...
}

```

上面索引表的`position`为`{ "doc_1" : [ 1 , 2 , 3 ] , "doc_2" : [ 5 , 6 , 10 ] }`,而查询文本分词后是`[0,1,2]`，对于doc_1的每个词，索引表中的`position`与查询文本分词的`position`相差是`[ 1 , 1 , 1]`，所以判定是命中，但对于doc_2，`position`偏移量为`[ 5 , 5 , 8 ]`,所是判定为未命中。

很多时候因为分词的原因，特别是像`ik_max_word`这种细粒度的分词策略，上面这种严格的匹配模式会使很多我们需要的文档无法命中。幸运的是`match_phrase`提供给我们`slop`参数可以把这个门槛降低，这个参数默认是`0`，上面的例子`[ 1 , 1 , 1 ]`之间的差距就是`0`，如果我们把这个值提高到`3`，那doc_2的`[ 5 , 5 , 8 ]`符合这个条件，也就会判定为命中。

##### match_phrase_prefix

```json
{
  "match_phrase": {
    "${text_filed}": "${text}"
  }
}

```

`match_phrase_prefix`在机制上与`match_phrase`很相似，不同点在于查询文本分词后的最后一个关键词不需要与文档分词后的关键词完全相同，只需要是它的前缀就可以。例如查询的文本分词后是：

```json
{
  "中华": {
    "position": 0
  },
  "人民": {
    "position": 1
  },
  "共和": {
    "position": 2
  }
}

```

doc_1的索引表是：

```json
{
  "中华": [
    {
      "id": "doc_1",
      "position": 1
    }
  ],
  "人民": [
    {
      "id": "doc_1",
      "position": 2
    }
  ],
  "共和国": [
    {
      "id": "doc_1",
      "position": 3
    }
  ],
  ...  
}

```

使用`match_phrase`无法命中doc_1，但是`match_phrase_prefix`可以，因为“共和”是“共和国”的前缀。

#### 精准匹配

精准匹配与全文检索相比简单很多，如果具有关系型数据库方面的知识，理解起来会非常容易。

##### term

```json
{
  "term": {
    "${field}": {
      "value": "${value}"
    }
  }
}
```

基本上对于所有的字段来说都可以使用`term`查询，甚至分词的文本字段也可以。如果是非分词字段（数字、日期、关键字等）就是简单的相等，而分词字段的查询方式是不将查询文本分词，只要文本在该字段的索引表中存在，就会判定为命中。

##### range

```json
{
  "range": {
    "${field}": {
      "${op}": "${value}"
    }
  }
}
```

对字段进行范围查询，被查询的字段要求是数字型或者是日期型。`${op}`代表比较符，包括`gt`、`gte`、`lt"、"lte`，分别对应>、>=、<、<=。要注意对于日期型传入的日期要与创建`mapping`时相同，例如对于格式为`yyyy-MM-dd HH:mm:ss`格式，我们查询大于某一天的，也要使用下面的格式。

```json
{
  "range": {
    "update_time": {
      "gt": "2018-01-01 00:00:00"
    }
  }
}
```

##### regexp

```json
{
  "regexp": {
    "${field}": "${regexp}"
  }
}
```

使用正则匹配。

##### prefix

```json
{
  "prefix": {
    "${field}": {
      "value": "${value}"
    }
  }
}
```

使用前缀查询。

#### 脚本查询

```json
{
  "script": {
    "script": "${script}"
  }
}
```

在一些极端的情况下，ES提供的查询语句无法实现我们需要的查询效果，那就可以尝试使用脚本查询。脚本的语法默认使用`painless`，它与Java具有很高的相似度。

例如我们使用脚本查询一个字段的长度是否符合要求，查询DSL就是下面这样。

```json
{
  "query": {
    "script": {
      "script": "doc['goods_url_md5'].value.length()!=32"
    }
  }
}
```

因为脚本比较复杂，并且使用的场景不多，所以我们就一笔带过，想要了解更多，可以查看[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/5.3/search-request-script-fields.html)。

#### 逻辑查询

```json
{
  "bool": {
    "must": ${must_array},
    "must_not": ${must_array},
    "should": ${must_array},
    "minimum_should_match": ${minimum_number},
      "filter": ${filter_body}
  }
}
```

ES的逻辑查询与关系型数据库相比较为复杂，但是熟悉之后会发现它的功能很强大，并且具有很强的抽象能力。

查询体主要分为3个列表：`must`、`must_not`、`should`，每个列表中的元素是一个查询，可以是`match`、`term`等简单的查询，也可以是一个逻辑查询，例如

```json
{
  "bool": {
    "must": [
      {
        "match": {
          "text": {
            "value": "中国"
          }
        }
      }
    ],

    "must_not": [
      {
        "range": {
          "update_time": {
            "gte": "2018-01-01 00:00:00"
          }
        }
      }
    ],
    "should": [
      {
        "bool": {
          "must": [
            {
              "prefix": {
                "title": {
                  "value": "庆祝"
                }
              }
            }
          ]
        }
      }
    ]
  }
}
```

下面说一下每个列表的含义。

+ `must`代表列表中的每个条件都要被满足
+ `must_not`代表列表中的任何条件都不能被满足
+ `should`代表列表中的条件至少需要满足一定的个数，这个数量可以通过`minimum_should_match`这个参数传入。

此外，逻辑查询还支持通过`filter`过滤文档，过滤文档与查询文档不同的是，过滤的文档不会参与分值的计算，不符合条件的文档直接放弃，而查询文档会进行分值计算来排序，使命中度高的文档排在前面。同时，符合过滤条件的文档会被缓存，提高的后面的查询速度。这也是比较符合一般的使用场景，比如说针对一个地区的数据我们可能会查询多次，每次查询的文本不同，但都会限定到这个地区。一般过滤后的文档会比整个索引小很多，通过缓存可以使查询速度提高很多。

### 排序

默认情况数据的排序是根据文档的匹配度来进行的，匹配度是一个分值，值越大说明越匹配。

ES的匹配度分值算法是使用Lucene的TF-IDF算法，简化一下该模型可以得出以下结论：

+ 命中的次数越多，分值越高
+ 文本越短，分值越高
+ 关键词越稀有（出现在所有文档中的次数比较少），分值越高。

最后一点可能初看会有些奇怪：对于单次查询来说关键词是固定的，关键词的稀有度就是固定的，怎么会对分值有影响呢？其实原因上面也有说到，`match`查询会有一个匹配度，也就是说查询出来的文档不一定是全部命中关键词的，比如查询的文本是“中华人民”，doc_1只命中关键词“中华”，doc_2只命中“人民”，假设“中华”比“人民”稀有，单看这方面doc_1的分值就会更高。

除了默认的匹配度排序，ES还提供了自定义排序，使用的方式如下：

```json
{
  "sort": [
    {
      "${field}": {
        "order": "${order}"
      }
    }
  ]
}
```

`order`有两种方式，`asc`、`desc`，分别代表正序和反序。传入的字段如果是数值型或者日期型是比较好理解的，如果传入的是关键字类型，排序的依据是字典序。另外要注意的是不可以传入被分词的字段，因为分词后的该字段的存储会变成倒排序引，丢失了完整的数据，所以无法进行排序，我们上面有说一个字段可以使用多种存储方式，如果确定一个分词的字段需要排序，那就增加一个关键词类型的额外字段，排序的时候使用该字段就可以实现排序。

此外排序可支持传入脚本，例如：

```json
{
  "sort": {
    "_script": {
      "type": "number",
      "script": {
        "lang": "painless",
        "inline": "doc['company_name.keyword'].size()==0?0:doc['company_name.keyword'].get(0).length()"
      },
      "order": "asc"
    }
  }
}
```

上面的脚本使字数较多的数据排在前面。当然，脚本的方式都是解决临时问题，它本身违反倒排索引的初衷，效率很低且耗费资源。如果长时间有这种需求，还是要冗余一个字段来进行排序比较好。

### 聚合

聚合的模板如下

```json
{
  "aggs":{
    "${agg_name}":{
      "${agg_type}":${agg_body}
    }
  }
}
```

ES的聚合接口相当丰富，我们只挑一些常用的聚合方式来说一下。

#### terms

```json
{
  "aggs": {
    "${agg_name}": {
      "terms": {
        "field": "${filed}",
        "size": ${fetch_size}
      }
    }
  }
}
```

这种聚合方式跟关系型数据库的聚合比较类似，结果是每个组的数量。

#### filter

```json
{
  "aggs": {
    "${agg_name}": {
      "filter": ${query_body}
    }
  }
}
```

指定查询条件，得到查询的数量。查询是在`query`（检索）的基础上增加额外的条件。如果我们只是简单的希望获取某个条件下的数据量，使用`query`并指定`size`为`0`，这样响应的结果里面`hits.total`就是命中的文档数量。使用聚合的方式可以让我们通过一次查询得到多个条件下的数据量，因为在`aggs`里面可以传入多个聚合，比如：

```json
{
  "aggs": {
    "${agg_1_name}": {
      "filter": ${query_body}
    },
    "${agg_2_name}": {
      "filter": ${query_body}
    }
  }
}
```

#### date_histogram

```json
{
  "aggs": {
    "${agg_name}": {
      "date_histogram": {
        "field": "${data_field}",
        "interval": "${interval}"
      }
    }
  }
}
```

日期直方图，当有类似“数据每天的增量”这种需求时就可以使用这种聚合方法。可以通过`${interval}`传入间隔，如`second`、`minute`、`day`、`month`等。

## 节点配置

### 系统配置

#### /etc/security/limits.conf

```shell
* soft nofile 65536
* hard nofile 65536

${user_name} soft memlock unlimited
${user_name} hard memlock unlimited
```

ES的启动时会进行检查，最大文件句柄数和内存锁，防止服务运行中长时间执行GC和内存交换导致响应太久。如果没有配置，ES无法启动。

### ES配置

```yam
thread_pool:
    bulk:
        queue_size: 800
```

对于数据中心当前的业务来说，ES的使用是写多读少的，在写入数据大部分是使用`bulk`（批量）方法，有时候查看错误日志会发现因为请求超过节点的线程队列大小，请求会被拒绝。但是这些错误都是间断性的，一般情况下写入速度不存在瓶颈。这种情况下通过增加线程队列大小就可以解决这个问题。要注意这个配置是针对单个节点的，需要修改该节点的配置并重启这个节点。

如果发现大部分请求都被拒绝，使用这个方法无法解决问题。应该要尝试放缓请求速度或者增加系统写入性能（增加节点、使用SSD）。



对于配置上面只说了一些平时碰到的问题，ES官方针对重要的配置有进行详细的说明，可以查看相应的[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/5.3/important-settings.html)。

## 集群管理

### 各节点硬盘占用

```http
GET _cat/allocation
```

### 各节点状态

```http
GET _nodes/stats
```

### 集群健康

```http
GET _cluster/health?level=indices
```

## 参考文档

首先是官方文档，最为详细:

[Elasticsearch Reference [5.3] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/5.3/index.html)

```shell
https://www.elastic.co/guide/en/elasticsearch/reference/5.3/index.html
```

一些博客也很有实践意义：

[eBay 的 Elasticsearch 性能调优实践](http://www.infoq.com/cn/articles/elasticsearch-performance-tuning-practice-at-ebay)

```shell
http://www.infoq.com/cn/articles/elasticsearch-performance-tuning-practice-at-ebay
```

[Elasticsearch5.0 安装问题集锦](https://www.cnblogs.com/sloveling/p/elasticsearch.html)

```shell
https://www.cnblogs.com/sloveling/p/elasticsearch.html
```

[全文搜索引擎 Elasticsearch 入门教程](http://www.ruanyifeng.com/blog/2017/08/elasticsearch.html)

```shell
http://www.ruanyifeng.com/blog/2017/08/elasticsearch.html
```