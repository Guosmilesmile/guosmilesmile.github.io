---
title: Elasticsearch 常用操作
date: 2020-03-14 20:55:40
tags:
categories:
	- Elasticsearch
---

本篇记录的是日常使用过程中一些常用的操作，作为笔记以供翻阅。Elasticsearch 使用的版本是1.6.x，其他版本出入有待确定。


### 查询写入的bucket队列

```
GET _cat/thread_pool?v
```

### 查询routing落在哪个服务器

```
GET {INDEX-NAME}/_search_shards?routing={ROUTING-KEY}
```

### 修改特定索引对应的分组

```
PUT index_name/_settings
{
   "index.routing.allocation.include.group" : ""
}
```

### 迁移的开启和关闭

```
PUT /index_name/_settings
{
  "index" : {
    "routing" : {
      "allocation" : {   
        "enable" : "none"
      }
    }
  }
}
```
cluster.routing.allocation.enable: 哪些分片可以参与重新分配。选项有：all(default), primaries(主分片), new_primaries(新增加的主分片), none.


### 修改特定索引在每个node上的个数


```
PUT index_name/_settings
{
   "index.routing.allocation.total_shards_per_node" : 5
}
```

### cancel shard relocation

```
POST /_cluster/reroute
{
  "commands": [
    {
      "cancel": {
        "index": "index_name",
        "shard": 0,
        "node": "target_node"
      }
    }
  ]
}
```

### 修改副本数量

```
PUT /my_temp_index/_settings
{
    "number_of_replicas": 1
}
```
### 丢失shard新建

```
POST /_cluster/reroute   
{
    "commands" : [ 
        {
          "allocate" : {
              "index" : "index_name",
              "shard" : 1,
              "node" : "node_name",
              "allow_primary": true
          }
        }
    ]
}
```

### tag cold 

```
PUT /index_name/_settings
{
  "index": {
    "routing": {
      "allocation": {
        "total_shards_per_node": 5,
        "enable": "all",
        "require": {
          "tag": "cold"
        }
      }
    },
    "number_of_replicas": "1"
  }
}


```

```
PUT index_name/_settings
{
   "index.routing.allocation.require.tag" : "cold"
}
```

### 删除特定文档


```
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/_query?q=user:kimchy'

$ curl -XDELETE 'http://localhost:9200/twitter/tweet/_query' -d '{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```


### disable merge throttling entirely

如果只为了导入而不在意查询，可以disable merge throttling entirely，可以加快导入速度

```
PUT /_cluster/settings
{
    "transient" : {
        "indices.store.throttle.type" : "none" 
    }
}
```

```
PUT /_cluster/settings
{
    "transient" : {
        "indices.store.throttle.type" : "none" 
    }
}
```

### 移除数据节点


```
PUT /_cluster/settings
{
  "transient" :{
      "cluster.routing.allocation.exclude._ip" : "10.0.0.1"
   }
}
```

### 更改group

```
PUT /index_name/_settings
{
  "index": {
    "routing": {
      "allocation": {
        "include": {
          "group": "web1,web2,web3"
        },
        "require": {
          "tag": "hot"
        }
      }
    }
  }
}
```