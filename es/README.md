# es


# es目录
.

|--- es es中间间

    |--- config 配置文件：插件配置文件、es的集群的配置文件
    
       |--- my_analysis es自定义近义词功能配置
    
    |---   data 数据映射
    
    |---   plugins 插件安装
    
    
#docker-compose.yml文件解释

```
version: '3.1'
services:
  elasticsearch:
    image: elasticsearch:7.8.1
    environment:
      - ES_JAVA_OPTS=-Xms128m -Xmx128m
      #单节点启动
      - discovery.type=single-node
    volumes:
      - ./data:/usr/share/elasticsearch/data
      #ik中文分词
      - ./plugins:/usr/share/elasticsearch/plugins
      #近义词配置文件,用内置的synonym实现同义词功能
      - ./config/my_analysis:/usr/share/elasticsearch/config/my_analysis
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - elk
networks:
    elk:
        external: true
```



#测试实验

我们来做一个，利用ik进行中分分词，和近义词搜索的数据。

ik只提供了分词，没有近义词，所以我们需要自定义一个分析器

## 测试实验必须满足以下的点


- 建立自定义分析器
- 自定义分析器包含：ik分词、近义词替换
- 建立测试mapping
- 导入测试数据
- 查询验证结果


## es倒排索引这分析器知识点

[参看文案-es权威指南](https://wiki.jikexueyuan.com/project/elasticsearch-definitive-guide-cn/030_Data/30_Create.html)

简单的总结：
  es做倒排索引查询时，会先经过分析器处理，分析器会经过:
 - char_filter  数据先进行一边过滤，可以用内置的过滤器什么鬼html标签过滤之类的
 - tokenizer 将数据分词，主要是用于生成倒排索引
 - filter  分词出来之后的数据，在进行统一处理转化（比如：大写的转小写字母，近义词替换）
 


### 测试索引的分析器解释

```

PUT my_test
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_ik_synonym": {    //自定义分析器的名字
          "type": "custom",   //类型，规定好的语法来的
          "tokenizer": "ik_max_word",  //用ik分词器去将文档分词
          "filter": [            //分词后，每个词语的转换器，我们这个案例是近义词替换
            "my_synonym_filter"  //自定义转换器名字（我叫他近义词转化器）
          ]
        }
      },
      "filter": {      //定义过滤器的语法
        "my_synonym_filter": {  //我们自定义转化器的名字
          "type": "synonym",   //类型必须是es 内置的 synonym
          "synonyms_path": "my_analysis/synonym.txt"  //近义词相对config的路径
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content":{    //文档只有content字段
        "type":"text",
        "analyzer": "my_ik_synonym",   //文档使用我们自定义的分析器做倒排索引分析
        "search_analyzer": "my_ik_synonym"  //请求过来的查询参与使用我们自定义的分析器做倒排索引
      }
    }
  }
}

```



### 导入测试数据

```
POST my_test/_create/1
{"content":"美国留给伊拉克的是个烂摊子吗"}
POST my_test/_create/2
{"content":"公安部：各地校车将享最高路权"}
POST my_test/_create/3
{"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}
POST my_test/_create/4
{"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}

```

### 执行搜索

```
POST mytest/_search

{
    "query" : { "match" : { "content" : "中国" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}


#结果

{
    "took": 14,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 2,
        "hits": [
            {
                "_index": "index",
                "_type": "fulltext",
                "_id": "4",
                "_score": 2,
                "_source": {
                    "content": "中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"
                },
                "highlight": {
                    "content": [
                        "<tag1>中国</tag1>驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首 "
                    ]
                }
            },
            {
                "_index": "index",
                "_type": "fulltext",
                "_id": "3",
                "_score": 2,
                "_source": {
                    "content": "中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"
                },
                "highlight": {
                    "content": [
                        "均每天扣1艘<tag1>中国</tag1>渔船 "
                    ]
                }
            }
        ]
    }
}
```


如果把content换成'大陆'，是搜不出任何文档的数据的

```
POST mytest/_search

{
    "query" : { "match" : { "content" : "大陆" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}

#将会没有任何结果
```



### 编辑synonym.txt文件生成近义词

```
#这种方式表示，中国、大陆、中华 都替换成中国
中国,大陆,中华 => 中国

#这种方式表示，中国、大陆、中华，只要有一个词语命中，都会添加其他的词语进去。比如，命中了大陆,会给内容多添加中国、中华两个词进去
中国,大陆,中华
```


### 重启es，有近义词之后，我们再次搜索数据

```

POST mytest/_search

{
    "query" : { "match" : { "content" : "大陆" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}


# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
# 哗啦啦啦啦的有数据里，自己去实验吧
```


