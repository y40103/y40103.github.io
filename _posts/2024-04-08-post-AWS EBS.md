---
title: "[筆記] AWS EBS"
categories:
  - 筆記
tags:
  - AWS
toc: true
toc_label: Index
---

## EBS

是一個持久化的獨立資源, 可以理解成 AWS 的網路硬碟

- 類似NFS,主要是讓EC2的數據持久化
- 非物理硬體,是一種network drive, 會有latency
- 具有區域性, 可以指定區域, 但是不能跨區
- 同個區域內可以自由在不同EC2上掛載
- EBS一次只能掛載一台EC2 , 少數例外, e.g. EBS Multi-Attach  
- 一台EC2可以同時掛載多個EBS
- 若有跨區需求 可以使用snapshot 進行跨區複製

創建EC2時, 就會引導創建一個EBS, 這個EBS就是EC2的系統硬碟, 預設生命週期會跟隨EC2, EC2刪除, EBS也會刪除

## EBS Snapshot

EC2 具有區域性, 若想將資料進行跨區複製, 可以使用snapshot,   
實際作法為 先在原本的EBS使用snapshot    
snapshot完成後,就可以使用該snapshot創建出該EBS, 創建時就可以選擇區域了

snapshot, 本質上很類似OOP的抽象類, 有該snapshot就可以無限創建出該EBS

### Snapshot Recycle Bin

可以理解成 snapshot的回收桶, 用來存放被刪除的snapshot, 可以制定留存規則, 可留存1天至1年
防止誤刪, 也可以用來做版本控制, 例如每天都會自動備份, 但是只保留最近7天的snapshot, 其他的就會被真正刪除

### Snapshot Archive

可以理解成 snapshot的冷儲存, 用來存放不常用的snapshot,  
跟S3的Glacier類似, 最多節省75%的費用, 代價是要取出的時候, 需等待24-72小時,

### Fast Snapshot Restore

可以理解成 snapshot的快速還原, 用來加速snapshot的還原速度,
若snapshot的EBS volume很大的話, 創建出原來的EBS需要較長的等待時間, 該功能可以大大加速創建的速度,  
需注意 費用很貴

### EBS Volume Types

| Type    | Description                                               |
|---------|-----------------------------------------------------------|
| gp2/gp3 | General Purpose SSD, 適合大部分的使用場景, 3 IOPS/GB, 最大16,000 IOPS |
| io1/io2 | Provisioned IOPS SSD, 適合需要高IOPS的場景, 最大64,000 IOPS         |
| st1     | Throughput Optimized HDD, 適合大量連續讀寫的場景, 最大500MB/s          |
| sc1     | Cold HDD, 適合大量連續讀寫的場景, 最大250MB/s                          |

#### General SSD

- CP值高, 適合大部分的使用場景, e.g. 系統, 測試, 開發 ...
- 1GiB - 16TiB

gp3
- IOPS 可3000-16000, 
- 吞吐量可125-1000MB/s,  可彈性調整

gp2: 
- 性能跟容量成正比
- 3 IOPS/GiB, 5334 G 可至最大性能 16,000 IOPS
- 一些小容量可以至 3000 IOPS


### PIOPS SSD

piops = provisioned iops  
適用需高度IOPS的場景, e.g. 資料庫, 大量讀寫的場景  

支援multi-attach

#### io1

- 4GiB - 16TiB
- max piops 64,000 for Nitro EC2 instance, 其他類型 32,000
- 可直接調整ipos, 無關容量

#### io2

- 4GiB - 64TiB  
- sub-millisecond latency  
- max piops 256000 , 但跟容量相關, 1Gib = 1000 IOPS 

#### 補充: multi-attach

- 可以同時掛載多台EC2, 但需在同一個AZ(同區同資料中心)
- 各節點寫入需統一管理  
- 最多可以掛載16台EC2
- 必須為 cluster-aware 的檔案系統  



### HDD

- 無法作為系統硬碟

#### sc1

- 冷資料,少用, 用來歸檔 
- 便宜, 價格敏感者可考慮  
- max 250 MiB/s, 250 IOPS

#### st1

- 適用 大量log, 資料倉儲, 大數據
- max 500 MiB/s, 500 IOPS


