---
title: "[筆記] FastAPI & Asyncio"
categories:
  - 筆記
tags:
  - AWS
toc: true
toc_label: Index
mermaid: true
---

## FastAPI & Asyncio

目前實務上支援異步的套件其實真的不多...   
比較實用的目前覺得有 asyncpg 與 httpx    
python 很常看到有人在討論異步 講得好像可以加速N倍, 但實際上只有IO密集才會有顯著效果    
而且支援的套件又很少 有大量IO/套件支援/性能敏感 的情況下才會想用  

近期剛好遇到 fastapi需要大量調用特定api, 這邊記錄一下用法  

fastapi 若handler使用async 啟動後server後 同asyncio function 進 run() ,

這邊舉一個一般asyncio的 httpx範例(sync & async) 與 一個Fastapi的 httpx範例(sync and async) ,   

兩個範例都是 request自己的api, 該api sleep 3秒, 各請求五次, 紀錄同步與異步的時間  


## Playground

有時間會補充asyncpg的範例, 這邊順便啟動一個db, 雖然目前沒用到  

docker-compose.yaml

```
services:
  api:
    restart: always
    image: tiangolo/uvicorn-gunicorn-fastapi:python3.11
    container_name: aio
    user: root
    environment:
      POSTGRES_USERNAME: ${POSTGRES_USERNAME}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - 8000:80
    volumes:
      - ./app:/app
      - /etc/timezone:/etc/timezone
      - /etc/localtime:/etc/localtime
    networks:
      aio_net:
        ipv4_address: 124.26.0.2

  PGdb:
    image: pg
    build:
      context: .
      dockerfile: DockerfilePG
    container_name: aiodb
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USERNAME}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata # 更換預設 pg 儲存資料預設
      PGPORT: 5432
      POSTGRES_DB: prefect
      USER: $(id -u):$(id -g)
      TZ: 'Asia/Taipei'
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone
      - ./pg_data/:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      aio_net:
        ipv4_address: 124.26.0.3


networks:
  aio_net:
    name: "aio_net"
    driver: bridge
    ipam:
      config:
        - subnet: 124.26.0.0/16
          gateway: 124.26.0.1
```

.env

```
POSTGRES_USERNAME="dev123"
POSTGRES_PASSWORD="dev123"
```

## 啟動測試環境  

```bash
docker compose up -d
```



## Asyncio  async + sync + httpx

async_and_sync.py

```python
import datetime

import httpx
import asyncio
from typing import List


def test_sync(url: str):
    r = httpx.get(url, timeout=20)
    return r.status_code


async def test_async(url: str):
    async with httpx.AsyncClient() as client:
        r = await client.get(url, timeout=20)
        return r.status_code


async def async_pull_all(urls: List[str]):
    return await asyncio.gather(*[test_async(url) for url in urls])


if __name__ == "__main__":

    app_url = "http://localhost:8000/items/%s"
    query_params = ["1", "2", "3", "4", "5"]

    print("async...")
    async_start = datetime.datetime.now()

    result = asyncio.run(async_pull_all(app_url % params for params in query_params))

    async_finish = datetime.datetime.now()

    print(result)

    print(f"async cost {async_finish - async_start}")

    print("sync...")
    sync_start = datetime.datetime.now()
    for param in query_params:
        url = app_url % param
        test_sync(url)

    sync_finish = datetime.datetime.now()

    print(f"sync cost {sync_finish - sync_start}")
```

stdout  

```
(.venv) hccuse@R075 ~/d/a/test2> python async_and_sync.py
async...
[200, 200, 200, 200, 200]
async cost 0:00:03.236654
sync...
sync cost 0:00:15.258665
```


## Fastapi async + sync + httpx

這邊會打 /get_all 這支api 效果與上面的 async_and_sync.py 相同  

/app/app.py

```python
import asyncio
import time

from fastapi import FastAPI
import httpx
from typing import List
import datetime

app = FastAPI()


def test_sync(url: str):
    r = httpx.get(url, timeout=20)
    return r.status_code


async def test_async(url: str):
    async with httpx.AsyncClient() as client:
        r = await client.get(url, timeout=20)
        return r.status_code


async def async_pull_all(urls: List[str]):
    return await asyncio.gather(*[test_async(url) for url in urls])


@app.get("/")
def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str = None):
    time.sleep(3)
    return {"item_id": item_id, "q": q}


@app.get("/all_items")
async def get_all():
    app_url = "http://api/items/%s"
    query_params = ["1", "2", "3", "4", "5"]
    urls = [app_url % p for p in query_params]
    print("async...")
    async_start = datetime.datetime.now()

    result = await asyncio.gather(*[test_async(url) for url in urls])

    async_finish = datetime.datetime.now()

    print(result)

    print(f"async cost {async_finish - async_start}")

    print("sync...")
    sync_start = datetime.datetime.now()
    for param in query_params:
        url = app_url % param
        test_sync(url)

    sync_finish = datetime.datetime.now()

    print(f"sync cost {sync_finish - sync_start}")
```


stdout
```
(.venv) hccuse@R075 ~/d/a/test2> docker logs aio
124.26.0.2:47004 - "GET /items/2 HTTP/1.1" 200
124.26.0.2:47036 - "GET /items/4 HTTP/1.1" 200
124.26.0.2:47050 - "GET /items/3 HTTP/1.1" 200
124.26.0.2:47020 - "GET /items/1 HTTP/1.1" 200
124.26.0.2:47054 - "GET /items/5 HTTP/1.1" 200
[200, 200, 200, 200, 200]
async cost 0:00:06.106267
sync...
124.26.0.2:47062 - "GET /items/1 HTTP/1.1" 200
124.26.0.2:53922 - "GET /items/2 HTTP/1.1" 200
124.26.0.2:53930 - "GET /items/3 HTTP/1.1" 200
124.26.0.2:53940 - "GET /items/4 HTTP/1.1" 200
124.26.0.2:39746 - "GET /items/5 HTTP/1.1" 200
sync cost 0:00:15.149704
```

純asyncio似乎比fastapi的異步 效果更好一些, 但無論哪個跟sync相比都是有顯著的提升  

