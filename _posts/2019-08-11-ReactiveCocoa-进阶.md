---
layout:     post
title:      ES聚合
subtitle:   ES聚合
date:       2019-08-11
author:     BY
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - ELK
---
[TOC]

## 组合查询

设置mapping productID不分词

```
put my_store
{
  "mappings": {
    "properties": {
      "productID": {
        "type": "keyword"
      }
    }
  }
}
```



```
my_store/_search
{
    "query" : {
        "bool" : {
            "filter" : {
                "range" : {
                    "price" : { "gt" : 10 }
                }
            },
            "must" : {
                "match" : {
                    "productID" : "KDKE-B-9947-#kL5"
                }
            }
        }
    }
}
```

## 聚合操作

所以让我们先看一个例子。我们将会创建一些对汽车经销商有用的聚合，数据是关于汽车交易的信息：车型、制造商、售价、何时被出售等。

首先我们批量索引一些数据：

```
POST /cars/_bulk
{ "index": {}}
{ "price" : 10000, "color" : "red", "make" : "honda", "sold" : "2014-10-28" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 30000, "color" : "green", "make" : "ford", "sold" : "2014-05-18" }
{ "index": {}}
{ "price" : 15000, "color" : "blue", "make" : "toyota", "sold" : "2014-07-02" }
{ "index": {}}
{ "price" : 12000, "color" : "green", "make" : "toyota", "sold" : "2014-08-19" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 80000, "color" : "red", "make" : "bmw", "sold" : "2014-01-01" }
{ "index": {}}
{ "price" : 25000, "color" : "blue", "make" : "ford", "sold" : "2014-02-12" }

```

notice:批量插入最后一行有一个回车

有了数据，开始构建我们的第一个聚合。汽车经销商可能会想知道哪个颜色的汽车销量最好，用聚合可以轻易得到结果，用 `terms` 桶操作：

```
GET /cars/_search
{
    "size" : 0,
    "aggs" : { 
        "popular_colors" : { 
            "terms" : { 
              "field" : "color"
            }
        }
    }
}
```

Fielddata会耗费大量的堆内存，尤其是在加载大文件时。一旦fielddata被加载到内存中，他会在segment的生命周期一直存在。而且，搜索时处理fielddata还会导致速度下降甚至严重延迟，所以默认fielddata被禁用。

如果你试图对text字段进行排序、聚合或者访问值等操作，你将会遇到如下异常：

> Fielddata is disabled on text fields by default. Set fielddata=true on [your_field_name] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory.

## 解决办法

由于我们在测试时添加的都是文本类型的数据，需要对文本字段开启fielddata，需要使用 [PUT mapping API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-mapping.html)，执行如下命令设置fielddata=true:

```
PUT my_index/_mapping/my_type
{
  "properties": {
    "my_field": { 
      "type":     "text",
      "fielddata": true
    }
  }
}
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_aggregation_test_drive.html#CO184-1) | 聚合操作被置于顶层参数   `aggs` 之下（如果你愿意，完整形式 `aggregations` 同样有效）。 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_aggregation_test_drive.html#CO184-2) | 然后，可以为聚合指定一个我们想要名称，本例中是： `popular_colors` 。 |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/3.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_aggregation_test_drive.html#CO184-3) | 最后，定义单个桶的类型 `terms` 。                            |

聚合是在特定搜索结果背景下执行的，  这也就是说它只是查询请求的另外一个顶层参数（例如，使用 `/_search` 端点）。 聚合可以与查询结对，但我们会晚些在 [限定聚合的范围（Scoping Aggregations）](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_scoping_aggregations.html) 中来解决这个问题。

可能会注意到我们将 `size` 设置成 0 。我们并不关心搜索结果的具体内容，**所以将返回记录数设置为 0 来提高查询速度。 设置 `size: 0` 与 Elasticsearch 1.x 中使用 `count` 搜索类型等价。**

#### 添加度量指标

前面的例子告诉我们每个桶里面的文档数量，这很有用。  但通常，我们的应用需要提供更复杂的文档度量。  例如，每种颜色汽车的平均价格是多少？

为了获取更多信息，我们需要告诉 Elasticsearch 使用哪个字段，计算何种度量。  这需要将度量 *嵌套* 在桶内， 度量会基于桶内的文档计算统计结果。

让我们继续为汽车的例子加入  `average` 平均度量：

```
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": { 
            "avg_price": { 
               "avg": {
                  "field": "price" 
               }
            }
         }
      }
   }
}
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_adding_a_metric_to_the_mix.html#CO186-1) | 为度量新增 `aggs` 层。                 |
| ------------------------------------------------------------ | -------------------------------------- |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_adding_a_metric_to_the_mix.html#CO186-2) | 为度量指定名字： `avg_price` 。        |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/3.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_adding_a_metric_to_the_mix.html#CO186-3) | 最后，为 `price` 字段定义 `avg` 度量。 |

正如所见，我们用前面的例子加入了新的 `aggs` 层。这个新的聚合层让我们可以将 `avg` 度量嵌套置于 `terms` 桶内。实际上，这就为每个颜色生成了平均价格。

正如 `颜色` 的例子，我们需要给度量起一个名字（ `avg_price` ）这样可以稍后根据名字获取它的值。最后，我们指定度量本身（ `avg` ）以及我们想要计算平均值的字段（ `price` ）：

```
{
...
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "avg_price": { 
                  "value": 32500
               }
            },
            {
               "key": "blue",
               "doc_count": 2,
               "avg_price": {
                  "value": 20000
               }
            },
            {
               "key": "green",
               "doc_count": 2,
               "avg_price": {
                  "value": 21000
               }
            }
         ]
      }
   }
...
}
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_adding_a_metric_to_the_mix.html#CO187-1) | 响应中的新字段 `avg_price` 。 |
| ------------------------------------------------------------ | ----------------------------- |
|                                                              |                               |

尽管响应只发生很小改变，实际上我们获得的数据是增长了。之前，我们知道有四辆红色的车，现在，红色车的平均价格是 $32，500 美元。这个信息可以直接显示在报表或者图形中。

#### 嵌套桶

在我们使用不同的嵌套方案时，聚合的力量才能真正得以显现。 在前例中，我们已经看到如何将一个度量嵌入桶中，它的功能已经十分强大了。

但真正令人激动的分析来自于将桶嵌套进 *另外一个桶* 所能得到的结果。 现在，我们想知道每个颜色的汽车制造商的分布：

```
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": {
            "avg_price": { 
               "avg": {
                  "field": "price"
               }
            },
            "make": { 
                "terms": {
                    "field": "make" 
                }
            }
         }
      }
   }
}
```

拷贝为 cURL[在 Sense 中查看](http://localhost:5601/app/sense/?load_from=https://www.elastic.co/guide/cn/elasticsearch/guide/current/snippets/300_Aggregations/20_basic_example.json) 

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_buckets_inside_buckets.html#CO188-1) | 注意前例中的 `avg_price` 度量仍然保持原位。               |
| ------------------------------------------------------------ | --------------------------------------------------------- |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_buckets_inside_buckets.html#CO188-2) | 另一个聚合 `make` 被加入到了 `color` 颜色桶中。           |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/3.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_buckets_inside_buckets.html#CO188-3) | 这个聚合是 `terms` 桶，它会为每个汽车制造商生成唯一的桶。 |

**这里发生了一些有趣的事。   首先，我们可能会观察到之前例子中的 `avg_price` 度量完全没有变化**，还在原来的位置。 一个聚合的每个 *层级* 都可以有多个度量或桶， `avg_price` 度量告诉我们每种颜色汽车的平均价格。**它与其他的桶和度量相互独立。**

这对我们的应用非常重要，因为这里面有很多相互关联，但又完全不同的度量需要收集。聚合使我们能够用一次数据请求获得所有的这些信息。

另外一件值得注意的重要事情是我们新增的这个 `make` 聚合，它是一个 `terms` 桶（嵌套在 `colors` 、 `terms` 桶内）。这意味着它 会为数据集中的每个唯一组合生成（ `color` 、 `make` ）元组。

让我们看看返回的响应（为了简单我们只显示部分结果）：

```
{
...
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "make": { 
                  "buckets": [
                     {
                        "key": "honda", 
                        "doc_count": 3
                     },
                     {
                        "key": "bmw",
                        "doc_count": 1
                     }
                  ]
               },
               "avg_price": {
                  "value": 32500 
               }
            },

...
}
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_buckets_inside_buckets.html#CO189-1) | 正如期望的那样，新的聚合嵌入在每个颜色桶中。          |
| ------------------------------------------------------------ | ----------------------------------------------------- |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_buckets_inside_buckets.html#CO189-2) | 现在我们看见按不同制造商分解的每种颜色下车辆信息。    |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/3.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_buckets_inside_buckets.html#CO189-3) | 最终，我们看到前例中的 `avg_price` 度量仍然维持不变。 |

响应结果告诉我们以下几点：

-  红色车有四辆。 
-  红色车的平均售价是 $32，500 美元。 
-  其中三辆是 Honda 本田制造，一辆是 BMW 宝马制造。 

#### 最后的修改

让我们回到话题的原点，在进入新话题之前，对我们的示例做最后一个修改，    为每个汽车生成商计算最低和最高的价格：

```
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": {
            "avg_price": { "avg": { "field": "price" }
            },
            "make" : {
                "terms" : {
                    "field" : "make"
                },
                "aggs" : { 
                    "min_price" : { "min": { "field": "price"} }, 
                    "max_price" : { "max": { "field": "price"} } 
                }
            }
         }
      }
   }
}
```

拷贝为 cURL[在 Sense 中查看](http://localhost:5601/app/sense/?load_from=https://www.elastic.co/guide/cn/elasticsearch/guide/current/snippets/300_Aggregations/20_basic_example.json) 

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_one_final_modification.html#CO190-1) | 我们需要增加另外一个嵌套的 `aggs` 层级。 |
| ------------------------------------------------------------ | ---------------------------------------- |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_one_final_modification.html#CO190-2) | 然后包括 `min` 最小度量。                |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/3.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_one_final_modification.html#CO190-3) | 以及 `max` 最大度量。                    |

 得到以下输出（只显示部分结果）：

```
{
...
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "make": {
                  "buckets": [
                     {
                        "key": "honda",
                        "doc_count": 3,
                        "min_price": {
                           "value": 10000 
                        },
                        "max_price": {
                           "value": 20000 
                        }
                     },
                     {
                        "key": "bmw",
                        "doc_count": 1,
                        "min_price": {
                           "value": 80000
                        },
                        "max_price": {
                           "value": 80000
                        }
                     }
                  ]
               },
               "avg_price": {
                  "value": 32500
               }
            },
...
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_one_final_modification.html#CO191-1) [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_one_final_modification.html#CO191-2) | `min` 和 `max`  度量现在出现在每个汽车制造商（ `make` ）下面。 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
|                                                              |                                                              |

有了这两个桶，我们可以对查询的结果进行扩展并得到以下信息：

-  有四辆红色车。 
-  红色车的平均售价是 $32，500 美元。 
-  其中三辆红色车是 Honda 本田制造，一辆是 BMW 宝马制造。 
-  最便宜的红色本田售价为 $10，000 美元。 
-  最贵的红色本田售价为 $20，000 美元。 

### 范围限定的聚合

所有聚合的例子到目前为止，你可能已经注意到，我们的搜索请求省略了一个 `query` 。   整个请求只不过是一个聚合。

聚合可以与搜索请求同时执行，但是我们需要理解一个新概念： *范围* 。 默认情况下，聚合与查询是对同一范围进行操作的，也就是说，聚合是基于我们查询匹配的文档集合进行计算的。

让我们看看第一个聚合的示例：

```
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "colors" : {
            "terms" : {
              "field" : "color"
            }
        }
    }
}
```

我们可以看到聚合是隔离的。现实中，Elasticsearch 认为 "没有指定查询" 和 "查询所有文档" 是等价的。前面这个查询内部会转化成下面的这个请求：

```
GET /cars/transactions/_search
{
    "size" : 0,
    "query" : {
        "match_all" : {}
    },
    "aggs" : {
        "colors" : {
            "terms" : {
              "field" : "color"
            }
        }
    }
}
```

因为聚合总是对查询范围内的结果进行操作的，所以一个隔离的聚合实际上是在对   `match_all` 的结果范围操作，即所有的文档。

一旦有了范围的概念，我们就能更进一步对聚合进行自定义。我们前面所有的示例都是对 *所有* 数据计算统计信息的：销量最高的汽车，所有汽车的平均售价，最佳销售月份等等。

利用范围，我们可以问“福特在售车有多少种颜色？”诸如此类的问题。可以简单的在请求中加上一个查询（本例中为 `match` 查询）：

```
GET /cars/transactions/_search
{
    "query" : {
        "match" : {
            "make" : "ford"
        }
    },
    "aggs" : {
        "colors" : {
            "terms" : {
              "field" : "color"
            }
        }
    }
}
```

拷贝为 cURL[在 Sense 中查看](http://localhost:5601/app/sense/?load_from=https://www.elastic.co/guide/cn/elasticsearch/guide/current/snippets/300_Aggregations/40_scope.json) 

因为我们没有指定 `"size" : 0` ，所以搜索结果和聚合结果都被返回了：

```
{
...
   "hits": {
      "total": 2,
      "max_score": 1.6931472,
      "hits": [
         {
            "_source": {
               "price": 25000,
               "color": "blue",
               "make": "ford",
               "sold": "2014-02-12"
            }
         },
         {
            "_source": {
               "price": 30000,
               "color": "green",
               "make": "ford",
               "sold": "2014-05-18"
            }
         }
      ]
   },
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "blue",
               "doc_count": 1
            },
            {
               "key": "green",
               "doc_count": 1
            }
         ]
      }
   }
}
```

看上去这并没有什么，但却对高大上的仪表盘来说至关重要。 加入一个搜索栏可以将任何静态的仪表板变成一个实时数据搜索设备。  这让用户可以搜索数据，查看所有实时更新的图形（由于聚合的支持以及对查询范围的限定）。 这是 Hadoop 无法做到的！

#### 全局桶

通常我们希望聚合是在查询范围内的，但有时我们也想要搜索它的子集，而聚合的对象却是 *所有* 数据。   

例如，比方说我们想知道福特汽车与 *所有* 汽车平均售价的比较。我们可以用普通的聚合（查询范围内的）得到第一个信息，然后用 `全局`   桶获得第二个信息。

`全局` 桶包含 *所有* 的文档，它无视查询的范围。因为它还是一个桶，我们可以像平常一样将聚合嵌套在内：

```
GET /cars/transactions/_search
{
    "size" : 0,
    "query" : {
        "match" : {
            "make" : "ford"
        }
    },
    "aggs" : {
        "single_avg_price": {
            "avg" : { "field" : "price" } 
        },
        "all": {
            "global" : {}, 
            "aggs" : {
                "avg_price": {
                    "avg" : { "field" : "price" } 
                }

            }
        }
    }
}
```



| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_scoping_aggregations.html#CO196-1) | 聚合操作在查询范围内（例如：所有文档匹配 `ford` ） |
| ------------------------------------------------------------ | -------------------------------------------------- |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_scoping_aggregations.html#CO196-2) | `global` 全局桶没有参数。                          |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/3.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_scoping_aggregations.html#CO196-3) | 聚合操作针对所有文档，忽略汽车品牌。               |

`single_avg_price` 度量计算是基于查询范围内所有文档，即所有 `福特` 汽车。`avg_price` 度量是嵌套在 `全局` 桶下的，这意味着它完全忽略了范围并对所有文档进行计算。聚合返回的平均值是所有汽车的平均售价。

如果能一直坚持读到这里，应该知道我们有个真言：尽可能的使用过滤器。它同样可以应用于聚合，在下一章中，我们会展示如何对聚合结果进行过滤而不是仅对查询范围做限定。

### 过滤和聚合

聚合范围限定还有一个自然的扩展就是过滤。因为聚合是在查询结果范围内操作的，任何可以适用于查询的过滤器也可以应用在聚合上。

#### 过滤

如果我们想找到售价在 $10,000 美元之上的所有汽车同时也为这些车计算平均售价，   可以简单地使用一个 `constant_score` 查询和 `filter` 约束：

```
GET /cars/transactions/_search
{
    "size" : 0,
    "query" : {
        "constant_score": {
            "filter": {
                "range": {
                    "price": {
                        "gte": 10000
                    }
                }
            }
        }
    },
    "aggs" : {
        "single_avg_price": {
            "avg" : { "field" : "price" }
        }
    }
}
```

这正如我们在前面章节中讨论过那样，从根本上讲，使用 `non-scoring` 查询和使用 `match` 查询没有任何区别。查询（包括了一个过滤器）返回一组文档的子集，聚合正是操作这些文档。使用 `filtering query` 会忽略评分，并有可能会缓存结果数据等等。

#### 过滤桶

但是如果我们只想对聚合结果过滤怎么办？   假设我们正在为汽车经销商创建一个搜索页面， 我们希望显示用户搜索的结果，但是我们同时也想在页面上提供更丰富的信息，包括（与搜索匹配的）上个月度汽车的平均售价。

这里我们无法简单的做范围限定，因为有两个不同的条件。搜索结果必须是 `ford` ，但是聚合结果必须满足 `ford` AND `sold > now - 1M` 。

为了解决这个问题，我们可以用一种特殊的桶，叫做 `filter` （注：过滤桶） 。  我们可以指定一个过滤桶，当文档满足过滤桶的条件时，我们将其加入到桶内。

查询结果如下：

```
GET /cars/transactions/_search
{
   "size" : 0,
   "query":{
      "match": {
         "make": "ford"
      }
   },
   "aggs":{
      "recent_sales": {
         "filter": { 
            "range": {
               "sold": {
                  "from": "now-1M"
               }
            }
         },
         "aggs": {
            "average_price":{
               "avg": {
                  "field": "price" 
               }
            }
         }
      }
   }
}
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_filter_bucket.html#CO197-1) | 使用 `过滤` 桶在 `查询` 范围基础上应用过滤器。           |
| ------------------------------------------------------------ | -------------------------------------------------------- |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_filter_bucket.html#CO197-2) | `avg` 度量只会对 `ford` 和上个月售出的文档计算平均售价。 |

因为 `filter` 桶和其他桶的操作方式一样，所以可以随意将其他桶和度量嵌入其中。所有嵌套的组件都会 "继承" 这个过滤，这使我们可以按需针对聚合过滤出选择部分。

### 后过滤器

目前为止，我们可以同时对搜索结果和聚合结果进行过滤（不计算得分的 `filter` 查询），以及针对聚合结果的一部分进行过滤（ `filter` 桶）。

我们可能会想，"只过滤搜索结果，不过滤聚合结果呢？"  答案是使用 `post_filter` 。

它是接收一个过滤器的顶层搜索请求元素。这个过滤器在查询 *之后* 执行（这正是该过滤器的名字的由来：它在查询之后 `post` 执行）。正因为它在查询之后执行，它对查询范围没有任何影响，所以对聚合也不会有任何影响。

我们可以利用这个行为对查询条件应用更多的过滤器，而不会影响其他的操作，就如 UI 上的各个分类面。让我们为汽车经销商设计另外一个搜索页面，这个页面允许用户搜索汽车同时可以根据颜色来过滤。颜色的选项是通过聚合获得的：

```
GET /cars/transactions/_search
{
    "size" : 0,
    "query": {
        "match": {
            "make": "ford"
        }
    },
    "post_filter": {    
        "term" : {
            "color" : "green"
        }
    },
    "aggs" : {
        "all_colors": {
            "terms" : { "field" : "color" }
        }
    }
}
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_post_filter.html#CO198-1) | `post_filter` 元素是 `top-level` 而且仅对命中结果进行过滤。 |
| ------------------------------------------------------------ | ----------------------------------------------------------- |
|                                                              |                                                             |

`查询` 部分找到所有的 `ford` 汽车，然后用 `terms` 聚合创建一个颜色列表。因为聚合对查询范围进行操作，颜色列表与福特汽车有的颜色相对应。

最后， `post_filter` 会过滤搜索结果，只展示绿色 `ford` 汽车。这在查询执行过 *后* 发生，所以聚合不受影响。

这通常对 UI 的连贯一致性很重要，可以想象用户在界面商选择了一类颜色（比如：绿色），期望的是搜索结果已经被过滤了，而 *不是* 过滤界面上的选项。如果我们应用 `filter` 查询，界面会马上变成 *只* 显示 `绿色` 作为选项，这不是用户想要的！

![警告](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/warning.png)

**性能考虑**

当你需要对搜索结果和聚合结果做不同的过滤时，你才应该使用 `post_filter` ，  有时用户会在普通搜索使用 `post_filter` 。

不要这么做！ `post_filter` 的特性是在查询 *之后* 执行，任何过滤对性能带来的好处（比如缓存）都会完全失去。

在我们需要不同过滤时， `post_filter` 只与聚合一起使用。

### 小结

选择合适类型的过滤（如：搜索命中、聚合或两者兼有）通常和我们期望如何表现用户交互有关。选择合适的过滤器（或组合）取决于我们期望如何将结果呈现给用户。

-  在 `filter` 过滤中的 `non-scoring` 查询，同时影响搜索结果和聚合结果。 
-  `filter` 桶影响聚合。 
-  `post_filter` 只影响搜索结果。 
