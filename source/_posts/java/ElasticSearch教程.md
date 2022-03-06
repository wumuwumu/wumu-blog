---
title: ElasticSearch教程
abbrlink: 88463ccb
date: 2020-12-08 15:00:00
---

# ElasticSearch概述

es的基本操作

```shell
GET _analyze
{
  "analyzer": "ik_max_word",
  "text": ["我叫做梧木"]
}

GET _analyze
{
  "analyzer": "ik_smart",
  "text": ["我叫做梧木"]
}

PUT /test3/_doc/2
{
  "name":"wumu",
  "age":24
}
PUT /test3/_doc/5
{
  "name":"wumu",
  "age":"123"
}

DELETE /test3/_doc/2

PUT /test3/_doc/5
{
  "name":"wumu1",
  "age":122
}

POST /test3/_update/5
{
  "doc":{"name":"wumu1",
  "age":12}
}

GET /test3/_doc/5

GET /test3/_search?q=name:wumu



GET _cat/indices


PUT /test2
{
  "mappings": {
    "properties": {
      "name":{
        "type": "text"
      },
      "age":{
        "type": "long"
      },
      "birthday":{
        "type": "date"
      }
    }
  }
}
```

# es复杂查询

```

```

