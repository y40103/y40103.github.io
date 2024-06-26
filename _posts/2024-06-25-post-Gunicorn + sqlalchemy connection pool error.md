---
title: "[Issue紀錄] Gunicorn + Sqlalchemy connection pool error"
categories:
  - Issue紀錄
tags:
  - Python
toc: true
toc_label: Index
mermaid: true
---

casbin相依sqlalchemy, API server啟動一段時間後, authorization會有機會返回500, 但非每次都會     
查看log, 發現是sqlalchemy產生的error    

```
sqlalchemy.exc.DatabaseError: (psycopg2.DatabaseError) error with status PGRES_TUPLES_OK and no message from the libpq
```
查看當前連接數, 發現很正常  
google後, 有發生類似情況的都是在異步或是有些人調用threads/process  
因此應該是類似 race condition的情況  

## 參考解法  

### 關閉 connection pool

後來發現若將 connection pool的功能關閉, 可以避免這個情況  

```python
engine = create_engine(url, connect_args={'options': '-csearch_path={}'.format(schema)},
                            poolclass=NullPool)
```
算是一個犧牲性能的消極做法  


### preload

不太想犧牲性能, 又仔細研究一下機制  
發現跟web server有一些關係  
該專案是使用 gunicorn  
gunicorn 啟動時會調用多個uvicorn作為worker  
每個worker都是使用獨立的pool, 理論上應該不太會有共用的問題  
該專案 gunicorn 啟動方式  
```bash
/usr/local/bin/gunicorn __init__:app --reload --preload --workers 8 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000 --keep-alive 65
```
關鍵在於 --preload, 這樣在啟動worker時 會從當前的主worker fork出去  
所有worker會使用主worker的記憶體,只有遇到需要寫入時,才會copy一份記憶體出去 原美意是可以節省資源  
但fork出去的worker從pool調用connection, worker間會有機會產生衝突  
取connection時,  
有機會會拿到正在使用的connection     
或是使用到已經被丟棄的connection  
因此使用--preload, 會需要確定業務場景會不會有race condition的狀況  
取消後, 就可以正常使用 connection pool

```bash
/usr/local/bin/gunicorn __init__:app --reload --workers 8 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000 --keep-alive 65
```

```python
engine = create_engine(url, connect_args={'options': '-csearch_path={}'.format(schema)},
                       pool_size=5,
                       poolclass=QueuePool,
                       max_overflow=10,
                       pool_recycle=3600)
```

