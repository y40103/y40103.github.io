---
title: "[筆記] AWS ECR Credential"
categories:
  - 筆記
tags:
  - AWS
  - Docker
toc: true
toc_label: Index
---


AWS ECR 沒有類似使用永久的credential
而是採用有時效性的方法

在設置好aws credentail的情況下, 使用aws cli 可以取得有時效性的key

## docker

```bash
aws ecr get-login-password --region ap-northeast-1
# 在stdout 輸出 key
```

```bash
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 528OOOOOOOO.dkr.ecr.ap-xxxxxxx.amazonaws.com  
# 直接用管道登錄
```

但每次pull image都需要執行一次 docker login  

無法利用一般的 ~/.docker/config 設置credentail 直接省略docker login  

這邊可以利用工具讓其自動通過驗證, 省略docker login

```bash

sudo apt-get install amazon-ecr-credential-helper

```

vim ~/.docker/config.json

```bash
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "eTQwMTAzOmEzNOOOOOOOOO"
    }
  },
  "credsStore": "ecr-login"
}

```

設置後 就不必每次都重新docker login
