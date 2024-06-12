---
title: "[筆記] Prefect Kubernetes-Worker"
categories:
  - 筆記
tags:
  - Kubernetes
  - Prefect
toc: true
toc_label: Index
mermaid: true
---

優點為 worker為由 scheduler調度, 不用自己管理運算資源  

```mermaid
graph LR
    subgraph K8s-Deployment1
        Prefect-Server
    end

    deploy --> Prefect-Server

    subgraph K8sDeployment2
        Prefect-Worker1
        Prefect-Worker2
    end
    Prefect-Server -.-|publish| k8sJob-flow-run1
    Prefect-Server -.-|publish| k8sJob-flow-run2 -->|Cron Trigger| Prefect-Worker1
    Prefect-Server -.-|publish| k8sJob-flow-runN...
```

### Prefect-Server

server負責UI & 發布flow-run(可以理解成 待辦事項)   
時間到後, worker會讀取flow-run, 完成這些事項    

### Kubernetes-worker

啟動時,會建立一個worker pool, 裡面至少有一個worker  
須注意worker pool中的worker會使用同一個image   
這邊建議可以先使用預設值, 後面在推送任務時, 有機會可以覆蓋設定  

```
command: ["prefect", "worker","start", "--pool","worker-pool","--type","kubernetes"]
```
需指定的話, 可設定  `base job template`, 若不設定會使用預設image e.g. `docker.io/prefecthq/prefect:2-latest`  

以下yaml來自Web UI , 可參考 worker-pool edit   

```yaml
{
  "variables": {
    "type": "object",
    "properties": {
      "env": {
        "type": "object",
        "title": "Environment Variables",
        "description": "Environment variables to set when starting a flow run.",
        "additionalProperties": {
          "type": "string"
        }
      },
      "name": {
        "type": "string",
        "title": "Name",
        "description": "Name given to infrastructure created by a worker."
      },
      "image": {
        "type": "string",
        "title": "Image",
        "example": "docker.io/prefecthq/prefect:2-latest",
        "description": "The image reference of a container image to use for created jobs. If not set, the latest Prefect image will be used."
      }, ..............略

  ## 主要設定部分
  "job_configuration": {
    "env": "{{ env }}",
    "name": "{{ name }}",
    "labels": "{{ labels }}",
    "command": "{{ command }}",
    "namespace": "{{ namespace }}",
    "job_manifest": {
      "kind": "Job",
      "spec": {
        "template": {
          "spec": {
            "containers": [
              {
                "env": "{{ env }}",
                "args": "{{ command }}",
                "name": "prefect-job",
                "image": "{{ image }}",
                "imagePullPolicy": "{{ image_pull_policy }}"
              }
            ],
            "completions": 1,
            "parallelism": 1,
            "restartPolicy": "Never",
            "serviceAccountName": "{{ service_account_name }}"
          }
        },
        "backoffLimit": 0,
        "ttlSecondsAfterFinished": "{{ finished_job_ttl }}"
      },
      "metadata": {
        "labels": "{{ labels }}",
        "namespace": "{{ namespace }}",
        "generateName": "{{ name }}-"
      },
      "apiVersion": "batch/v1"
    },
    "stream_output": "{{ stream_output }}",
    "cluster_config": "{{ cluster_config }}",
    "job_watch_timeout_seconds": "{{ job_watch_timeout_seconds }}",
    "pod_watch_timeout_seconds": "{{ pod_watch_timeout_seconds }}"
  }
}
```
### deploy

佈署任務, 這邊會需要flow的程式目錄 與 deploy的yaml(prefect.yaml)   
概念上就是 程式與yaml file 可以發布 一組排程(prefect-deployment)給 prefect-server,  
他之後就會定期創建 k8s-job (prefect-flow-run)   
flow-run時間到被trigger後, 對應的kubernetes-worker 會解析flow-run的工作設定, 開始完成  
若有重複設定 會以flow-run的為優先覆蓋  
flow-run設定包含image,jobname ....  

prefect.yaml  job_variables的內容就會覆蓋worker-poo原本的設定    
```yaml
deployments:
- name: demo
  version: 1.0
  tags: ["test"]
  description: "demo"
  schedule:
    cron: "* * * * *"
    timezone: Asia/Taipei
  entrypoint: ./prefect_demo.py:repo_info
  parameters: {}
  work_pool:
    name: worker-pool
    work_queue_name: default
    job_variables:
      service_account_name: "worker"
      name: "testset"
      namespace: "dev"
      image: "docker.io/library/prefect_worker:1.0"
      image_pull_policy: IfNotPresent
      finished_job_ttl: 300
```
這邊的image很重要, 需要打包執行程式  , 否則worker會沒有檔案可以執行   



### prefect worker config

這邊就是啟動worker. 之後推送任務   
使server可以開始排程    

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prefect-worker
  namespace: dev
spec:
  selector:
    matchLabels:
      app: prefect-worker
  replicas: 1
  template: # pod template
    metadata:
      labels:
        app: prefect-worker
    spec:
      serviceAccountName: worker
      containers:
        - name: prefect-worker
          image: prefect_worker:1.0
          command: [ "/opt/prefect/entrypoint.sh", "prefect", "worker","start", "--pool","worker-pool","--type","kubernetes" ]
          imagePullPolicy: IfNotPresent
          env:
            - name: PREFECT_API_URL
              value: "http://svc-prefect.dev.svc.cluster.local:4200/api"
          lifecycle:
            postStart:
              exec: # 在main container 啟動時 執行命令
                command: ["/bin/sh", "-c", "sleep 10; prefect --no-prompt deploy --all"]
```



### 測試用例


```
etl
├── prefect.yaml
└── prefect_demo.py
```

prefect.yaml
```yaml
deployments:
- name: demo
  version: 1.0
  tags: ["test"]
  description: "demo"
  schedule:
    cron: "* * * * *"
    timezone: Asia/Taipei
  entrypoint: ./prefect_demo.py:repo_info
  parameters: {}
  work_pool:
    name: worker-pool
    work_queue_name: default
    job_variables:
      service_account_name: "worker"
      name: "testestset"
      namespace: "dev"
      image: "docker.io/library/prefect_worker:1.0"
      image_pull_policy: IfNotPresent
      finished_job_ttl: 300
```

```python
import httpx   # an HTTP client library and dependency of Prefect
from prefect import flow, task

@task(retries=2)
def get_repo_info(repo_owner: str, repo_name: str):
    """Get info about a repo - will retry twice after failing"""
    url = f"https://www.google.com"
    api_response = httpx.get(url)
    api_response.raise_for_status()

@flow(log_prints=True)
def repo_info(repo_owner: str = "PrefectHQ", repo_name: str = "prefect"):
    """
    Given a GitHub repository, logs the number of stargazers
    and contributors for that repo.
    """
    get_repo_info(repo_owner, repo_name)
    print(f"Stars 🌠 ")

if __name__ == "__main__":
    repo_info()

```




dockerfile,主要只是打包程式進image  
```
FROM prefecthq/prefect:2-python3.11-kubernetes

COPY ./etl /home/

WORKDIR /home
```

```
docker build . -t prefect_worker:1.0
```


```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg
  namespace: dev
spec:
  selector:
    matchLabels:
      app: pg
  serviceName: svc-pg # 後續配合的service名稱, 這邊是必須的 否則不會產生綁定pod的dns
  replicas: 1
  template: # pod template
    metadata:
      labels:
        app: pg
    spec:
      containers:
        - name: info
          imagePullPolicy: IfNotPresent
          image: postgres:15.1
          ports:
            - containerPort: 5432
              name: pg
          env:
            - name: "POSTGRES_DB"
              value: "prefect"
            - name: "POSTGRES_PASSWORD"
              value: "dev123"

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prefect-server
  namespace: dev
spec:
  selector:
    matchLabels:
      app: prefect-server
  replicas: 1
  template: # pod template
    metadata:
      labels:
        app: prefect-server
    spec:
      containers:
        - name: prefect-server
          image: prefecthq/prefect:2-python3.11
          command: [ "/opt/prefect/entrypoint.sh", "prefect", "server", "start" ]
          imagePullPolicy: IfNotPresent
          env:
            - name: PREFECT_API_DATABASE_CONNECTION_URL
              value: "postgresql+asyncpg://postgres:dev123@pg-0.svc-pg.dev.svc.cluster.local:5432/prefect"
            - name: PREFECT_SERVER_API_HOST
              value: "0.0.0.0"
            - name: PREFECT_UI_URL
              value: "http:///svc-prefect.dev.svc.cluster.local:4200/api"
            - name: PREFECT_API_URL
              value: "http://svc-prefect.dev.svc.cluster.local:4200/api"
          lifecycle:
            preStop:
              exec:
                command: [ "prefect", "work-pool", "delete", "worker-pool" ]
---
apiVersion: v1
kind: Service
metadata:
  name: svc-prefect # service name
  namespace: dev # namspace
spec:
  selector: # label選擇器,用於確定 service代理哪些Pod
    app: prefect-server # label key:value
  #若開啟 , 同一個IP的請求會分到 固定的pod上
  type: NodePort
  ports: # port基本資訊
    - port: 4200 # service port
      protocol: TCP
      targetPort: 4200 # pod port
      nodePort: 30001
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: worker-role
  namespace: dev
rules:
  - apiGroups: [ "batch","" ]
    resources: [ "jobs","namespaces","pods","pods/log" ]
    verbs: [ "get", "list", "watch", "create","delete","patch" ]
---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: worker
  namespace: dev
  labels:
    app: prefect-worker
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: worker-rolebinding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: worker
  namespace: dev
roleRef:
  kind: ClusterRole
  name: worker-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prefect-worker
  namespace: dev
spec:
  selector:
    matchLabels:
      app: prefect-worker
  replicas: 1
  template: # pod template
    metadata:
      labels:
        app: prefect-worker
    spec:
      serviceAccountName: worker
      containers:
        - name: prefect-worker
          image: prefect_worker:1.0
          command: [ "/opt/prefect/entrypoint.sh", "prefect", "worker","start", "--pool","worker-pool","--type","kubernetes" ]
          imagePullPolicy: IfNotPresent
          env:
            - name: PREFECT_API_URL
              value: "http://svc-prefect.dev.svc.cluster.local:4200/api"
          lifecycle:
            postStart:
              exec: # 在main container 啟動時 執行命令
                command: ["/bin/sh", "-c", "sleep 15; prefect --no-prompt deploy --all"]

---

apiVersion: v1
kind: Service
metadata:
  name: svc-pg # service name
  namespace: dev # namspace
spec:
  selector: # label選擇器,用於確定 service代理哪些Pod
    app: pg # label key:value
  #若開啟 , 同一個IP的請求會分到 固定的pod上
  clusterIP: None
  ports: # port基本資訊
    - port: 5432 # service port
      protocol: TCP
      targetPort: 5432 # pod port
```

http://127.0.0.1:30001 驗證  

