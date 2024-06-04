---
title: "[筆記] Kubernetes Config"
categories:
  - 筆記
tags:
  - Kubernetes
toc: true
toc_label: Index
mermaid: true
---


kubernetes config物件, 也可作為環境變數注入

- configmap: 可將config注入至deployment or statefulSet
- secret(Opaque): 同上, 但會經過base64編碼



## configmap

文本(or key value)抽象化為kubernetes的資源類型

### 輸出方式

- 將secret作為檔案注入pod
- 將secret輸出為環境變數


### 創建

- 方法一

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap1
  namespace: dev
data: # 會產生兩個檔案 username 與 password , 內容為對應 value
  username: "admin"
  password: "123456"
```

or

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap2
  namespace: dev
data:
  info: |
    username: admin
    password: 123456
# 這邊 info 會成 目標目錄的檔名 | 下面為 檔案內容
```

- 方法二

不限於檔案 也可是目錄(會將目錄下所有檔案打包)

```bash

cat << EOF > myconf.txt
username: admin
password: 123456
EOF

kubectl create configmap configmap3 --from-file=./myconf.txt -n dev

kubectl get configmap configmap3 -n dev -o yaml

```

- 方法三

```bash
kubectl create configmap configmap4 --from-literal=username=admin --from-literal=password=123456 -n dev
```

查看
```bash
kubectl get configmap configmap4 -n dev -o yaml
```

### ENV & Volume file

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config1
  namespace: dev
data:
  username: "admin1"
  password: "123456"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config2
  namespace: dev
data:
  username: "admin2"
  password: "1234567"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config3
  namespace: dev
data:
  username: "admin3"
  password: "12345678"
---
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-test-pod
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:latest
      env: # 擷取configmap中特定key value 作為環境變數
        - name: USERNAME1  # 輸出至container內部的 環境變數 key
          valueFrom:
            configMapKeyRef:
              name: env-config1 # 取得的configmap name
              key: username # 該configmap 目標key
        - name: PASSWORD2  # 輸出至container內部的 環境變數 key
          valueFrom:
            configMapKeyRef:
              name: env-config2 # 取得的configmap name
              key: password # 該configmap 目標key
      envFrom: #此方式會將cm中 env-config3中所有變量 作為環境變數輸入
        - configMapRef:
            name: env-config3
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config

  volumes:
    - name: config-volume
      configMap:
        name: env-config3

  restartPolicy: Never
```


```bash
kubectl logs cm-env-test-pod -n dev
#KUBERNETES_PORT=tcp://10.96.0.1:443
#KUBERNETES_SERVICE_PORT=443
#HOSTNAME=cm-env-test-pod
#SHLVL=1
#username=admin3      <<< 輸出成功
#HOME=/root
#PASSWORD2=1234567     <<< 輸出成功
#KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
#PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
#KUBERNETES_PORT_443_TCP_PORT=443
#password=12345678      <<< 輸出成功
#KUBERNETES_PORT_443_TCP_PROTO=tcp
#USERNAME1=admin1       <<< 輸出成功
#KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
#KUBERNETES_SERVICE_PORT_HTTPS=443
#KUBERNETES_SERVICE_HOST=10.96.0.1
#PWD=/
ls /etc/config/
#password  username
cat /etc/config/username
#admin3
cat /etc/config/password
#12345678
```

### SubPath

使用configmap注入為檔案時, 預設是將空白目錄覆蓋, 之後再注入configmap的檔案
e.g.  nginx.conf  注入 /etc/nginx/  , 此時 /etc/nginx/ 該目錄會被空白目錄覆蓋, 之後只剩nginx.conf
若只想更換該目錄下的特定檔案, 就會需要使用SubPath


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: dev
data:
  nginx.conf: |
    user  nginx;
    worker_processes  auto;

    error_log  /var/log/nginx/error.log notice;
    pid        /var/run/nginx.pid;


    events {
    worker_connections  1024;
      }


    http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
      }

---
## NO SubPath
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-subpath-non
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:latest
      command:
        - /bin/sh
        - -c
        - "sleep 3600"
      volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/

  volumes:
    - name: nginx-config-volume
      configMap:
        name: nginx-config

  restartPolicy: Never
---
## SubPath
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-subpath
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:latest
      command:
        - /bin/sh
        - -c
        - "sleep 3600"
      volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf

  volumes:
    - name: nginx-config-volume
      configMap:
        name: nginx-config

  restartPolicy: Never
```

實際效果
```bash

kubectl exec -it cm-env-subpath-non -n dev -- bash
root@cm-env-subpath-non:/# ls /etc/nginx/
#nginx.conf
exit
kubectl exec -it cm-env-subpath -n dev -- bash
root@cm-env-subpath:/# ls /etc/nginx/
#conf.d  fastcgi_params  mime.types  modules  nginx.conf  scgi_params  uwsgi_params
```





## secret


- Opaque部分, 基本上跟上面configmap相同, 只是為base64 , 但注入pod後, 內容會為明文
- kubernetes.io / dockerconfigjson: 主要用來儲存docker 私有repository 的授權認證
- Service Account: 主要用來作為pod 訪問apiServer的授權認證方式

kubernetes.io / dockerconfigjson , Service Account 待補充....

### 輸出方式

- 將secret作為檔案注入pod
- 將secret輸出為環境變數

### 創建

```bash
leo2@master ~> echo -n 'admin' | base64
#YWRtaW4=
leo2@master ~> echo -n '123456' | base64
#MTIzNDU2
#加密

leo2@leo ~> echo MTIzNDU2 | base64 -d
#123456
# 解密
```

### 使用範例

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
  namespace: dev
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
# admin 編碼後密文
# 123456 編碼後密文

---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: secret-config # 使用 volume
          mountPath: /secret/config # container 內部 volume目錄
      envFrom:
        - secretRef:
            name: my-secret
  volumes: #定義volume
    - name: secret-config
      secret:
        secretName: secret
```


```bash
ls /secret/config/
#password  username
cat /secret/config/username
#admin
cat /secret/config/password
#123456
env
#KUBERNETES_SERVICE_PORT_HTTPS=443
#KUBERNETES_SERVICE_PORT=443
#HOSTNAME=secret-pod
#PWD=/
#PKG_RELEASE=1~bookworm
#HOME=/root
#KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
#NJS_VERSION=0.8.4
#TERM=xterm
#username=admin   <<< 輸出成功
#SHLVL=1
#KUBERNETES_PORT_443_TCP_PROTO=tcp
#KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
#password=123456  <<< 輸出成功
#KUBERNETES_SERVICE_HOST=10.96.0.1
#KUBERNETES_PORT=tcp://10.96.0.1:443
#KUBERNETES_PORT_443_TCP_PORT=443
#PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
#NGINX_VERSION=1.25.5
#NJS_RELEASE=3~bookworm
#_=/usr/bin/env
```
