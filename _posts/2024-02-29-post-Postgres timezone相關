---
title: "[筆記] Postgres timezone相關"
categories:
  - 筆記
tags:
  - Postgres
toc: true
toc_label: Index
mermaid: true
---

# Postgres timezone相關 

## 基本檢視語法  

檢視系統當前預設時區  
```
show timezone;
```

檢視postgres支援時區  
```
select * from pg_timezone_names;
```


Postgres 預設使用UTC時間, 


## 容器預設時區  

容器中若要使用其他時區, 可以透過環境變數TZ來設定, 例如:
```
docker run -it -e "TZ=GMT+2" postgres:alpine
```
或是
```yaml
version: '3.7'
services:
  PGdb:
    image: pg
    build:
      context: .
      dockerfile: DockerfilePG
    container_name: db-pg
    restart: always
    environment:
      POSTGRES_PASSWORD: example 
      PGDATA: /var/lib/postgresql/data/pgdata # 更換預設 pg 儲存資料預設
      PGPORT: 5432
      POSTGRES_DB: prefect
      USER: $(id -u):$(id -g)
      TZ: 'Asia/Taipei'
    volumes:
      - ./pg_data/:/var/lib/postgresql/data
    profiles:
      - main
      - all
    ports:
      - "5432:5432"
```

### 其它注意事項  

這邊有個坑, 若一開始沒注意, 已經開始使用UTC預設的資料庫  
此時新增TZ變數, 重新啟動, 會發現資料庫時區不會隨著變數改變  
因 postgres啟動時, 若發現資料庫已經存在資料, 則會跳過初始化的步驟, 因此會沿用已存在的資料庫資料的設定    

```
hccuse@R075 ~ [1]> docker logs -f db-pg

PostgreSQL Database directory appears to contain a database; Skipping initialization

2024-02-29 06:52:16.400 UTC [1] LOG:  starting PostgreSQL 15.1 (Debian 15.1-1.pgdg110+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
2024-02-29 06:52:16.459 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2024-02-29 06:52:16.459 UTC [1] LOG:  listening on IPv6 address "::", port 5432
```

因此, 若要更改時區, 可以選擇清空資料庫, 重啟容器, 或是透過指令更改資料庫時區  

或是直接使用SQL指令更改資料庫時區 


## SQL時區修改相關 

### 修改單一資料庫時區

若已經使用預設時區建立資料庫, 可以參考以下指令,  
demo為想修改的資料庫名稱, asia/Taipei為想設定的時區   

```
ALTER DATABASE demo SET timezone TO 'asia/Taipei';
```
 

### 修改預設時區  

修改預設時區  , 需注意此方式不會影響已經存在的資料庫, 只會影響新建立的資料庫  
若要修改已經存在的資料庫, 需使用上述的指令  
```
set timezone TO 'asia/Taipei';
```


