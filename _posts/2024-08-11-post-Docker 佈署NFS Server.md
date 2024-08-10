---
title: "[筆記] Docker 佈署NFS Server"
categories:
  - 筆記
tags:
  - NFS
toc: true
toc_label: Index
mermaid: true
---


直接使用container佈署nfs server


## NFS Server

NFS server

```bash
docker run -itd --privileged \
  --restart unless-stopped \
  -e SHARED_DIRECTORY=/data \
  -v /data/nfs-storage:/data \
  -p 2049:2049 \
  itsthenetwork/nfs-server-alpine:12
```

or

```yaml
version: "2.1"
services:
  # https://hub.docker.com/r/itsthenetwork/nfs-server-alpine
  nfs:
    image: itsthenetwork/nfs-server-alpine:12
    container_name: nfs
    restart: unless-stopped
    privileged: true
    environment:
      - SHARED_DIRECTORY=/data
    volumes:
      - ./data/nfs-storage:/data
    ports:
      - 2049:2049
    networks:
      my_nfs_network:
        ipv4_address: 172.16.0.2
networks:
  my_nfs_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.16.0.0/16
```

```bash
docker compose up -d
```

### NFS Clent

這邊執行為客戶端主機,  簡單說就是要把server的目錄共享在客戶端

```bash
sudo mount -v <nfs server ip>:/ <client nfs path>
```

執行範例
假設客戶端共享目錄為 ./client , 這邊客戶端為為 host主機,  目的是在host主機作與docker內部的nfs server交互
因設置 2049:2049  nfs server ip 直接使用host主機的172.16.1.143, 會直接轉至 nfs server container 內部

```bash
sudo mount -v 192.168.1.143:/ ./client
# /home/hccuse/Insync/y40103@gmail.com/GoogleDrive/hccuse/hccuse/learn/nfs/client
```

這樣就會將 nfs server volume目錄 /data 與 host的 client目錄 共享

```bash
hccuse@hcuuse-PC ~/I/y/G/h/h/l/n/client> df
#Filesystem      1K-blocks      Used Available Use% Mounted on
#tmpfs             1576704      2164   1574540   1% /run
#/dev/nvme0n1p5  338600516 223910256  97417020  70% /
#tmpfs             7883516    900992   6982524  12% /dev/shm
#tmpfs                5120         4      5116   1% /run/lock
#/dev/nvme0n1p1     523248     32416    490832   7% /boot/efi
#tmpfs             1576700       144   1576556   1% /run/user/1001
#192.168.1.143:/ 338600960 223909888  97417216  70% /home/hccuse/Insync/y40103@gmail.com/GoogleDrive/hccuse/hccuse/learn/nfs/client
```

取消則在host
```bash
sudo umount /home/hccuse/Insync/y40103@gmail.com/GoogleDrive/hccuse/hccuse/learn/nfs/client
```

## Troubleshooting

docker compose 無法卸除 , 可能是client端還在執行, 產生阻塞


```bash
Error response from daemon: cannot stop container: a832fb77e839c36d3f961171e5e6af3146ebf9a08ea0f78eff1ff8e31affd4be: tried to kill container, but did not receive an exit event
```

參考解決方式

```bash
sudo systemctl restart docker

killall Docker && open /Applications/Docker.app
```
