---
title: "[筆記] Kubernetes 身份驗證與授權"
categories:
  - 筆記
tags:
  - Kubernetes
toc: true
toc_label: Index
mermaid: true
---

鑒權基本上分為身份驗證(authentication)與授權(authorization)

- authentication:
    - service account
    - user account
- authorization:
    - RBAC
        - Role
        - ClusterRole
        - RoleBinding
        - ClusterRoleBinding

## Authentication

身份驗證

### Service Account

用於Pod與API server進行交互時的身份驗證

- 若需要使用Pod與API server進行交互時, 需要特別設置service account進行鑒權
- 在無設置的情況下, 會自動使用名為`default`的service account, 該default 為 每個namespace創建時就會被一起創建
- 可打包 imagePullSecret, 若Pod無設置, 會自動帶入 Service Account imagePullSecret
- Service Account 提供 Pod 訪問apiServer的形式為將 token 以 volume方式掛在 Pod上 ,
  在/var/run/secret/kubernetes.io/serviceaccount, 該token其實就是使用JWT的機制來驗證使apiServer驗證身份

```bash
kubectl get sa
#NAME      SECRETS   AGE
#default   0         24h
```

參考field  
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: example-serviceaccount
  namespace: default
  labels:
    app: my-app
    environment: production
  annotations: # 註釋
    description: "This is an example ServiceAccount in the default namespace"
secrets:
  - name: my-secret
imagePullSecrets:
  - name: my-registry-key
automountServiceAccountToken: true # 自動掛載token
```

實際範例

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-ac
  namespace: dev
```


### User Account

用於kubectl與API server進行交互時的身份驗證

- 實做待補充 ...

紀錄一下概念
這邊User Account 實際上就是在創建一個合法的~/.kube/config檔案
這檔案實際上就是一個auth文件, kubectl交互就是依賴它表明身份並跟哪個cluster交互
可以理解成一個client端的auth文件
<br/>
這邊創建User是同HTTPS身份驗證的概念,
作法上是創建一個私鑰 key , 並創建一個CSR,
使用kubernetes的CA 自簽憑證
有該憑證與key , 可以使用kubectl 創建一個User 並設置 credentials (實際上就是幫你在~/.kube/config中新增user)
創建該user後, 就可以使用context 來切換使用者身份
<br/>
設置權限的部份
剛創建的user 是無任何權限的, 添加權限的方式是同service account,  使用RBAC,
透過創建Role, ClusterRole, RoleBinding, ClusterRoleBinding  來設置權限
<br/>
透過User account, 可以達成在隨意的server只要有kubectl,只要~/.kube/config 有授權, 透過切換context, 轉換不同身份與多個k8s的cluster進行交互



## Authorization

驗證該身份可以做哪些事

### RBAC

Role-Based-Access Control, 基於角色進行權限控制

定義完後, 可以綁定的目標為

- User
- Group
- ServiceAccount

#### apiGroups Resources Verbs 參考

常用資源對照

| API Group                   | Resources                                                                                                         |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------|
| `""` (core api group)       | `pods`, `services`, `nodes`, `configmaps`, `secrets`, `namespaces`, `persistentvolumeclaims`, `persistentvolumes` |
| `apps`                      | `deployments`, `statefulsets`, `daemonsets`, `replicasets`                                                        |
| `batch`                     | `jobs`, `cronjobs`                                                                                                |
| `autoscaling`               | `horizontalpodautoscalers`                                                                                        |
| `networking.k8s.io`         | `ingresses`, `networkpolicies`                                                                                    |
| `rbac.authorization.k8s.io` | `roles`, `clusterroles`, `rolebindings`, `clusterrolebindings`                                                    |
| `storage.k8s.io`            | `storageclasses`, `volumeattachments`                                                                             |

| Verb     |
|----------|
| `get`    |
| `list`   |
| `watch`  |
| `create` |
| `update` |
| `patch`  |
| `delete` |

```bash
kubectl api-versions
# list api group

kubectl api-resources --api-group=<api-group-name>
# list resource of the api group
```

```bash
kubectl api-resources --api-group=apps
#NAME                  SHORTNAMES   APIVERSION   NAMESPACED   KIND
#controllerrevisions                apps/v1      true         ControllerRevision
#daemonsets            ds           apps/v1      true         DaemonSet
#deployments           deploy       apps/v1      true         Deployment
#replicasets           rs           apps/v1      true         ReplicaSet
#statefulsets          sts          apps/v1      true         StatefulSet
```

#### Role

namespace level, 用於定義針對 namespace level 特定resource(Pods,Services,ConfigMap...) 的權限控制

範例, 該role, 在namespace dev 可以 get,watch,list,create,delete,patch 所有jobs

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: worker-role
rules:
  - apiGroups: [ "batch" ]
    resources: [ "jobs" ]
    verbs: [ "get", "list", "watch", "create","delete","patch" ]
```

#### clusterRole

cluster level, 用於定義針對 cluster level 特定resource(Pods,Services,ConfigMap...) 的權限控制

範例, cluster level, 可以get,watch,list 該cluster 內的所有 pods

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: worker-role
rules:
  - apiGroups: [ "batch" ]
    resources: [ "jobs" ]
    verbs: [ "get", "list", "watch", "create","delete","patch" ]
```

#### RoleBinding & ClusterRoleBinding

RoleBinding與ClusterRoleBinding的設定方式相同, 只是RoleBinding是針對namespace, ClusterRoleBinding是針對cluster

subjects: 主詞, 可以是User, Group, ServiceAccount , group待釐清 ,,,
roleRef: 綁定哪個Role or  ClusterRole


Binding kind apiGroup 對照表

| Kind             | API Group                   |
|------------------|-----------------------------|
| `Role`           | `rbac.authorization.k8s.io` |
| `ClusterRole`    | `rbac.authorization.k8s.io` |
| `User`           | `rbac.authorization.k8s.io` |
| `Group`          | `rbac.authorization.k8s.io` |
| `ServiceAccount` | `""` (core API group)       |


設定檔範例

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: example-rolebinding-sa
  namespace: default
  labels:
    app: my-app
    environment: production
  annotations:
    description: "This RoleBinding binds a Role to a ServiceAccount in the default namespace"
subjects:
  - kind: ServiceAccount
    name: example-serviceaccount
    namespace: default
  - kind: User
    name: example-user
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: example-group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```


實際範例

這邊為某pod, 可以get,watch,list,create,delete,patch 所有jobs,namespaces,pods,pods/log

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: worker-role
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
```
