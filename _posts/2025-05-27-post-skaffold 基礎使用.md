---
title: "[筆記] skaffold 基礎使用"
categories:
  - 筆記
tags:
  - skaffold
  - kubernetes
toc: true
toc_label: Index
---

skaffold 基本上是 build + push + deploy + port-forward 的自動化工具,
目的是讓容器工具開發時, 可以自動化這些操作, 如你修改code or config
一個指令幫你重build image, push, deploy ... 甚至可以幫你監聽你的目錄 有操作就自動再跑一次流程

deploy可以跟很多工具配合 docker, kubectl, helm ...

## Quickstart

```bash
.
├── helm_demo
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   │   └── main_api.yaml
│   └── values.yaml
├── main_api
│   ├── demo
│   │   ├── application
│   │   ├── domain
│   │   ├── go.mod
│   │   ├── go.sum
│   │   ├── infrastructure
│   │   ├── interface
│   │   ├── migrate
│   │   └── readme.md
│   └── Dockerfile
└── skaffold.yaml
```

skaffold.yaml

```yaml
apiVersion: skaffold/v4beta13
kind: Config
metadata:
  name: demo
build:
  tagPolicy:
    gitCommit: {}
  artifacts:
    - image: OOO.dkr.ecr.ap-northeast-1.amazonaws.com/main_api
      context: ./main_api
      docker:
        dockerfile: Dockerfile
  local:
    push: true

deploy:
  helm:
    releases:
      - name: demo
        chartPath: ./helm_demo/
        createNamespace: true

portForward:
  - resourceType: service
    resourceName: main-api
    port: 8090 # Service port
    localPort: 8090 # 轉發到本地機器 port , 之後打localhost:localPort 可以打到serivce
```

```bash
skaffold run
# 會執行 build + push + deploy + port-forward
```

## 基礎指令

基本上本質就是跑 build + push + deploy + port-forward 流程, 實際細節是看設定檔  
若要執行的流程 該設定檔沒有 就會省略該步驟

```bash
skaffold build -v debug
# image + push

skaffold run 
# image + push + deploy + port-forward

skaffold dev
# image + push + deploy + port-forward 
# 並且會持續監聽相關目錄, 有異動會再次觸發流程, 結束監聽後, 會把deploy uninstall
```

## Push False 坑

一般容器工具設定檔 假設沒有指定tag, registry上有latest, 通常會直接pull latest

skaffold 通常是分佈式設計 不會考慮local image,

假設push local, build 後 它deploy時 無論local有沒有 它都會去registry拉剛剛build的image tag  
`因此若要dpeloy, push必須開啟`, 因為tag是動態的, 即使知道, 也不可能每次都去改容器設定檔  

`實際上這邊設計應該把 push:true 與 deploy 綁一起, 假設有使用deploy, push:false 應該要報錯比較好`
