---
title: "[筆記] AWS EFS"
categories:
  - 筆記
tags:
  - AWS
toc: true
toc_label: Index
---

# EFS

AWS的NFS, 可以直接掛在EC2上, 且可跨區域(multi AZ), 也可以同時被多個EC2掛載  


- 主要用於多個EC2共享數據  
- 與EBS相同, 也是一種network drive, 會有latency  
- 價格為高, 約為EBS 的 3倍 (3x gp2)  
- 只適用於Linux, 不支援Windows  
- 可跨區 (e.g. us-east-1a , us-east-1b , us-east-1c ... )
- 使用 secure group 管理訪問權限  (EC2 > security group )
- 不需規劃容量, 自動擴展  
- kms加密存儲

## Scale

- 可併發1000+ nfs client, 10GB+ 的吞吐量  
- 可自動擴展至PB級吞吐量

## Throughput Mode

- Bursting, 主要是依據使用容量對吞吐量進行擴展 ,1TB = 50MiB/s, 最大可以擴展至100MiB/s  
- (Enhance)Provisioned, 可自定義吞吐量, e.g. 1GiB/s for 1TB
- (Enhance)Elastic, autoscale throughput on workload 自動調整吞吐量,可用於吞吐無法預測情況, 最大至 3GiB/s for reads, 1GiB/s for writes


## Storage class

存儲類型, 同s3可設置Lifecycle management, 依據設定將資料移動至不同的存儲類型  

- Standard, 高頻存取
- Infrequent Access, 存儲成本低, 但取出資量需另外計費 (需開啟 EFS-IA with a life cycle)

## Availability and durability

可用性

- Standard: Multi-AZ, 適合生產環境, 高可用 (default) 
- One Zone: Single-AZ, 適合開發環境, 低成本, 可與Infrequent Access搭配使用, 可節省90%的費用  


## Performance Mode

- General Purpose, 為default 用於latency-sensitive workloads
- Max I/O, 可容忍更高的延遲, 但可以提供更高的吞吐量 (影像處理, 巨量資料處理)  



## Mount Target


可設置不同的AZ的EC2 instance 若共用特定group, 就可共享EFS, 可設置policy, 來控制存取權限  


## 與EBS的比較

主要差異  

### EBS

- 只能volume於單一EC2 (除非使用 io1/io2,才可以multi-attach)
- default EC2被刪除時, EBS會被刪除(可設置保留)
- 只能單一AZ level (e.g. us-east-1a )
- gp2: IO 隨著容量增加  
- gp3, io1 : 可自定義IOPS

migrate EBS to other AZ

- create snapshot (會吃到該機器的IO, 若有大量IO, 不適合使用)
- restore EBS from snapshot to another AZ

### EFS

- 可多個EC2 instance共享數據  
- 只適用於Linux(POSIX,Portable Operating System Interface), 不支援Windows
- 相較於EBS,價格高 
