---
title: "[筆記] ElasticSearch 基礎操作"
categories:
  - 筆記
tags:
  - elasticsearch
toc: true
toc_label: Index
---

ElasticSearch入門隨筆


## 層級單位對應

|MySQL|Elastic|
|:--:|:--:|
|table|Index|
|Row|Document|
|column|Field|
|Schema|Mapping|
|SQL|DSL|

差異較大的部分為  
因 Index 存儲單位是json. 若沒有定義mapping , 整個index可以沒有統一field, index中裡面每一筆Document都可以是不同格式的json  


### 儲存與查找概念

- 存儲  
elasticsearch 存儲document時, 會根據field類型判斷將內容進行分詞, 並用倒排索引方式將document做索引標記  

- 查找  
elasticseartch 搜索一段 文字時  
也會將該文字拆為多組term  
該terms查找索引後 document id 出現次數愈高  
該document candiate score 分數愈高  
會將相關結果由score高至低輸出  


## 倒排索引

一種利用keyword作為索引的方式,  
elasticsearch 中是利用詞彙作為Index去查找document id  
為一對多, 

e.g.  

keyword| document_id
:-:|:-:
大毛| 1001,1008,1023
小毛| 1001,1010


e.g.   
搜索 "大毛與小毛"  
評分最高的是1001,. 其他 1008,1010,1023也會被查找出來  

output: 1001 1008 1010 1023  



## Index


### list all Index

```
GET _cat/indices?v
```



### create

idempotence, 創建一個名為shoppinga的index

```
PUT shopping
```

return
```
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "shopping"
}
```



### read

讀取index基本屬性

```
GET shopping
```

return

```
{
  "shopping" : {
    "aliases" : { },
    "mappings" : { },
    "settings" : {
      "index" : {
        "routing" : {
          "allocation" : {
            "include" : {
              "_tier_preference" : "data_content"
            }
          }
        },
        "number_of_shards" : "1",
        "provided_name" : "shopping",
        "creation_date" : "1699322452796",
        "number_of_replicas" : "1",
        "uuid" : "X3DBdgmCQAS41BoQYW8d0w",
        "version" : {
          "created" : "7170699"
        }
      }
    }
  }
}

```


### delete

```
delete shopping
```

return
```
{
  "acknowledged" : true
}

```




## Document


### create


#### non-idempotent  
每次創建都會隨機一串 id, 重複執行會重複創建, 該id可用來查詢用 類似SQL PrimaryKey

```
POST shopping/_doc
{
	"title":"IPhone100",
	"category":"Apple",
	"price":"999"
}
```


return
```
{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "UVaep4sBBjIGApZi34_r",  ## <<<<隨機產生
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```


#### idempotent  
產生的id 也可自已定義, 重複執行不影響, 僅version紀錄

```
PUT shopping/_doc/1000
{
	"title":"IPhone100",
	"category":"Apple",
	"price":"999"
}
```
return
```
{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "1000", ## << 固定
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 7,
  "_primary_term" : 1
}

```


### update

#### 全局更新  

其實等同create的 idempotent, 利用doc id 更新 __整個__ doc,  因此field 必須完整

```
PUT shopping/_doc/1000
{
	"title":"IPhone100",
	"category":"Apple",
	"price":"1999"
}
```
return
```

{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "1000",
  "_version" : 2,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 8,
  "_primary_term" : 1
}


```

#### 局部更新  

non-idempotent, 可修改原數值或是增加field, 沒有用put是因為只更新其中一個field無法保證該doc的整個狀態, 只能保證其中之一,  若下次執行時 其他feld已被改動 這次執行與上次執行後 對於整個doc的狀態是不相同的  為non-idempotent,

```
POST shopping/_update/1000
{
	"doc":{
		"title":"IPhone300"
	}
}
```

return
```
{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "1000",
  "_version" : 4,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 10,
  "_primary_term" : 1
}
```


```
POST shopping/_update/1000
{
	"doc":{
		"title22222":"IPhone300"
	}
}
```
return
```
{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "1000",
  "_version" : 5,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 11,
  "_primary_term" : 1
}

```




### delete

```
DELETE shopping/_doc/1000
```

return
```
{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "1000",
  "_version" : 7,
  "result" : "deleted",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 13,
  "_primary_term" : 1
}
```




### read

#### 查詢特定doc id

```
GET shopping/_doc/1000
```

return

```
{
  "_index" : "shopping",
  "_type" : "_doc",
  "_id" : "1000",
  "_version" : 1,
  "_seq_no" : 7,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "IPhone100",
    "category" : "Apple",
    "price" : "999"
  }
}

```

#### 查詢某index下所有document

```
GET shopping/_search
```
or   
```
GET shopping/_search
{
	"query": {
		"match_all":{}
	}
}

```

return  
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3, ## 表示共三筆
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "shopping", ## index name
        "_type" : "_doc",
        "_id" : "UVaep4sBBjIGApZi34_r", ## doc id
        "_score" : 1.0,
        "_source" : { ## fieldName:value
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : "999"
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : "999"
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "1000",
        "_score" : 1.0,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : "999"
        }
      }
    ]
  }
}
```


#### 條件查詢  

elasticsearch複雜的條件查詢推薦使用body   


可能較常用的   

關鍵字| 功能
:-:|:-:
terms | 不分詞 查詢詞綴需與資料庫index中完全相同
match| 會將查詢詞綴進行分詞 只要之中包含任一詞彙 就符合資格
match_phrase| 會將查詢詞綴進行分詞 匹配方式是依據array的index 相對位置做匹配, 強順序性
wildcard | 可使用通配符 查詢





match  
```
GET shopping/_search
{
	"query": {
		"match": {
			"title": "IPhone100 is a cell phone"
		}
	}
}
```
return
```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.13353139,
    "hits" : [
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "1000",
        "_score" : 0.13353139,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 999
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "aLG-qIsBnOvF5S7_MlFU",
        "_score" : 0.13353139,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 999
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "gLG-qIsBnOvF5S7_SlFE",
        "_score" : 0.13353139,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 2000
        }
      }
    ]
  }
}


## "IPhone100 is a cell phone" 會被拆成 IPhone100,is,a,cell,phone 去查詢 因此可以查找到IPhone100
```


match_phrase  
```
GET shopping/_search
{
	"query": {
		"term": {
			"title": "IPhone100 is a cell phone"
		}
	}
}
```
return
```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}


## 會需完全匹配 IPhone100 is a cell phone, 因此查不到 'IPhone100'
```



#### 多條件查詢

其實就是多一層bool, 類似 if 概念   , bool 下 { }  需同時滿足

	- must == AND

```

GET shopping/_search
{
  "query": {
    "bool": {
    
      "must": [
        {
          "match": {
            "category": "Apple"
          }
        },
        {
          "match": {
            "title": "IPhone100"
          }
        }


      ]
    }
  }
}


##  bool {must: [ 條件1,條件2 ....] }     ==    if (條件1 and 條件2)
```

	- should == or

```

GET shopping/_search
{
  "query": {
    "bool": {
    
      "should": [
        {
          "match": {
            "category": "Apple"
          }
        },
        {
          "match": {
            "title": "IPhone100"
          }
        }

      ]
      
    }
  }
}

##  bool {must: [ 條件1,條件2 ....] }     ==    if (條件1 or 條件2)
```

	- filer range ==  >= , <= , > , <

```

GET shopping/_search
{
  "query": {
    "bool": {

      "filter": {
	      "range": {
	      
		      "price": {
			      "gt": 1000
		      }
	      
	      }
      }
        
    }
  }
}

##  bool {filter: range: { field1 : { gt : <value> }  } }     ==    if field >　value
## gt: > , lt, < , gte >= , lte <=

```

   
範例 category=Apple  and title = iphone100 and price > 1000   

```
GET shopping/_search
{
  "query": {
    "bool": {
    
      "must": [
        {
          "match": {
            "category": "Apple"
          }
        },
        {
          "match": {
            "title": "IPhone100"
          }
        }
      ],

      "filter": {
	      "range": {
	      
		      "price": {
			      "gt": 1000
		      }
	      
	      }
      }


    }
  }
}

### ==    if (條件1 and 條件2) and (price > 1000)
```

### 指定特定資源

類似SQL , select A,B,C ... 不返回所有資源    

```
GET shopping/_search
{
	"_source": ["price"],
	"query": {
		"match_all":{}
	}
}
```
return  

```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "1000",
        "_score" : 1.0,
        "_source" : {
          "price" : 999
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "aLG-qIsBnOvF5S7_MlFU",
        "_score" : 1.0,
        "_source" : {
          "price" : 999
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "gLG-qIsBnOvF5S7_SlFE",
        "_score" : 1.0,
        "_source" : {
          "price" : 2000
        }
      }
    ]
  }
}

```


### 分頁

用 查詢某index下所有document 舉例  
```
GET shopping/_search
{
	"query": {
		"match_all": {}
	},
	"from": 0, 
	"size": 1, 
	"_source":["title","price"],
	"sort":{
		"price": "asc"
	}
}



#"from": 0, # 起始位置 假設查詢結果有100筆 從第0筆開始, 
#"size": 1,  # 顯示多少筆資料
#"_source":["title","price"] # 想顯示的field = select A,B..l.
#"sort":{"price": "asc" }   # 排序 asc or desc

```

from 頁數公式:  參考公式: (page-1)*size   
e.g.  第二頁  =  (2-1)*1  



return
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "UVaep4sBBjIGApZi34_r",
        "_score" : 1.0,
        "_source" : {
          "price" : "999",
          "title" : "IPhone100"
        }
      }
    ]
  }
}

```


### Highligt

將查詢結果特定field做highlight, 這邊設為藍色  

```
GET shopping/_search
{
	"query": {
		"match": {
			"category": "Apple"
		}
	},
	"highlight": {
	"fields": {
		"category": {}
		},
	"pre_tags": ["<span style='color: blue;'>"],
	"post_tags": ["</span>"]
	}
}
```

return  

```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.13353139,
    "hits" : [
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "1000",
        "_score" : 0.13353139,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 999
        },
        "highlight" : {
          "category" : [
            "<span style='color: blue;'>Apple</span>"
          ]
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "aLG-qIsBnOvF5S7_MlFU",
        "_score" : 0.13353139,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 999
        },
        "highlight" : {
          "category" : [
            "<span style='color: blue;'>Apple</span>"
          ]
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "gLG-qIsBnOvF5S7_SlFE",
        "_score" : 0.13353139,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 2000
        },
        "highlight" : {
          "category" : [
            "<span style='color: blue;'>Apple</span>"
          ]
        }
      }
    ]
  }
}
```




### 聚合函數

可能較常用的

關鍵字| 功能
:-:|:-:
terms | 分組 類似SQL groupby
avg| 計算平均
sum| 計算總和
min | 統計最小值
max| 統計最大值
histogram | 數值區間統計
stats|統計數值訊息,max,min,avg,sum,數值總數量
top_hits | index前N doc中最佳匹配



```
GET shopping/_search
{
  "aggs": {
    "task_name": { 
      "terms": {
        "field": "price"
      }
    }
  },
  "size":0
}


## task_name 可隨便自訂
## terms 表示功能 
## field 表示 函數作用範圍
## size:0 表示不顯示查詢結果 表示只顯示統計果
```


return

```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "task_name" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 999,
          "doc_count" : 2
        },
        {
          "key" : 2000,
          "doc_count" : 1
        }
      ]
    }
  }
}


## buckets 回傳類似SQL group_by的分組 分組key, 與該組數量


```


## Mapping

類似MySQL table schema, 只能創建或新增 不能修改  
### create

```
PUT user

PUT user/_mapping
{
	"properties": {

		"name": {
			"type": "text",
			"index": true,
			"analyzer": "ik_max_word",
			"search_analyzer": "ik_max_word"
			
		},
		"sex": {
			"type":"keyword",
			"index": true
		},
		"tel": {
			"type": "keyword",
			"index": false
		}

	}
}


## type: text  建立時 會被分詞
## type: keyword 建立時 不會被分詞
## index: false, 不會建立索引, 無法被作為條件query
## analyzer: 儲存時 使用的分析器
## search_analyzer: 查詢時 使用的分析器
```


### 測試範例

#### text 
```
PUT user/_doc/1000
{
	"name": "大毛",
	"sex": "男性",
	"tel": "3345678"
}
```


```
GET user/_search
{
	"query": {
		"match": {
			"name": "毛"
		}
	}
}
```

return  
```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1000",
        "_score" : 0.2876821,
        "_source" : {
          "name" : "大毛",
          "sex" : "男性",
          "tel" : "3345678"
        }
      }
    ]
  }
}
# 用 "毛"查找 "大毛" 可以被找到 , 表示大毛 儲存時有被分詞
```

#### keyword
```
GET user/_doc/1000
{
	"query": {
		"match": {
			"sex": "男"
		}
	}
}
```

return
```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}

## 用"男"無法查找"男性" 表示 儲存時 沒有被拆分
```

#### index: false  

```
GET user/_doc/1000
{
	"query": {
		"match": {
			"tel": "3345678"
		}
	}
}
```

```
{
  "error" : {
    "root_cause" : [
      {
        "type" : "query_shard_exception",
        "reason" : "failed to create query: Cannot search on field [tel] since it is not indexed.",
        "index_uuid" : "CPvT5dd4RpekoekLL8Ne2w",
        "index" : "user"
      }
    ],
    "type" : "search_phase_execution_exception",
    "reason" : "all shards failed",
    "phase" : "query",
    "grouped" : true,
    "failed_shards" : [
      {
        "shard" : 0,
        "index" : "user",
        "node" : "HP2s1sefTHSahOSTYc0EjA",
        "reason" : {
          "type" : "query_shard_exception",
          "reason" : "failed to create query: Cannot search on field [tel] since it is not indexed.",
          "index_uuid" : "CPvT5dd4RpekoekLL8Ne2w",
          "index" : "user",
          "caused_by" : {
            "type" : "illegal_argument_exception",
            "reason" : "Cannot search on field [tel] since it is not indexed."
          }
        }
      }
    ]
  },
  "status" : 400
}

# tel 儲存時 沒有被建立index, 因此不支援被query
```




## score機制

評分方式基於 TF-IDF公式,   
簡單說 會統計該term在這個doc裡出現次數, 愈多相關性愈高, 但若在所有文檔中出現愈多次反而愈不重要  

一般查詢  
```
GET shopping/_search?
{
	"query": {
		"match": {
			"title": "IPhone100 is a cell phone"
		}
	}
}

```
return   
```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.13353139, 
    "hits" : [
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "1000",
        "_score" : 0.13353139,  ##<<<< 查詢score
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 999
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "aLG-qIsBnOvF5S7_MlFU",
        "_score" : 0.13353139,  ##<<<<
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 999
        }
      },
      {
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "gLG-qIsBnOvF5S7_SlFE",
        "_score" : 0.13353139,  ##<<<<
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 2000
        }
      }
    ]
  }
}

```




使用explain=true 可以知道查詢的評分權重
```
GET shopping/_search?explain=true
{
	"query": {
		"match": {
			"title": "IPhone100 is a cell phone"
		}
	}
}

```

return

```
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.13353139,
    "hits" : [
      {
        "_shard" : "[shopping][0]",
        "_node" : "HP2s1sefTHSahOSTYc0EjA",
        "_index" : "shopping",
        "_type" : "_doc",
        "_id" : "1000",
        "_score" : 0.13353139,
        "_source" : {
          "title" : "IPhone100",
          "category" : "Apple",
          "price" : 999
        },
        "_explanation" : {
          "value" : 0.13353139,
          "description" : "sum of:",
          "details" : [
            {
              "value" : 0.13353139,
              "description" : "weight(title:iphone100 in 0) [PerFieldSimilarity], result of:",  
              "details" : [
                {
                  "value" : 0.13353139,
                  "description" : "score(freq=1.0), computed as boost * idf * tf from:", #### 計算方式
                  "details" : [
                    {
                      "value" : 2.2,
                      "description" : "boost",   ## boost
                      "details" : [ ]
                    },
                    {
                      "value" : 0.13353139,
                      "description" : "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5)) from:",  ######################## idf
                      "details" : [
                        {
                          "value" : 3,
                          "description" : "n, number of documents containing term", 
                          "details" : [ ]
                        },  
                        {
                          "value" : 3,
                          "description" : "N, total number of documents with field", 
                          "details" : [ ]
                        }
                      ]
                    },
                    {
                      "value" : 0.45454544,
                      "description" : "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl)) from:",  ############### tf
                      "details" : [
                        {
                          "value" : 1.0,
                          "description" : "freq, occurrences of term within document",
                          "details" : [ ]
                        },
                        {
                          "value" : 1.2,
                          "description" : "k1, term saturation parameter",
                          "details" : [ ]
                        },
                        {
                          "value" : 0.75,
                          "description" : "b, length normalization parameter",
                          "details" : [ ]
                        },
                        {
                          "value" : 1.0,
                          "description" : "dl, length of field",
                          "details" : [ ]
                        },
                        {
                          "value" : 1.0,
                          "description" : "avgdl, average length of field",
                          "details" : [ ]
                        }
                      ]
                    }
                  ]
                }
              ]
            }
          ]
        }


### ... 只保留一組
```
