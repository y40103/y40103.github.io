---
title: "[隨筆] AWS ALB 偶爾出現 502 Bad Gateway"
categories:
  - 隨筆
tags:
  - AWS
toc: true
toc_label: Index
mermaid: true
---

從AWS ALB作為入口訪問API時, 偶爾會返回502,  
查詢後發現是ALB的連線機制, 從ALB進行請求, ALB 會keepalive  
保持練線一段時間, 後端伺服器也會有一個 keepalive的 timeout時間  
通常會有這個情況是 後端伺服器的timeout 小於 ALB的keepalive時間  


### 參考解法

AWS ALB的預設 keepalive時間為 60s, 只要將後端伺服器的keepalive時間大於60s

這邊是同時調整nginx 與 gunicorn  

/etc/nginx/nginx.conf
```
http {

        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;


        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        keepalive_timeout  300s 300s;
        keepalive_requests 10000;
....

```

gunicorn 啟動arg

```bash
/usr/local/bin/gunicorn __init__:app --reload --workers 8 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000 --keep-alive 65
```


