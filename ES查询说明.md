
### Create "my_index"" with a single primary shard
<pre><code>
PUT /my_index
{ "settings": { "number_of_shards": 1 }}
</code></pre>

### Analyze the `text` with the analyzer
<pre><code>
GET /_analyze
{
  "analyzer" : "ik_max_word",
  "text" : ["this is a test", "the second text"]
}
</code></pre>

### 忽略词条的出现频次，只在乎是否出现过
将index_options设置为docs会禁用词条频度以及词条位置信息。
norms:词条所在文章长度越短，那么其权重就越高。可以设置将其禁用。
(例如同样一个词在title中和content中，content比title长，title中的权重就比content高。)
<pre><code>
PUT /my_index
{
  "mappings": {
    "doc": {
      "properties": {
        "text": {
          "type":          "string",
          "index_options": "docs",
          "norms": { "enabled": false }
        }
      }
    }
  }
}
</code></pre>

### Query-Time Boosting(查询期间提升)
<p>title的boost为2，计算出来评分更高，content为1</p>
<pre><code>
GET /_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "title": {
              "query": "quick brown fox",
              "boost": 2
            }
          }
        },
        {
          "match": {
            "content": "quick brown fox"
          }
        }
      ]
    }
  }
}
</code></pre>

### Boosting an Index(提升索引)
<p>查询所有的docs_2014_*开头的index，其中2014_10为3，2014_09为2.
可以区别对待每一个index</p>
<pre><code>
GET /docs_2014_*/_search
{
  "indices_boost": {
    "docs_2014_10": 3,
    "docs_2014_09": 2
  },
  "query": {
    "match": {
      "text": "quick brown fox"
    }
  }
}
</code></pre>

### Not Quite Not(不完全的不)
<pre><code>
GET /_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "text": "apple"
        }
      },
      "must_not": {
        "match": {
          "text": "pie tart fruit crumble tree"
        }
      }
    }
  }
}
</code></pre>
apple不能和下面的这些词一起出现，但是这样过于严格。可以使用下面的
当apple和下面的这些词一起出现时，评分降低。

<pre><code>
GET /_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "text": "apple"
        }
      },
      "negative": {
        "match": {
          "text": "pie tart fruit crumble tree"
        }
      },
      "negative_boost": 0.5
    }
  }
}
</code></pre>

### Ignoring TF/IDF
constant_score不关心TF/IDF，包含词的为1不包含的为0，不考虑TF/IDF。
也可以设置boost来提高score。
note:最后的得分不仅仅是所有匹配句子分数的总和，协调因子和查询归一化还是会计算的。
<pre><code>
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "constant_score": {
          "query": { "match": { "description": "wifi" }}
        }},
        { "constant_score": {
          "query": { "match": { "description": "garden" }}
        }},
        { "constant_score": {
          "boost":   2
          "query": { "match": { "description": "pool" }}
        }}
      ]
    }
  }
}
</code></pre>

### Boosting by Popularity
[详细文档说明](https://www.elastic.co/guide/en/elasticsearch/guide/current/boosting-by-popularity.html)
function_score:包含查询主体和函数.
例如下面的每个文档都需要经过函数field_value_factor.
field_value_factor函数是计算用户提交的的votes值，使用log1p来使评分更加平滑，可以添加factor使数据更加平滑。
`new_score = old_score * log(1 + factor * number_of_votes)`
modifier可设为：none(默认), log, log1p, log2p, ln, ln1p, ln2p, square, sqrt, and reciprocal.

假设乘以field_value_factor还是影响太大，我们可以使用boost_mode来进行修正。
multiply(乘以_score), sum, min, max, replace
最后还可以设置max_boost来限制函数的结果最大值。这里永远不会大于1.5

<pre><code>
GET /blogposts/post/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field":    "votes",
        "modifier": "log1p",
        "factor":   0.1
      },
      "boost_mode": "sum",
      "max_boost":  1.5
    }
  }
}
</code></pre>

### 根据过滤子集来提升(Boosting Filtered Subsets)
functions: 函数列表
score_mode: 每个函数都会有返回值，我们需要一种方法将所有函数的返回值生成单一的值，再合并到原有的_score。 取值范围有:
multiply sum avg max min first(只使用第一个函数结果)

<pre><code>
GET /_search
{
  "query": {
    "function_score": {
      "filter": {
        "term": { "city": "Barcelona" }
      },
      "functions": [
        {
          "filter": { "term": { "features": "wifi" }},
          "weight": 1
        },
        {
          "filter": { "term": { "features": "garden" }},
          "weight": 1
        },
        {
          "filter": { "term": { "features": "pool" }},
          "weight": 2
        }
      ],
      "score_mode": "sum",
    }
  }
}
</code></pre>

### 随机分值计算(Random Scoring)
random_score: 使用当前的查询，拥有相同的_score的结果每次的返回顺序都是相同的。此时引入一定程度的随机性会更好，来保证拥有相同分值的文档都能有同等的展示机会。
seed: 可以是用户的会话(Session)ID，当传递相同的id时，返回的结果相同。如果重新索引，结果也会变化。
<pre><code>
GET /_search
{
  "query": {
    "function_score": {
      "filter": {
        "term": { "city": "Barcelona" }
      },
      "functions": [
        {
          "filter": { "term": { "features": "wifi" }},
          "weight": 1
        },
        {
          "filter": { "term": { "features": "garden" }},
          "weight": 1
        },
        {
          "filter": { "term": { "features": "pool" }},
          "weight": 2
        },
        {
          "random_score": {
            "seed":  "the users session id"
          }
        }
      ],
      "score_mode": "sum"
    }
  }
}
</code></pre>

### The Closer, The Better
衰减函数有 linear, exp, gauss 下面是一副图，来说明3者的区别。
3个函数都有同样的参数:
origin 中心点
offset 扩大中心点的范围，取值为`-offset <= origin <= +offset` _score为1
scale  从offset到scale的范围，使用下面的decay作为评分
decay  衰减值，默认值为0.5

![](https://raw.githubusercontent.com/yonddream/learn/master/elas_closer.png "衰减函数对比")
设置一个中心点origin，中心点2公里范围内的按分数1计算，2公里到3公里处的按0.5计算。

<pre><code>
GET /_search
{
  "query": {
    "function_score": {
      "functions": [
        {
          "gauss": {
            "location": {
              "origin": { "lat": 51.5, "lon": 0.12 },
              "offset": "2km",
              "scale":  "3km"
            }
          }
        },
        {
          "gauss": {
            "price": {
              "origin": "50",
              "offset": "50",
              "scale":  "20"
            }
          },
          "weight": 2
        }
      ]
    }
  }
}
</code></pre>

### Scoring with Scripts
ES2以上版本默认禁用了动态脚本功能.通过修改配置文件开启动态脚本功能  
编辑 config/elasticsearch.yml 文件，添加  
script.inline: on  
script.indexed: on   
script.file: on  

script_score: 查询到的内容经过script的计算。
params里保存的是下面脚本的变量
<pre><code>
GET /_search
{
  "function_score": {
    "functions": [
      { ...location clause... },
      { ...price clause... },
      {
        "script_score": {
          "params": {
            "threshold": 80,
            "discount": 0.1,
            "target": 10
          },
          "script": "price  = doc['price'].value; margin = doc['margin'].value;
          if (price < threshold) { return price * margin / target };
          return price * (1 - discount) * margin / target;"
        }
      }
    ]
  }
}
</code></pre>

### Changing Similarities
<pre><code>
PUT /my_index
{
  "mappings": {
    "doc": {
      "properties": {
        "title": {
          "type":       "string",
          "similarity": "BM25"
        },
        "body": {
          "type":       "string",
          "similarity": "default"
        }
      }
  }
}
</code></pre>

### 高频和低频分开管理
cutoff_frequency: 分割哪些是高频哪些是低频
下面的低于0.01的低频的需要条件为and，高频的的需要75%就行了。
<pre><code>
{
  "common": {
    "text": {
      "query":                  "Quick and the dead",
      "cutoff_frequency":       0.01,
      "low_freq_operator":      "and",
      "minimum_should_match": {
        "high_freq":            "75%"
      }
    }
  }
}
</code></pre>

### 同义词&映射
my_synonym_filter: 我们设置的同义词filter
synonym: 设置同义词
char_filter中设置了一个emoticons做了个符号到字母的映射。
<pre><code>
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "emoticons": {
          "type": "mapping",
          "mappings": [
            ":)=>emoticon_happy",
            ":(=>emoticon_sad"
          ]
        }
      },
      "filter": {
        "my_synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "british,english",
            "queen,monarch"
          ]
        }
      },
      "analyzer": {
        "my_synonyms": {
          "char_filter": "emoticons",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "my_synonym_filter"
          ]
        }
      }
    }
  }
}
</code></pre>

### 模糊查询
模糊查询只支持match和multi_match
<pre><code>
GET /my_index/my_type/_search
{
  "query": {
    "multi_match": {
      "fields":  [ "text", "title" ],
      "query":     "SURPRIZE ME!",
      "fuzziness": "AUTO"
    }
  }
}
</code></pre>


## 聚合
相当于MySQL中的group by
`SELECT COUNT(color) FROM table GROUP BY color`
桶(Buckets)一个桶就是满足特定条件的一个文档集合
指标(Metrics) 对桶中的数据进行计算，多数指标仅仅是简单的数学运算(比如，min，mean，max以及sum)。


### 桶中的桶(Buckets inside Buckets)
[详细实例](http://localhost:5601/app/sense?load_from=https://www.elastic.co/guide/en/elasticsearch/guide/current/snippets/300_Aggregations/20_basic_example.json)
下面的例子是划分有多少不同颜色的汽车，不同颜色汽车的平均价钱，来自的制造商。为每个制造商添加两个指标来计算最低和最高价格：
<pre><code>
GET /cars/transactions/_search
{
  "size": 0,
  "aggs": {
    "colors": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "avg_price": {"avg": {"field": "price"}
        },
        "make": {
          "terms": {"field": "make"},
          "aggs": {
            "min_price": {"min": {"field": "price"}},
            "max_price": {"max": {"field": "price"}}
          }
        }
      }
    }
  }
}
</code></pre>

### 创建条形图(Building Bar Chart)
histogram是个关键字，生成柱图。定义了一个histogram的聚合将价格以20000分割，然后镶嵌了一个sum指标：
<pre><code>
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs":{
      "price":{
         "histogram":{
            "field": "price",
            "interval": 20000
         },
         "aggs":{
            "revenue": {
               "sum": {
                 "field" : "price"
               }
             }
         }
      }
   }
}
</code></pre>

当然，你可以使用任何生成类别和统计信息的聚合来创建条形图，并不仅限于使用histogram桶。让我们创建一个受欢迎的汽车制造商的条形图，其中包含了它们的平均价格和标准误差(Standard Error)。需要使用的而是terms桶以及一个extended_stats指标：
<pre><code>
GET /cars/transactions/_search
{
  "size" : 0,
  "aggs": {
    "makes": {
      "terms": {
        "field": "make",
        "size": 10
      },
      "aggs": {
        "stats": {
          "extended_stats": {
            "field": "price"
          }
        }
      }
    }
  }
}
</code></pre>

### 时间数据处理(Looking at Time)
date_histogram按时间段分割数据生成桶，min_doc_count参数会强制返回空桶，extended_bounds参数会强制返回一整年的数据。
<pre><code>
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "sales": {
         "date_histogram": {
            "field": "sold",
            "interval": "month",
            "format": "yyyy-MM-dd",
            "min_doc_count" : 0,
            "extended_bounds" : {
                "min" : "2014-01-01",
                "max" : "2014-12-31"
            }
         }
      }
   }
}
</code></pre>


### 嵌套的date_histogram
创建一个用来显示每个季度，所有制造商的总销售额的聚合。同时，我们也会在每个季度为每个制造商单独计算其总销售额，因此我们能够知道哪种汽车创造的收益最多：
<pre><code>
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "sales": {
         "date_histogram": {
            "field": "sold",
            "interval": "month",
            "format": "yyyy-MM-dd",
            "min_doc_count" : 0,
            "extended_bounds" : {
                "min" : "2014-01-01",
                "max" : "2014-12-31"
            }
         },
         "aggs": {
            "top_selling": {
               "terms": {
                  "field": "make",
                  "size": 1
               },
               "aggs": {
                  "avg_price": {
                     "avg": { "field": "price" }
                  }
               }
            }
         }
      }
   }
}
</code></pre>


### 聚合作用域(Scoping Aggregations)
聚合总是在查询的作用域下工作的，所以下面的single_avg_price计算的时候上面查询结果的平均价钱。
all中设置了个global: {}，改变了作用域，成了一个全局桶，全局桶会包含你的所有文档。使这个计算指标所使用的是全部数据的桶。
<pre><code>
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
</code></pre>


### filtered查询
如果你想要找到所有售价高于10000美刀的车，同时也对这些车计算其平均价格，那么可以使用一个filtered查询：
把filter写在query同级，好像只能过滤文本但是聚合的时候只是看query的结果，所以需要写在query里面。
<pre><code>
GET /cars/transactions/_search?search_type=count
{
    "query" : {
        "filtered": {
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
</code></pre>


### 过滤桶(Filter Bucket)
获取制造商为ford，sold为上个月的集合。并求满足上面条件的平均值。
为了解决这一问题，我们使用一个名为filter的特殊桶。通过制定一个过滤器，当文档匹配了该过滤器的规则时，它就会被添加到桶中。
<pre><code>
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
</code></pre>

### 后置过滤器(Post Filter)
目前，我们有了用于过滤搜索结果和聚合的过滤器(filtered查询)，也有了用于过滤聚合中某一部分的过滤器(filter桶)。
你也许会好奇，“是否有一种过滤器只过滤搜索结果，而不过滤聚合呢？”这个问题的答案就是使用post_filter。
post_filter元素是一个顶层元素，只会对搜索结果进行过滤。

>警告：性能考量
只有当你需要对搜索结果和聚合使用不同的过滤方式时才考虑使用post_filter。有时一些用户会直接在常规搜索中使用post_filter。
不要这样做！post_filter会在查询之后才会被执行，因此会失去过滤在性能上帮助(比如缓存)。
post_filter应该只和聚合一起使用，并且仅当你使用了不同的过滤条件时。

<pre><code>
GET /cars/transactions/_search?search_type=count
{
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
</code></pre>


### 集合排序
order 设置排序规则，这里是镶嵌了aggs，并且按照里面的规则来排序。
<pre><code>
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "colors" : {
            "histogram" : {
              "field" : "price",
              "interval": 20000,
              "order": {
                "red_green_cars>stats.variance" : "asc"
              }
            },
            "aggs": {
                "red_green_cars": {
                    "filter": { "terms": {"color": ["red", "green"]}},
                    "aggs": {
                        "stats": {"extended_stats": {"field" : "price"}}
                    }
                }
            }
        }
    }
}
</code></pre>


### Finding Distinct Counts
想要实现MySQL `SELECT DISTINCT(color) FROM cars` 这样的功能可能需要cardinality关键字。统计不重复的颜色有几个。
precision_threshold 可以设置精度，取值范围0-40000。
计算唯一是使用的 HyperLogLog++ (HLL)算法。方法是对你输入的内容进行hash处理，然后转换成二进制，然后做概率性判断。
[文档还有详细的用其他hash方法代替](https://www.elastic.co/guide/en/elasticsearch/guide/current/cardinality.html)

<pre><code>
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "distinct_colors" : {
            "cardinality" : {
              "field" : "color",
              "precision_threshold" : 100
            }
        }
    }
}
</code></pre>


### Calculating Percentiles
计算百分比需要用到percentiles关键字。默认percentiles会返回[1, 5, 25, 50, 75, 95, 99]
下面的例子表示按时区分桶%1到99%的用户加载所用时间的排名，我们只对高延迟用户感兴趣，所以设置了percents，只显示高延迟用户的加载时间。然后计算了个整体平均事件来做对比。
<pre><code>
GET /website/logs/_search
{
    "size" : 0,
    "aggs" : {
        "zones" : {
            "terms" : {
                "field" : "zone"
            },
            "aggs" : {
                "load_times" : {
                    "percentiles" : {
                      "field" : "latency",
                      "percents" : [50, 95.0, 99.0]
                    }
                },
                "load_avg" : {
                    "avg" : {
                        "field" : "latency"
                    }
                }
            }
        }
    }
}
</code></pre>


### Percentiles Ranks(百分比排名)
下面的例子使用了percentile_ranks，最后得到的结果可以看到我们设定的values在query得到的文档中的整体排名。
<pre><code>
GET /website/logs/_search
{
    "size" : 0,
    "aggs" : {
        "zones" : {
            "terms" : {
                "field" : "zone"
            },
            "aggs" : {
                "load_times" : {
                    "percentile_ranks" : {
                      "field" : "latency",
                      "values" : [210, 800]
                    }
                }
            }
        }
    }
}
</code></pre>


### 添加别名
下面的例子添加一个别名将logs_2014-10设置为当前logs，删除logs_2014-09的别名。  
将logs_2014-10添加到last_3_months，删除logs_2014-07  
<pre><code>
POST /_aliases
{
  "actions": [
    { "add":    { "alias": "logs_current",  "index": "logs_2014-10" }}, 
    { "remove": { "alias": "logs_current",  "index": "logs_2014-09" }}, 
    { "add":    { "alias": "last_3_months", "index": "logs_2014-10" }}, 
    { "remove": { "alias": "last_3_months", "index": "logs_2014-07" }}  
  ]
}
</code></pre>


### 索引模板
所有logstash开头的索引都会应用到该模板，新建的索引会直接加入last_3_months别名。  
<pre><code>
PUT /_template/my_logs 
{
  "template": "logstash-*", 
  "order":    1, 
  "settings": {
    "number_of_shards": 1 
  },
  "mappings": {
    "_default_": { 
      "_all": {
        "enabled": false
      }
    }
  },
  "aliases": {
    "last_3_months": {} 
  }
}
</code></pre>


### 索引优化
当作为日志系统来使用的时候，可能昨天的数据已经就可用性不太大了所以可以将复制设置为0。  
并优化将所有的索引放在一个文件中。如果以后需要的话也可以直接重新复制。  
<pre><code>
POST /logs_2014-09-30/_settings
{ "number_of_replicas": 0 }

POST /logs_2014-09-30/_optimize?max_num_segments=1

POST /logs_2014-09-30/_settings
{ "number_of_replicas": 1 }
</code></pre>
长时间没使用的索引，我们可以进行关闭。以后需要使用的时候再打开。  
关闭前需要先flush索引，确保没有正在处理的事务。    
<pre><code>
POST /logs_2014-01-*/_flush 
POST /logs_2014-01-*/_close 
POST /logs_2014-01-*/_open 
</code></pre>
新进索引的数据，需要更加强劲的机器来进行处理，时间过去很久的数据，    
被查询的几率降低，热度降低，所以可以选择不太强劲的机器来存储。  
node.box_type设置node类别，可以是任何名称来告诉ES来分配索引。  
<pre><code>
./bin/elasticsearch --node.box_type strong
PUT /logs_2014-10-01
{
  "settings": {
    "index.routing.allocation.include.box_type" : "strong"
  }
}

POST /logs_2014-09-30/_settings
{
  "index.routing.allocation.include.box_type" : "medium"
}
</code></pre>

### 自定义查询
<pre><code>
GET /doc_crawl_content/contents/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "tags": "服务"
          }
        },
        {
          "match": {
            "tags": "点击"
          }
        }
      ],
      "should": [
        {
          "multi_match": {
            "query":    "请问如果接了仓单之后做了仓单质押",
            "fields":   [ "title", "content" ],
            "minimum_should_match": "50%"
          }
        }
      ]
    }
  },
  "filter":{
    "term":{
      "category": "投资"
    }
  }
}
</code></pre>

