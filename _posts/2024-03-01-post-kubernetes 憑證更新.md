---
title: "[筆記] Kubernetes 憑證更新"
categories:
  - 筆記
tags:
  - Kubernetes
toc: true
toc_label: Index
mermaid: true
---


kubernetes 憑證期限只有一年  

若組件期間有更新, 也會renew憑證日期  

除此之外需自行更新  


### 查看有效期限  

```
sudo kubeadm certs check-expiration
```


```
hccuse@master ~> sudo kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[check-expiration] Error reading configuration from the Cluster. Falling back to default configuration

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Oct 16, 2023 09:06 UTC   <invalid>       ca                      no
apiserver                  Oct 16, 2023 09:06 UTC   <invalid>       ca                      no
apiserver-etcd-client      Oct 16, 2023 09:06 UTC   <invalid>       etcd-ca                 no
apiserver-kubelet-client   Oct 16, 2023 09:06 UTC   <invalid>       ca                      no
controller-manager.conf    Oct 16, 2023 09:06 UTC   <invalid>       ca                      no
etcd-healthcheck-client    Oct 16, 2023 09:06 UTC   <invalid>       etcd-ca                 no
etcd-peer                  Oct 16, 2023 09:06 UTC   <invalid>       etcd-ca                 no
etcd-server                Oct 16, 2023 09:06 UTC   <invalid>       etcd-ca                 no
front-proxy-client         Oct 16, 2023 09:06 UTC   <invalid>       front-proxy-ca          no
scheduler.conf             Oct 16, 2023 09:06 UTC   <invalid>       ca                      no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Oct 13, 2032 09:06 UTC   8y              no
etcd-ca                 Oct 13, 2032 09:06 UTC   8y              no
front-proxy-ca          Oct 13, 2032 09:06 UTC   8y              no
```


### 更新憑證  

```
kubeadm certs renew all
```


```
hccuse@master /e/kubernetes> sudo kubeadm certs renew all
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[renew] Error reading configuration from the Cluster. Falling back to default configuration

certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
certificate for the API server to connect to kubelet renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed
certificate for liveness probes to healthcheck etcd renewed
certificate for etcd nodes to communicate with each other renewed
certificate for serving etcd renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed

Done renewing certificates. You must restart the kube-apiserver, kube-controller-manager, kube-scheduler and etcd, so that they can use the new certificates.
```

```
hccuse@master ~> sudo kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Mar 01, 2025 02:33 UTC   364d            ca                      no
apiserver                  Mar 01, 2025 02:33 UTC   364d            ca                      no
apiserver-etcd-client      Mar 01, 2025 02:33 UTC   364d            etcd-ca                 no
apiserver-kubelet-client   Mar 01, 2025 02:33 UTC   364d            ca                      no
controller-manager.conf    Mar 01, 2025 02:33 UTC   364d            ca                      no
etcd-healthcheck-client    Mar 01, 2025 02:33 UTC   364d            etcd-ca                 no
etcd-peer                  Mar 01, 2025 02:33 UTC   364d            etcd-ca                 no
etcd-server                Mar 01, 2025 02:34 UTC   364d            etcd-ca                 no
front-proxy-client         Mar 01, 2025 02:34 UTC   364d            front-proxy-ca          no
scheduler.conf             Mar 01, 2025 02:34 UTC   364d            ca                      no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Oct 13, 2032 09:06 UTC   8y              no
etcd-ca                 Oct 13, 2032 09:06 UTC   8y              no
front-proxy-ca          Oct 13, 2032 09:06 UTC   8y              no
```

### 其他補充  


```
hccuse@master ~> kubectl get node
error: You must be logged in to the server (Unauthorized)
```

是因為雖然 /etc/kubernetes/下的憑證已更新 , 但客戶端訪問cluster的文件($HOME/.kube)還是舊的  
只需把更新完的文件更新過去就可以正常使用  

```
cp /etc/kubernetes/admin.conf ~/.kube/config
```


```
hccuse@master ~> kubectl get node
NAME     STATUS   ROLES           AGE    VERSION
master   Ready    control-plane   501d   v1.25.3
node1    Ready    <none>          501d   v1.25.3
node2    Ready    <none>          501d   v1.25.3
```


也更新其他兩個節點的客戶端文件,使非master node也可以訪問api-server   

```
scp -r ~/.kube/ hccuse@10.140.0.10:/home/hccuse/
scp -r ~/.kube/ hccuse@10.140.0.11:/home/hccuse/
```


