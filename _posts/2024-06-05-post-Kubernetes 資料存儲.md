---
title: "[筆記] Kubernetes 資料存儲"
categories:
  - 筆記
tags:
  - Kubernetes
toc: true
toc_label: Index
mermaid: true
---
kubernetes Pod資料持久化, 這邊舉目前比較有機會用到的方式  

- volume
  - emptyDir
  - hostPath
  - NFS
- Persistent volume
  - PV
  - PVC

## volume

類似docker volume 機制, 可以將容器內外目錄共享  

### emptyDir

該類型目的為將pod內部的container目錄共享, 無持久化功能(該目錄不會被掛載至node上), 容器被撤銷時, 資料也會被刪除
EmptyDir會在Pod被分配到Node時被創建, 初始為空, 無須指定volume的目錄,   
kubernetes會自動分配一個目錄, Pod被刪除時, EmptyDir也會永久被刪除  (若執行中 Pod崩潰重啟, mptyDir資料不會受到影響,
只有刪除pod時 emptyDir的資料才會丟失 )   
<br/>
適用場景  

- 臨時空間  
- container與container之間需要交換數據時 (共享目錄)  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-busybox
  namespace: dev
spec:
  containers:
    - image: nginx:latest
      name: nginx
      ports:
        - name: nginx-port
          containerPort: 80
          protocol: TCP
      volumeMounts:
        - name: logs-volume # 將/var/log/nginx掛載至 logs-volume
          mountPath: /var/log/nginx/

    - image: busybox:latest
      name: busybox
      command: [ "/bin/sh","-c","tail -f /log/access.log" ]
      volumeMounts:
        - name: logs-volume  #將 /logs 掛載至 logs-volume
          mountPath: /log/

  volumes: # 聲明volume
    - name: logs-volume # 定義 logs-volume
      emptyDir: { } # volume 類型
```

測試  
在busybox /log/ 可以看到nginx /var/log/nginx/ 的檔案    
在busybox 創建檔案, 可以在nginx /var/log/nginx/下看到   
可以理解兩者目錄共享  

```bash
kubectl exec -it nginx-busybox -n dev -c nginx  --  bash
root@nginx-busybox:/# ls /var/log/nginx/
#access.log  error.log
exit
kubectl exec -it nginx-busybox -n dev -c nginx  --  bash
root@nginx-busybox:/# ls /var/log/nginx/
#access.log  error.log
root@nginx-busybox:/#
exit
kubectl exec -it nginx-busybox -n dev -c busybox  --  sh
/ #
/ # ls /log/
#access.log  error.log
/ # touch /log/frombusybox
kubectl exec -it nginx-busybox -n dev -c nginx  --  bash
root@nginx-busybox:/# ls /var/log/nginx/
#access.log  error.log  frombusybox
```

### hostPath

效果類似emptyDir, 只是會將共享的目錄掛載於實際node上, 若Pod被撤銷, 該目錄會被保留下來  
也因scheduler調度不一定會是同一個node, 因此會與網路磁碟或是分布式的存儲工具一起使用  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-busybox
  namespace: dev
spec:
  containers:
    - image: nginx:latest
      name: nginx
      ports:
        - name: nginx-port
          containerPort: 80
          protocol: TCP
      volumeMounts:
        - name: logs-volume # 將/var/log/nginx掛載至 logs-volume
          mountPath: /var/log/nginx/

    - image: busybox:latest
      name: busybox
      command: [ "/bin/sh","-c","tail -f /log/access.log" ]
      volumeMounts:
        - name: logs-volume  #將 /logs 掛載至 logs-volume
          mountPath: /log/

  volumes: # 聲明volume
    - name: logs-volume # 定義 logs-volume
      hostPath:
        path: /tmp/hostpath_temp/
        type: DirectoryOrCreate #預設資料夾 不存在則創建

  # type
  # Directory # 目錄必須存在
  # FileOrCreate 文件存在就使用, 不存在就先創建後使用
  # File 檔案必須存在
  # ....
```


測試  
```bash
root@dev-worker:/# ls /tmp/hostpath_temp/
access.log  error.log
```

### NFS

Pod中共享目錄, 且目錄掛載於NFS   

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-busybox
  namespace: dev
spec:
  containers:
    - image: nginx:latest
      name: nginx
      ports:
        - name: nginx-port
          containerPort: 80
          protocol: TCP
      volumeMounts:
        - name: logs-volume # 將/var/log/nginx掛載至 logs-volume
          mountPath: /var/log/nginx/

    - image: busybox:latest
      name: busybox
      command: [ "/bin/sh","-c","tail -f /log/access.log" ]
      volumeMounts:
        - name: logs-volume  #將 /logs 掛載至 logs-volume
          mountPath: /log/
	
  volumes: # 聲明volume
  - name: logs-volume # 定義 logs-volume
    nfs:
      server: 10.140.0.2
      path: /home/nfsVolume/
```

### 補充nfs佈署

```bash

yum install nfs-utils -y
# 安裝nfs

#or

apt-get install nfs-kernel-server nfs-common
	# 安裝nfs


# 包含 nfs server and work node 


mkdir /home/nfsVolume/ -p
# 在nfs 機器上 創建掛載目錄 (這邊我利用master作為 nfs server測試)

vim /etc/exports
# nfs server 上

```

/etc/exports
```
# 暴露至所有該domain的主機
/home/nfsVolume 10.140.0.0/24(rw,sync,no_root_squash)
```

```bash
sudo systemctl start nfs
	
# or
sudo systemctl start nfs-kernel-server
	
```




## Persistent Volume

volume抽象化物件管理, 簡單說就是把可以掛載至pod的目錄抽象化成kubernetes的物件, 可以被kubernetes管理    

### PV

PV = 被抽象化的資源名稱, 支援多種不同的類型  

- hostPath
- nfs
- local
- awsebs
- gcepd
- azuredisk
- cephfs
- ....


文件說明
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv2
spec:
  capacity: # 儲存能力 
    stroage: 20Gi
  accessModes: #訪問模式
# ReadWriteOnce(RWO) 讀寫權限,同時只能被單個node掛載 
# ReadOnlyMany (ROX) 只讀權限,可被多個node掛載
# ReadWriteMany (RWX) 讀寫權限,可被多個node掛載
# 不同類型的儲存模式(nfs等等...) 可用的類型可能會不同

  storageClassName: # 儲存類型
# 有指定特定類型的pv 其只能與該類型綁定
# 無指定特定類型的pv 能與任何類型綁定 (pvc無指定)

  persistentVolumeReclaimPolicy: #回收策略
# Retain  Pod撤銷時, 裡面檔案保留 , PVC被刪除後 ,須手動取出資料
# Recycle Pod撤銷時, PV中的資料自動刪除, 將狀態初始化為可用狀態 (僅於nfs hostpath)
# Delete  Pod撤銷時, 由掛載的底層儲存 進行刪除volume操作, 常見於雲服務(會自動產生pv 被釋放則自動刪除)
  nodeAffinity: # (optional) 同pod affinity設定, 可設置綁定node
  ...
  nfs: # 底層儲存類型 e.g. nfs , hostpath , cephfs ....
```




hostpath
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
```


nfs
```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: nfs-pv
    spec:
      capacity:
        storage: 5Gi
      accessModes:
        - ReadWriteMany
      nfs:
        path: /exported/path
        server: nfs-server.example.com
```

cephfs
```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: cephfs-pv
    spec:
      capacity:
        storage: 10Gi
      accessModes:
        - ReadWriteMany
      cephfs:
        monitors:
          - 10.16.154.78:6789
        path: /volumes/kubernetes
        user: admin
        secretRef:
          name: ceph-secret
```

#### Release復原

若被PVC綁定, 但後來PVC被刪除, PV STATUS會轉為release    
此時會被鎖定, 若有其他PVC有資格綁定該PV, 會無法綁定    
要回復狀態為 Available  

可使用edit 或是 patch , 刪除 `claimRef` 的內容 ,　就可回到Available  
```yaml
kubectl edit pv hostpath-pv -n dev
```

### PVC


PVC 可以理解成PV的 consumer, 需要先定義PVC 中PV需求, 當PVC被創建時, 會嘗試去綁定適合的PV    
PVC可分 `靜態` 與 `動態` , 動態可設置storageClass, 可使PVC自行創建適合的PV去綁定    


```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
  namespace: dev
spec:
  storageClassName: ""
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
  namespace: dev
spec:
  storageClassName: ""
  accessModes: #訪問模式
    - ReadWriteOnce
  resources: # 空間請求
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: dev
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["/bin/sh","-c","touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt;sleep 3;done;"]
      volumeMounts:
        - name: volume # 使用 volume
          mountPath: /tmp/ # container 內部 volume目錄

  volumes: #定義volume
    - name: volume
      persistentVolumeClaim:
        claimName: pvc1
        readOnly: false

```



 

