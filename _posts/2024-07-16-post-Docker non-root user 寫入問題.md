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

```bash
id -u
# 1000
id -g
# 1000
```

這邊dockerfile 使用非root user,  也可配合ARG使用  
有時候會有需要root的場景, 加入sudo without password

```Dockerfile
FROM ubuntu:22.04

RUN apt-get update -y
RUN apt-get install build-essential procps curl file git -y
RUN apt-get install -y gnupg software-properties-common
RUN apt-get install -y wget
RUN wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
RUN gpg --no-default-keyring \
  --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
  --fingerprint
RUN echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  tee /etc/apt/sources.list.d/hashicorp.list

RUN apt update
RUN apt-get install terraform -y
RUN apt-get install -y fish
RUN apt-get install vim -y && apt-get install -y init
RUN apt-get install sudo -y
RUN wget https://github.com/gruntwork-io/terragrunt/releases/download/v0.69.0/terragrunt_linux_amd64
RUN mv terragrunt_linux_amd64 /usr/bin/terragrunt
RUN chmod 775 /usr/bin/terragrunt
RUN groupadd -g 1000 hccuse
RUN useradd -m -u 1000 -g hccuse -d /home/hccuse -s /usr/bin/fish hccuse && usermod -aG sudo hccuse
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> \
  /etc/sudoers.d/nopasswd
USER hccuse
WORKDIR /home/hccuse/
```

docker compose.yaml

```yaml
services:
  terraform:
    image: terraform_play_ground
    container_name: tenv
    restart: always
    user: hccuse
    build:
      context: .
      dockerfile: ./Dockerfile
    entrypoint: sleep 999999
```
