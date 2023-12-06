---
title: "ElasticSearch Analyzer"
categories:
  - ElasticSearch
tags:
  - Analyzer
toc: true
toc_label: Index
---


分析器隨筆

<script src="https://unpkg.com/mermaid@8.0.0/dist/mermaid.min.js"></script>

<div class="mermaid">
flowchart LR
A[text] --> F[character filter]
subgraph Analyzer
F --> G[tokenizer]
G --> H[token filter]
end
H --> J[Index]
</div>




## 組成

1. character filter: 主要做篩選與過濾 e.g. 將HTML字元濾掉  
2. tokenizer: 主要將字串進行分詞 e.g. "It's a cool guy" >> "it's" "a" "cool" "guy"  
3. token filter:  可將拆出的分詞進行 增加,刪除,修改 e.g. 大小寫轉換  


## 觸發時機

1. 查詢時
2. 寫入時




## 使用分析器


```
PUT user
## 範例index
```

### Index 特定field

```
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
```


### Index 全局預設

```

POST /user/_close

PUT user/_settings
{
    "analysis": {
      "analyzer": {
        "default": {
          "type": "ik_max_word"
        },
        "default_search":{
	        "type": "ik_max_word"
        }
      }
    }
}

POST /user/_open
```


###  檢視Index當前設定值

```
GET /user/_settings
```

return
```
{
  "user" : {
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
        "provided_name" : "user",
        "creation_date" : "1699429776184",
        "analysis" : {
          "analyzer" : {
            "default_search" : {
              "type" : "ik_max_word"
            },
            "default" : {
              "type" : "ik_max_word"
            }
          }
        },
        "number_of_replicas" : "1",
        "uuid" : "81Jc8eyxSN6uYqQskE7YFg",
        "version" : {
          "created" : "7170699"
        }
      }
    }
  }
}
```




### 自訂分析器

自訂分析器, 先將html符號濾掉 再用分析器拆分 最後被用大寫輸出  

```
PUT test_index
{
   "settings":{
      "analysis":{
         "analyzer":{
            "my_analyzer":{
               "type":"custom",
               "char_filter":[
                  "html_strip"
               ],
               "tokenizer":"standard",
               "filter":[
                  "uppercase"
               ]
            }
         }
      }
   },
   "mappings":{
      "properties":{
         "my_text":{
            "type":"text",
            "analyzer":"my_analyzer",
            "search_analyzer":"my_analyzer"
         }
      }
   }
}

# character filer 使用 html_strip 去除html
# tokenizer 使用 standard 拆分
# tokenfilter 使用 lowercase 處理輸出分詞

```

其他內建標準 [tokenizer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-tokenizers.html) , [token_filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-tokenfilters.html)[character filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-charfilters.html)


測試自訂分析器
```
POST _analyze
{
	 "char_filter":[
	 	"html_strip"
	 ],
	 "tokenizer": "standard",
	  "filter": [
		"uppercase"
	  ],
	  "text": "<title>Hello World</title>"
}
```

or


```
POST test_index/_analyze
{
  "analyzer": "my_analyzer",
  "text": "<title>Hello World</title>"
}
```


return  

```
{
  "tokens" : [
    {
      "token" : "HELLO",
      "start_offset" : 7,
      "end_offset" : 12,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "WORLD",
      "start_offset" : 13,
      "end_offset" : 18,
      "type" : "<ALPHANUM>",
      "position" : 1
    }
  ]
}

```

