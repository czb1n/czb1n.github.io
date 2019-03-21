---
layout: post
author: czb1n
title:  "Elasticsearch基础使用"
date:   2015-08-17 22:17:59
categories: [Database]
tags: [Database, Server, ElasticSearch]
---

### 1. 安装ElasticSearch。
[Elasticsearch - 下载地址](http://www.elastic.co/downloads/elasticsearch)

* windows
控制台进入 ( ***cd*** ) ElasticSearch目录下的 ***bin*** 目录, 然后运行 ***elasticsearch.bat***
* linux
终端进入 ( ***cd*** ) ElasticSearch目录下的 ***bin*** 目录, 然后运行 ***./elasticsearch***

* 执行`$ curl localhost:9200`返回类似以下片段即代表运行成功

``` Java
{
  "status" : 200,
  "name" : "Changeling",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "1.7.1",
    "build_hash" : "b88f43fc40b0bcd7f173a1f9ee2e97816de80b19",
    "build_timestamp" : "2015-07-29T09:54:16Z",
    "build_snapshot" : false,
    "lucene_version" : "4.10.4"
  },
  "tagline" : "You Know, for Search"
}
```

### 2. 简单配置中文分词mmseg插件。
* 下载mmseg插件的源码包 :
[mmseg - 插件源码](https://github.com/medcl/elasticsearch-analysis-mmseg)

* 下载mmseg插件的jar包 :
[mmseg - 插件jar包](https://github.com/medcl/elasticsearch-rtf/tree/master/plugins/analysis-mmseg)

* 在ElasticSearch的文件夹下的plugins文件夹中新建一个analysis-mmseg文件夹, 然后把mmseg插件的jar包放进去。
* 把mmseg的源码包中的config文件夹下的mmseg文件夹移动到ElasticSearch的文件夹下的config文件夹。
* 把以下代码追加到ElasticSearch文件夹下的config文件夹下的elasticsearch.yml中 ***(注意缩进)***

``` Java
index:
  analysis:
    analyzer:
      mmseg:
          alias: [news_analyzer, mmseg_analyzer]
          type: org.elasticsearch.index.analysis.MMsegAnalyzerProvider
index.analysis.analyzer.default.type : "mmseg"
```

* 完成之后在创建mapping时就可以使用indexAnalyzer来指定索引分词器和searchAnalyzer来指定搜索分词器。
如 :
"indexAnalyzer": "mmseg",
"searchAnalyzer": "mmseg",

``` Java
$ curl -XPOST http://localhost:9200/index/fulltext/_mapping -d'
{
    "fulltext": {
             "_all": {
            "indexAnalyzer": "mmseg",
            "searchAnalyzer": "mmseg",
            "term_vector": "no",
            "store": "false"
        },
        "properties": {
            "content": {
                "type": "string",
                "store": "no",
                "term_vector": "with_positions_offsets",
                "indexAnalyzer": "mmseg",
                "searchAnalyzer": "mmseg",
                "include_in_all": "true",
                "boost": 8
            }
        }
    }
}'
```

### 3. 配置文件 ***elasticsearch.yml*** 部分说明。

``` Java
#cluster.name: elasticsearch		//配置集群的名称, 如果不指定则默认为elasticsearch
#node.name: "Franz Kafka"			//配置节点的名称, 如果不指定,每次运行时elasticsearch会随机指定一个名称
#node.master: true					//设置当前这个节点是否可以指定为集群的master, 默认为true
#node.data: true					//设置当前这个节点是否可以存储和操作数据, 默认为true
#discovery.zen.ping.unicast.hosts: ["host1", "host2:port"]		//指定在节点运行开始时探索集群master的ip
```

### 4. Elasticsearch shell的一些使用方法。

``` Java
/**
	创建:
	twitter 为 index(索引名).
	tweet 为 type(文档类型).
	1 为 id.
	*/
$ curl -XPUT 'http://localhost:9200/twitter/tweet/1' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'

/**
	获取:
	*/
$ curl -XGET 'http://localhost:9200/twitter/tweet/1'

/**
	删除:
	*/
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1'		//删除单条记录
$ curl -XDELETE 'http://localhost:9200/twitter'		//删除整个文档

/**
	更新:
	*/
$ curl -XPOST 'localhost:9200/twitter/tweet/1/_update' -d '{
    "user" : "Benjamin",
    "post_date" : "2015-08-17T21:50:45",
    "message" : "trying to update Elasticsearch"
}'

/**
	搜索:
	*/
$ curl -XGET 'http://localhost:9200/twitter/tweet/_search?q= '{
    "user" : "Benjamin"
}
$ curl -XGET 'http://localhost:9200/twitter/tweet/_search?pretty' -d '{
	"query":{
		"match":{
			"user":"Benjamin"
		}
	}
}'

/**
	统计:
	*/
$ curl -XGET 'http://localhost:9200/twitter/_count
```

### 5. Spring Data - Elasticsearch。

``` Java

ElasticsearchOperations elasticsearchOperations;

// 创建 / 修改
elasticsearchOperations.index(new IndexQuery());

// 获取
elasticsearchOperations.queryForObject(new GetQuery(), Class<T>);

// 删除
elasticsearchOperations.delete(new DeleteQuery(), Class<T>);

// 搜索
SearchQuery searchQuery = new NativeSearchQueryBuilder().
                withQuery(QueryBuilder).
                withFilter(FilterBuilder).
                withPageable(new PageRequest(pageIndex, pageSize)).
                build();
elasticsearchOperations.queryForList(searchQuery, Class<T>);

// 显示搜索匹配分数
// org.elasticsearch.client.Client client;
// ElasticSearchIndex 为搜索指定的索引名
SearchResponse searchResponse = client.prepareSearch(ElasticSearchIndex).setQuery(QueryBuilder).execute().actionGet();
SearchHits searchHits = searchResponse.getHits();
for(SearchHit searchHit : searchHits){
	System.out.println(searchHit.getScore());
}
```
