---
title: "[筆記] Postgres 備份與還原"
categories:
  - 筆記
tags:
  - Postgres
toc: true
toc_label: Index
---

pg 備份還原script紀錄  

## 參考指令


### 特定資料庫dump

```bash
pg_dump -U $POSTGRES_USERNAME -h $PGHOST -Fc $POSTGRES_DB > /tmp/pg_dump/pg_$(date +%Y%m%d%H%M).sql

-h dbhost

-U username

-d dbname

-Fc 備份 輸出特定格式

```

### 特定資料庫restore

```bash
pg_restore -h $POSTGRES_DB -U postgres -d $$POSTGRES_DB -a  -Fc /tmp/pg_dump/pg_dump_latest_backup.sql

-h dbhost

-a data only, no schema, ## 配合 go migrate 使用, 只需dump數值  

-U username

-d dbname

-Fc 備份 輸出特定格式

```



### No Prompt 技巧

dump 與 restore 的時候 可以直接用文件(~/.pgpass)跳過密碼輸入, 權限需600

```bash

echo PGdb:$PGPORT:$POSTGRES_DB:$POSTGRES_USERNAME:$POSTGRES_PASSWORD > ~/.pgpass
## 新增 .pgpass

chmod 600 ~/.pgpass
## 修改權限
```



## 實際應用


### dump 至 S3 

```bash
#!/usr/bin/bash
echo "$(date +%Y%m%d%H%M) start pg dump" >> /src/logs/pg_backup.log
echo PGdb:$PGPORT:$POSTGRES_DB:$POSTGRES_USERNAME:$POSTGRES_PASSWORD > /root/.pgpass
chmod 600 ~/.pgpass
if [ ! -d "/tmp/pg_dump/" ]; then
  mkdir -p /tmp/pg_dump
fi
pg_dump -U $POSTGRES_USERNAME -h PGdb -FC $POSTGRES_DB > /tmp/pg_dump/pg_$(date +%Y%m%d%H%M).sql
if [ $? -ne 0 ]; then
  echo "$(date +%Y%m%d%H%M) pg dump fail" >> /src/logs/pg_backup.log
  exit 1
fi
echo "$(date +%Y%m%d%H%M) pg dump success,  upload s3 start" >> /src/logs/pg_backup.log
aws s3 cp /tmp/pg_dump/pg_$(date +%Y%m%d%H%M).sql s3://etc-s3-bucket/pg_dump/pg_dump.sql >> /src/logs/pg_backup.log 2>&1
if [ $? -ne 0 ]; then
  echo "$(date +%Y%m%d%H%M) upload to s3 fail" >> /src/logs/pg_backup.log
  exit 1
fi
rm /tmp/pg_dump/pg_$(date +%Y%m%d%H%M).sql
echo "$(date +%Y%m%d%H%M) upload to s3 success" >> /src/logs/pg_backup.log
```


### S3 restore 

```bash
#!/usr/bin/bash
echo "$(date +%Y%m%d%H%M) start pg dump" >> /src/logs/pg_backup.log
echo PGdb:$PGPORT:$POSTGRES_DB:$POSTGRES_USERNAME:$POSTGRES_PASSWORD > /root/.pgpass
chmod 600 ~/.pgpass
if [ ! -d "/tmp/pg_dump/" ]; then
  mkdir -p /tmp/pg_dump
fi
echo "$(date +%Y%m%d%H%M) start to download form s3" >> /src/logs/pg_backup.log
aws s3 cp s3://etc-s3-bucket/pg_dump/pg_dump.sql /tmp/pg_dump/pg_dump_latest_backup.sql >> /src/logs/pg_backup.log 2>&1
if [ $? -ne 0 ]; then
  echo "$(date +%Y%m%d%H%M) fail to download from s3" >> /src/logs/pg_backup.log
  exit 1
fi
echo "$(date +%Y%m%d%H%M) download from s3 success, start to restore postgres " >> /src/logs/pg_backup.log
pg_restore -h PGdb -U $POSTGRES_USERNAME -d $POSTGRES_DB -a  -Fc /tmp/pg_dump/pg_dump_latest_backup.sql >> /src/logs/pg_backup.log 2>&1
rm  /tmp/pg_dump/pg_dump_latest_backup.sql
echo "$(date +%Y%m%d%H%M) pg restore success" >> /src/logs/pg_backup.log
```


