---
title: "[筆記] AWS Log Insight"
categories:
  - 筆記
tags:
  - AWS
toc: true
toc_label: Index
mermaid: true
---

可以理解成 CloudWatch Log Group 專用的log query tool  

可以對log 做一些基礎的filter sort ... 

以 vpc log flow為例  

共有 4個 field , timestamp, message , logStram , log   

### 基礎 pattern

```
fields @timestamp, @message, @logStream, @log
| sort @timestamp desc
| limit 1000
```

類似SQL  

```sql
select *
from vpc_log_flow(timestamp, message, logStram, log)
order by timestamp
    limit `1000`  
```

output  

| @timestamp                    | @message                                                                                                           | @logStream                | @log                                          |
|-------------------------------|--------------------------------------------------------------------------------------------------------------------|---------------------------|-----------------------------------------------|
| 2024-06-28T15:32:14.000+08:00 | 2 767397684503 eni-087a88e383f52560f 23.98.107.225 15.0.1.175 443 41976 6 43 33953 1719559934 1719559962 ACCEPT OK | eni-087a88e383f52560f-all | 767397684503:etcprod24-etc-vpc-flow-log-group |

### state , filter

state 類似SQL group by 效果 , 但作為聚合條件的key, 也會被輸出成一個欄位    
filter 類似SQL where 效果  

```
fields @timestamp, @message
 | stats count(*) as records by dstPort, srcAddr, dstAddr as Destination
 | filter dstPort="80" or dstPort="443" or dstPort="22" or dstPort="25"
 | sort HitCount desc
 | limit 10
```

row2,3 會等同於SQL  

```sql
select count(*) as records, srcAddr, dstAddr as Destination
from vpc_log_flow(@timestamp, @message)
group by dstPort, arcAddr, dstAddr
order by HitCount desc limit 1
```

output 結果大概會是以下

| dstPort | srcAddr    | dstAddr        | records |
|---------|------------|----------------|---------|
| 80      | 15.0.2.190 | 15.0.1.175     | 237     |
| 443     | 15.0.1.175 | 61.216.156.141 | 64      |

### vpc log flow format

| Key         | Value                 |
|-------------|-----------------------|
| accountId   | 流日誌記錄的AWS帳戶ID         |
| action      | 流量動作（REJECT , ACCEPT） |
| bytes       | 流量中的字節數量              |
| dstAddr     | 流量的目的IP地址             |
| dstPort     | 流量的目的端口               |
| end         | 流量記錄結束時間（UNIX時間戳格式）   |
| interfaceId | 產生流日誌的網絡接口的ID         |
| logStatus   | 流日誌記錄的狀態（OK 表示正常）     |
| packets     | 流量中的數據包數量             |
| protocol    | 流量使用的協議類型（6 表示TCP協議）  |
| srcAddr     | 流量的來源IP地址             |
| srcPort     | 流量的來源端口               |
| start       | 流量記錄開始時間（UNIX時間戳格式）   |
| version     | 流日誌的版本號               |

### 範例

本機直連ec2 ssh log

FROM: 61.216.156.141 , ephemeral=51204
TO: 10.0.1.41 port=22

request

```
fields @timestamp, @message
| filter dstPort="22" and action="ACCEPT" and srcAddr="61.216.156.141" and and dstAddr="10.0.1.41"
| sort @timestamp desc
| limit 100
```

```
2024-07-01T11:21:47.000+08:00
2 767397684503 eni-04dff7854a5273004 61.216.156.141 10.0.1.41 51204 22 6 23 5269 1719804107 1719804133 ACCEPT OK
# 10.0.1.41 ( ec2 VPC IP )
```



TO: 10.0.1.41 port=22
FROM: 61.216.156.141 , ephemeral=51204

response

```
fields @timestamp, @message
| filter srcPort="22" and action="ACCEPT" and dstAddr="61.216.156.141" and srcAddr="10.0.1.41"
| sort @timestamp desc
| limit 100
```

```
2024-07-01T11:21:47.000+
2 767397684503 eni-04dff7854a5273004 10.0.1.41 61.216.156.141 22 51204 6 24 5941 1719804107 1719804133 ACCEPT OK
```
