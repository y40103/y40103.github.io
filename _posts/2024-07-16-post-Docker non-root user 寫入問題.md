---
title: "[隨筆] Docker non-root user 寫入問題"
categories:
  - 隨筆
tags:
  - Docker
toc: true
toc_label: Index
mermaid: true
---

DockerFile 使用非root user時  
若執行 container 有volume目錄進去   
會有container內部user無法寫入該volume目錄的問題   
會有這問題是因為 內部user的 uid 與 gid 與 host主機不一致  
而volume進去的目錄是屬於host user的權限  
解決方式就是把內外user的uid gid一致 (user名稱不一樣沒關係)  


### 參考範例

這邊dockerfile 使用非root user, 
```Dockerfile
# FROM python:3.11-alpine
FROM pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime

RUN pip install \
    azure-cosmos \
    pandas \
    pydantic \
    pydantic-settings \
    scikit-learn \
    matplotlib \
    sqlalchemy \
    psycopg2-binary \
    aiohttp \
    asyncpg \
    click


RUN apt update -y && apt --no-install-recommends install -y curl && rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*

ENV SUPERCRONIC_URL=https://github.com/aptible/supercronic/releases/download/v0.2.27/supercronic-linux-amd64 \
    SUPERCRONIC=supercronic-linux-amd64
RUN curl -fsSLO "$SUPERCRONIC_URL" \
 && chmod +x "$SUPERCRONIC" \
 && mv "$SUPERCRONIC" "/usr/local/bin/${SUPERCRONIC}" \
 && ln -s "/usr/local/bin/${SUPERCRONIC}" /usr/local/bin/supercronic


RUN useradd -m -d /home/docker -s /bin/bash docker
USER docker  #非root user

RUN <<EOF cat > /tmp/cronjob.conf
*/10 * * * * /opt/conda/bin/python /app/main.py -j predict_job
EOF

RUN <<EOF cat > /tmp/cron_run.sh
#! /usr/bin/bash
sleep 5
/opt/conda/bin/python /app/main.py -j predict_job
sleep 60
/usr/local/bin/supercronic /tmp/cronjob.conf
EOF

RUN chmod a+x /tmp/cron_run.sh

WORKDIR /app
```

```bash
export user_uid=$(id -u)
export user_gid=$(id -g)
```


docker compose.yaml
```yaml
services:
  predictor:
    user: ${user_uid}:${user_gid}  # 這邊可以指定 container user 的 gid uid
    build:
      dockerfile: Dockerfile
      context: ./
    working_dir: /app

# ...
```


