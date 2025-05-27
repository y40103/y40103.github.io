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
  "credHelpers": {
    "OOOOO.dkr.ecr.ap-northeast-1.amazonaws.com": "ecr-login"
  }
}

```

設置後 就不必每次都重新docker login

## kubernetes

由於ecr auth 只能依靠有時校性的token
這邊簡單說是啟動一個pod來 持續更新 ECR 用的 docker secret, 這邊example是用pod  
實際上會用cronjon讓它持續更新  
在pod內更新secret 會需要設置service account給pod基礎權限,需要設置基本的rbac

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: default
spec:
  replicas: 1 # Number of pod replicas
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/demo:v1.0
          ports:
            - containerPort: 9000
      imagePullSecrets:
        - name: regcred
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubectl-sa
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-writer # 給一個描述性的名字
rules:
  - apiGroups: [""] # "" 代表核心 API 組
    resources: ["pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubectl-sa-pod-reader-writer-binding
subjects:
  - kind: ServiceAccount
    name: kubectl-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: pod-reader-writer
  apiGroup: rbac.authorization.k8s.io

---
# 以下無關 只是剛好做aws ecr  token 重刷測試

# 一般會用cronjob 這邊用pod 確認有正常執行
apiVersion: v1
kind: Pod
metadata:
  name: kubectl-pod
  namespace: default # 和 ServiceAccount 相同的 namespace
spec:
  serviceAccountName: kubectl-sa # 使用上面創建的 ServiceAccount
  containers:
    - name: kubectl-container
      restartPolicy: OnFailure
      image: odaniait/aws-kubectl:latest # 或者 alpine/k8s:1.28.2
      envFrom:
        - secretRef:
            name: ecr-registry-helper-secrets
        - configMapRef:
            name: ecr-registry-helper-cm
      command:
        - /bin/sh
        - -c
        - |-
          ECR_TOKEN=`aws ecr get-login-password --region ${AWS_REGION}`
          kubectl delete secret --ignore-not-found $DOCKER_SECRET_NAME -n $namespace
          kubectl create secret docker-registry $DOCKER_SECRET_NAME \
          --docker-server=https://${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com \
          --docker-username=AWS \
          --docker-password="${ECR_TOKEN}" \
          --namespace=$namespace
---
apiVersion: v1
kind: Secret
metadata:
  name: ecr-registry-helper-secrets
  namespace: default
stringData:
  AWS_SECRET_ACCESS_KEY: ""
  AWS_ACCESS_KEY_ID: ""
  AWS_ACCOUNT: ""

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecr-registry-helper-cm
  namespace: default
data:
  AWS_REGION: "ap-northeast-1"
  DOCKER_SECRET_NAME: regcred #######################################
  namespace: default
---

```
