---
title: "[筆記] Prometheus 基礎使用"
categories:
  - 筆記
tags:
  - Prometheus
toc: true
toc_label: Index
mermaid: true
---

# Prometheus

基本組件

- Prometheus: 監控核心
- 各類Exporter: 監控資源串流擷取主件 , e.g. node_exporter
- Grafana: 前端顯示元件

## Prometheus 接收單位

| 元素          | 描述                                  |
|-------------|-------------------------------------|
| `metric`    | 指標名稱 (例如: `process_open_fds`)，特徵，標籤 |
| `timestamp` | 時間戳                                 |
| `value`     | 數值                                  |

```
process_open_fds{instance="localhost:9100", job="node"} @121313123 100
## metrics  @timestamp  value  
## process_open_fds  121313123  100  對應

等同

__name__="process_open_fds", instance="localhost:9100", job="node"  @121313123 100
```

### metric

```
<metric name>{<label name>=<label value>, ...} 
```

#### metric 數值類型

| 監控指標類型      | 描述              | 例子                          |
|-------------|-----------------|-----------------------------|
| `counter`   | 累加計數器，只能增加，不能減少 | `http_request_total`        |
| `gauge`     | 可增可減            | `node_memory_MemFree_bytes` |
| `histogram` | 統計數據分佈          | API response time 分佈        |

## 常見exporter

### postgres_exporter

主要是監測postgresql資料庫的資源使用情況

| 監控指標                                         | 描述                |
|----------------------------------------------|-------------------|
| `pg_stat_activity_max_tx_duration`           | 最大事務執行時間，最大延遲     |
| `pg_settings_max_connections`                | 最大連線數             |
| `pg_settings_superuser_reserved_connections` | 保留連線數             |
| `pg_stat_activity_count`                     | 目前連線數             |
| `pg_database_size_bytes`                     | 資料庫大小             |
| `pg_up`                                      | PostgreSQL 正常啟動數量 |

```
((sum(pg_settings_max_connections) by (server) - sum(pg_settings_superuser_reserved_connections) by (server)) - sum(pg_stat_activity_count) by (server)) / sum(pg_settings_max_connections) by (server)) * 100 < 10

# 可用連接數少於10% alert
```

### node-exporter

主要是用來監測實體機器的資源使用情況

#### 常用metric

#### CPU

``` 

node_cpu_seconds_total
# CPU使用時間

e.g.
node_cpu_seconds_total{mode="idle"}
# 空閒CPU時間
node_cpu_seconds_total{instance="test-node-linux"}
# 指定節點CPU使用時間

node_load5
# 五分鐘平均負載

```

#### Memory

``` 
node_memory_MemTotal_bytes
# 總記憶體大小 單位: byte, /1024/1024 = MB 

node_memory_MemAvailable_bytes
# 可用記憶體大小 空閒可用 = MemFree + Buffers + Cached

node_memory_MemFree_bytes
# 空閒記憶體大小 

```

#### Filesystem

``` 
node_filesystem_avail_bytes
# 磁碟可用空間大小 單位: byte, /1024/1024 = MB 

node_filesystem_size_bytes
# 磁碟總空間大小


100 - ((node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"} * 100) /            node_filesystem_size_bytes{mountpoint="/",fstype!="rootfs"})
# 烤用磁碟空間百分比

```

#### network

```
node_network_transmit_bytes_total
## 網路傳輸流出資料量 單位: byte, /1024/1024 = MB 

node_network_receive_bytes_total
## 網路傳輸流入資料量 單位: byte, /1024/1024 = MB


container_network_receive_bytes_total
## 容器網路傳輸流入資料量 單位: byte, /1024/1024 = MB

container_network_transmit_bytes_total
## 容器網路傳輸流出資料量 單位: byte, /1024/1024 = MB
```

## PromQL

| 資料型態             | 描述            |
|------------------|---------------|
| `instant vector` | 瞬間向量，一個時間點的資料 |
| `range vector`   | 區間向量，一段時間內的資料 |
| `scalar`         | 純量，單一數值       |
| `string`         | 字串            |

### instant vector

如 node_memory_Active_bytes 進行查詢的數值

 ```
 node_memory_Active_bytes{instance="test-node-linux", job="node1"}
 
 4157243392
 ```

### range vector

node_memory_Active_bytes[1m] 進行查詢的數值

```
node_memory_Active_bytes{instance="test-node-linux", job="node1"}
4156256256 @1701962276.312
4157243392 @1701962291.312
4157317120 @1701962306.311
4157358080 @1701962321.312

# default 1 collection per 15s  , 1m = 4 times
```

### scalar

純數值

```
1024
```

### string

目前無用

### 常用標籤

| 標籤名稱       | 描述     |
|------------|--------|
| `instance` | 節點名稱   |
| `job`      | 節點工作名稱 |

### Instant Vector Filter

``` 
node_cpu_seconds_total
# CPU使用時間
```

#### {} 條件過濾

可利用{}進行條件過濾

``` 
node_cpu_seconds_total{cpu="0"}
# 只顯示特定 CPU 核心資料

node_cpu_seconds_total{instance="test-node-linux"}
# 只顯示特定 node 資料，需要依靠 label 指定 node
```

| 運算符  | 描述      | 例子                                                        |
|------|---------|-----------------------------------------------------------|
| `=`  | 等於      | `node_cpu_seconds_total{cpu="0"}`                         |
| `!=` | 不等於     | `node_cpu_seconds_total{cpu!="0"}`                        |
| `=~` | 正規表達式   | `node_cpu_seconds_total{instance=~"test-node-[a-z]{1,}"}` |
| `!~` | 排除正規表達式 | `node_cpu_seconds_total{instance!~"test-node-[a-z]{1,}"}` |

Mode 為 `node_cpu_seconds_total` 的 arguments

| Mode      | 描述                          |
|-----------|-----------------------------|
| `guest`   | CPU 運行本VM中process時間         |
| `steal`   | CPU 被其他VM調度使用時間             |
| `user`    | CPU 運行用戶組態中,一般優先級process的時間 |
| `nice`    | CPU 運行用戶組態中,優先級較高process的時間 |
| `idle`    | CPU 空閒時間                    |
| `iowait`  | CPU 等待 I/O 時間               |
| `irq`     | CPU 硬體中斷時間                  |
| `softirq` | CPU 軟體中斷時間                  |
| `system`  | CPU 運行系統 kernel 程式代碼時間      |

#### [] 區間過濾

可利用[]進行區間過濾

``` 
node_cpu_seconds_total[5m]
# 5 分鐘內的資料
```

| 時間單位 | 描述 |
|------|----|
| `s`  | 秒  |
| `m`  | 分鐘 |
| `h`  | 小時 |
| `d`  | 天  |
| `w`  | 週  |
| `y`  | 年  |

#### offset 時間位移

``` 
node_cpu_seconds_total{cpu="0"} offset 5m
# 5 分鐘前的instant vector

node_cpu_seconds_total{cpu="0"}[5m] offset 3d
# 當前時間 3天前時間點 五分鐘內的 range vector
```

### binary operation 二元運算

可用於計算

- scalar/scalar = scalar
- vector/scalar = vector
- vector/vector = vector

| 運算符 | 描述  |
|-----|-----|
| `+` | 加   |
| `-` | 減   |
| `*` | 乘   |
| `/` | 除   |
| `%` | 取餘數 |
| `^` | 次方  |

``` 

e.g. 

1024/1024
# scalar/scalar 計算純量

node_memory_MemTotal_bytes/(1024/1024)
# vector/scala 計算記憶體大小 MB

(node_memory_MemAvailable_bytes/ node_memory_MemTotal_bytes) * 100
# vector/vector 計算記憶體使用率

```

### 關係運算

需注意 若在比較時帶上bool 輸出只會有 1 或 0  
表示 true 或 false

若是不用bool 會輸出符合條件的資料  
e.g.  
node_load 某時間點 是 3  
若用 > bool 2 會輸出 1  
若直接用 > 2 會輸出 數值3 , 若不符合就不會輸出

| 運算符  | 描述   |
|------|------|
| `==` | 等於   |
| `!=` | 不等於  |
| `>`  | 大於   |
| `<`  | 小於   |
| `>=` | 大於等於 |
| `<=` | 小於等於 |

### Scalar 之間握關係運算

需要用bool 結果只會輸出 0 或 1

``` 
2 > bool 1  # true
```

### vector 與 scalar 之間的關係運算

不使用bool 若結果不符合條件 則不會顯示該筆資料 若符合條件 則顯示該筆資料  
使用bool 若結果為 0 則不會顯示該筆資料 若是1 則顯示該筆資料

``` 
e.g. 

up != 0

node_load1 > 1
# 結果為 true 或 false
node_load1 > bool 1
# 結果為 true 或 false

up == 1
up != 1
```

### 集合運算

| 運算符      | 描述 |
|----------|----|
| `and`    | 並且 |
| `or`     | 或者 |
| `unless` | 排除 |

``` 

node_cpu_seconds_total{mode=~"idle|user|system"} and node_cpu_seconds_total{mode=~"system",cpu="0"}
# and 同時符合兩個條件

node_cpu_seconds_total{cpu="0"} or node_cpu_seconds_total{cpu="1"}
# or 符合其中一個條件

node_cpu_seconds_total unless node_cpu_seconds_total{cpu="1"} unless node_cpu_seconds_total{cpu="2"} unless node_cpu_seconds_total{cpu="3"} unless node_cpu_seconds_total{cpu="4"} unless node_cpu_seconds_total{cpu="5"} unless node_cpu_seconds_total{cpu="6"} unless node_cpu_seconds_total{cpu="7"} unless node_cpu_seconds_total{cpu="8"} unless node_cpu_seconds_total{cpu="9"} unless node_cpu_seconds_total{cpu="10"} unless node_cpu_seconds_total{cpu="11"} unless node_cpu_seconds_total{cpu="12"} unless node_cpu_seconds_total{cpu="13"} unless node_cpu_seconds_total{cpu="14"} unless node_cpu_seconds_total{cpu="15"}
# unless 排除條件, 這邊範例是把cpu 1-15 去掉 


probe_http_status_code <= bool 199 or probe_http_status_code >= bool 400
# status code <= 199 or >= 400 ,且用 1 or 0 輸出 true = 1 , false = 0

```

### 一般聚合函數

| 運算符          | 說明                                |
|--------------|-----------------------------------|
| sum          | 計算總和                              |
| min          | 計算最小值                             |
| max          | 計算最大值                             |
| avg          | 計算平均值                             |
| stddev       | 計算標準差                             |
| stdvar       | 計算標準變異數                           |
| count        | 計算資料筆數                            |
| count_values | 計算特定值的資料筆數                        |
| bottomk      | 計算最小的 k 個值 類似 order by  asc limit |
| topk         | 計算最大的 k 個值 類似 order by desc limit |
| quantile     | 計算分位數                             |

- without: 用於排除標籤
- by: 用於指定標籤

只排除或保留特定標籤達到特定效果 類似group

``` 
node_cpu_seconds_total
# 預設會有標籤: cpu, mode, instance, job

sum(node_cpu_seconds_total) without (cpu,job,mode) 
# 計算總和, 並排除 cpu,job,mode 標籤

上下相等 

sum(node_cpu_seconds_total) by (instance)
# 計算總和, 只使用 instance 標籤分組

# without by 皆不影響數值, 只差異在輸出標籤
```

max,avg

``` 

max(node_cpu_seconds_total) by (mode)
# 各cpu core 各mode 最大值  e.g. cpu0-15 的 idle 中的最大值

avg(node_cpu_seconds_total) by (mode)
# 各cpu core 各mode 平均值  e.g. cpu0-15 的 idle 中的平均值


# 可理解成groupby mode 再做聚合

e.g. 
avg by(instance) ( rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
# rate是指每秒的平均增長量 概念為 (最終值-初始值) / 區間秒數
# = 五分鐘內 CPU idle 每秒增長的平均值 * 100
# 每秒增長約 0.9x 再 * 100

```

count,topk,bottomk

``` 

count(prometheus_http_requests_total)
# prometheus localhost 被請求的數量

bottomk(5, sum(node_cpu_seconds_total) by (mode))
# order by mode 取總和 並輸出最小的5個

topk(5, sum(node_cpu_seconds_total) by (mode))
# order by mode 取總和 並輸出最大的5個

```

### 時間聚合函數

會將區間內的資料進行聚合運算

| 運算符                                      | 說明          |
|------------------------------------------|-------------|
 avg_over_time(range vector)              | 計算區間內的平均值   
 min_over_time(range vector)              | 計算區間內的最小值   
 max_over_time(range vector)              | 計算區間內的最大值   
 sum_over_time(range vector)              | 計算區間內的總和    
 count_over_time(range vector)            | 計算區間內的資料筆數  
 quantile_over_time(range vector, scalar) | 計算區間內的分位數   
 stddev_over_time(range vector)           | 計算區間內的標準差   
 stdvar_over_time(range vector)           | 計算區間內的標準變異數 

``` 
max_over_time(pg_stat_activity_max_tx_duration[5m])
# 5分鐘內pg 最大的事務執行時間

avg_over_time(probe_duration_seconds[5m])
# 5分鐘內的平均http probe 響應時間

min_over_time(node_timex_sync_status[1m])
# 1分鐘內最小的時間同步狀態 , 1表示正常, 0 表示無同步

```

### 向量批配

用於兩個瞬時向量運算

#### one to one

``` 
vector <operator> vector
```

該指標的label必須完全相同, 才能進行計算  
如前後都包含 instance,job, 若前有instance,job,後只有instance,則無法計算

``` 
process_open_fds
# 各節點各程序的當前描述符數

#process_open_fds{alias="cadvisor", instance="cadvisor:8080", job="cadvisor"} 11
#process_open_fds{instance="localhost:9090", job="prometheus"} 36
#process_open_fds{instance="test-node-linux", job="node1"} 9
#process_open_fds{instance="test-postgres-exporter", job="postgres-exporter"} 10


process_max_fds
# 各節點各程序的最大描述符數
process_max_fds{alias="cadvisor", instance="cadvisor:8080", job="cadvisor"} 1048576
process_max_fds{instance="localhost:9090", job="prometheus"} 1048576
process_max_fds{instance="test-node-linux", job="node1"} 1048576
process_max_fds{instance="test-postgres-exporter", job="postgres-exporter"} 1048576



process_open_fds{instance="test-node-linux",job="node1"} / process_max_fds{instance="test-node-linux",job="node1"}
# 顯示特定節點 node1這個程序的當前描述符數 / 最大描述符數   (這邊的job node1 實際上是node expoter) 
# {instance="test-node-linux", job="node1"} 0.00000858306884765625
```

on 用於指定標籤   
因向量運算需要相同標籤 若前後只有部份相同, 可用on 指定標籤

``` 
sum by (instance,job)(rate(node_cpu_seconds_total{mode="idle"}[5m]))
## 顯示各節點各job 5分鐘內的CPU idle 總和
## 會有instance,job 標籤
## 假設為A
## {instance="test-node-linux", job="node1"} 15.334278355940707

sum by (instance)(rate(node_cpu_seconds_total[5m]))
## 顯示各節點 5分鐘內的CPU idle 總和
## 會有instance 標籤
## 假設為B
{instance="test-node-linux"} 15.959228070173797

## 若要計算會有問題, 因為標籤不同 , 可用on 指定標籤
## A/ on(instance) B
sum by (instance,job)(rate(node_cpu_seconds_total{mode="idle"}[5m])) / on (instance) sum by (instance)(rate(node_cpu_seconds_total[5m]))
# {instance="test-node-linux"} 0.9698661327507738
# 其實就只是硬把不同標籤的輸出結果做運算
```

ignoring 用途同上 只是用於忽略標籤

```

sum by (instance,job)(rate(node_cpu_seconds_total{mode="idle"}[5m])) / ignoring (job) sum by (instance)(rate(node_cpu_seconds_total[5m]))
# 上方式 標注於 (instance,job)的instance
# 這邊是 忽略至 (instance,job)的job

```

#### one to many / many to one

``` 
vectors <operator> vectors
```

operator 有以下幾種

| Action | Label(s)            | Grouping    |
|--------|---------------------|-------------|
| ignore | label1, label2, ... | group_left  |
| ignore | label1, label2, ... | group_right |
| on     | label1, label2, ... | group_left  |
| on     | label1, label2, ... | group_right |

用以下範例說明

```
sum without (cpu) (rate(node_cpu_seconds_total[5m])) / ignoring (mode) group_left sum without (mode,cpu) (rate(node_cpu_seconds_total[5m]))  
```

輸出

| instance        | job   | mode    | value                  |
|-----------------|-------|---------|------------------------|
| test-node-linux | node1 | idle    | 0.9644908753948906     |
| test-node-linux | node1 | iowait  | 0.00010332645223131466 |
| test-node-linux | node1 | irq     | 0                      |
| test-node-linux | node1 | nice    | 0                      |
| test-node-linux | node1 | softirq | 0.003016253031092099   |
| test-node-linux | node1 | steal   | 0                      |
| test-node-linux | node1 | system  | 0.0061622137362622835  |
| test-node-linux | node1 | user    | 0.026227331385523776   |

左測

``` 
sum without (cpu) (rate(node_cpu_seconds_total[5m]))
```

輸出

| instance        | job   | mode    | value                 |
|-----------------|-------|---------|-----------------------|
| test-node-linux | node1 | idle    | 15.393578947367342    |
| test-node-linux | node1 | iowait  | 0.0016491228070176396 |
| test-node-linux | node1 | irq     | 0                     |
| test-node-linux | node1 | nice    | 0                     |
| test-node-linux | node1 | softirq | 0.04814035087718387   |
| test-node-linux | node1 | steal   | 0                     |
| test-node-linux | node1 | system  | 0.09835087719297197   |
| test-node-linux | node1 | user    | 0.4185964912281059    |

右側

``` 
 sum without (mode,cpu) (rate(node_cpu_seconds_total[5m]))  
```

輸出

| instance        | job   | value              |
|-----------------|-------|--------------------|
| test-node-linux | node1 | 15.959228070173797 |

連接

``` 
ignoring (mode) group_left
```

group_left 會以左側標籤輸出  
ignoring 因為需相同標籤才能做計算 , 忽略 mode 標籤
數值上就是直接左側 / 右側

### 數學函數

#### abs() 絕對值

``` 
abs(process_open_fds-100)
```

#### sqrt() 平方根

``` 
sqrt(process_open_fds)
```

#### round() 四捨五入

``` 
round(sqrt(process_open_fds))
```

### 時間函數

#### time() 當前時間, 單位 timestamp

``` 
time()
```

程式啟動多久 單位: 小時

``` 
(time() - process_start_time_seconds ) / (3600)
```

### 瞬時函數

#### rate()

區間常用函數 rate , 常用於gauge
rate 會計算區間內的每秒平均變化量 簡單說為 (最終值-初始值) / 區間秒數

``` 
rate(metric_name[rate_interval])
```

``` 
probe_duration_seconds[1m]
# 1分鐘內的資料 預設為 15s 一筆, 因此會回傳四筆資料

rate(probe_duration_seconds[1m])
# 會將所有回傳/秒 , 會將四筆中第一筆與最後一筆差 / 經過秒數
# 1m 不一定是60s , 它會用你計算當下一分鐘內 採集樣本最接近的時間點

e.g. 左側數值 右側時間搓 , 可發現時間搓第一與最後差 75-30=45s , 因此會用 45s 來計算
1426938.43 @1703001330.949
1426952.33 @1703001345.949
1426966.39 @1703001360.949
1426980.95 @1703001375.949
>>> (1426980.95 - 1426938.43)/45
0.9448888888893028

```

#### increase()

區間函數increase , 常用於counter   
increase 會計算區間內的增長量, 簡單說就是 (最終值-初始值)   
但實際上不會這麼剛好, 會受到採集率影響, 若1m內, 採集樣本差只到45s, 會用推估的方式補到1m的時間長度
就意義上 increase() / 時間長度 = rate()
需注意 因輸出為平均結果, 無法看出瞬間數值

``` 
increase(metric_name[increase_interval])
```

``` 
increase(node_cpu_seconds_total{mode="idle"}[1m])
# 1分鐘內的數值變化量 最終值-初始值

increase(node_cpu_seconds_total{cpu="0",mode="idle"}[1m]) / 60
等同
rate(node_cpu_seconds_total{cpu="0",mode="idle"}[1m])

```

#### irate()

與rate()類似, 但會用踩集時間內倒數最後兩筆資料計算, 相較rate()更靈敏, 可凸顯出極值

``` 
probe_duration_seconds[1m]
0.001598031 @1703699907.838
0.002492671 @1703699922.838
0.001710222 @1703699937.838 <<
0.003617856 @1703699952.838 <<
# 近一分鐘內採集的資料樣本, 設定間隔為15s

irate(probe_duration_seconds[1m])
0.0001271756
# (0.003617856 - 0.001710222)/15  = 0.0001271756
# 採集間隔為15s 簡單說 就是計算出 最小單位的瞬時變化


```

### 趨勢函數

#### predict_linear()  

範例對象說明, 利用磁碟可用空間為例

```promQL
node_filesystem_avail_bytes

node_filesystem_avail_bytes{device="/dev/nvme0n1p1", fstype="vfat", instance="test-node-linux", job="node1", mountpoint="/boot/efi"}
502886400
node_filesystem_avail_bytes{device="/dev/nvme0n1p5", fstype="ext4", instance="test-node-linux", job="node1", mountpoint="/"}
168449896448
node_filesystem_avail_bytes{device="jetbrains-toolbox", fstype="fuse.jetbrains-toolbox", instance="test-node-linux", job="node1", mountpoint="/tmp/.mount_jetbraLovDpD"}
0
node_filesystem_avail_bytes{device="ramfs", fstype="ramfs", instance="test-node-linux", job="node1", mountpoint="/run/credentials/systemd-sysusers.service"}
0
node_filesystem_avail_bytes{device="tmpfs", fstype="tmpfs", instance="test-node-linux", job="node1", mountpoint="/run"}
1610575872
node_filesystem_avail_bytes{device="tmpfs", fstype="tmpfs", instance="test-node-linux", job="node1", mountpoint="/run/lock"}
5238784
node_filesystem_avail_bytes{device="tmpfs", fstype="tmpfs", instance="test-node-linux", job="node1", mountpoint="/run/user/1001"}
1612759040

# device是硬碟分區的意思 , 一顆硬碟安裝時 可能會被分成多個磁區  
# fstype 是該磁區格式化時的種類
# instance 節點名稱
# job 任務名稱

node_filesystem_avail_bytes{fstype!="tmpfs"}[1h]
## 1 hr 內 , fstype != tmpfs , 的所有採集資料 , 輸出單位為byte timestamp, 表示當前時間點可使用空間  , 會有多筆資料

```

```bash
df -lh

Filesystem      Size  Used Avail Use% Mounted on
tmpfs           1.6G  2.3M  1.5G   1% /run
/dev/nvme0n1p5  323G  150G  157G  49% /
tmpfs           7.6G  187M  7.4G   3% /dev/shm
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
/dev/nvme0n1p1  511M   32M  480M   7% /boot/efi
tmpfs           1.6G  112K  1.6G   1% /run/user/1001
```

```
predict_linear(node_filesystem_avail_bytes{fstype!="tmpfs"}[1h], 24*3600)
## 利用前一小時的增長率, 預測往後24hr 的增長
```

alert 應用

``` 

((node_filesystem_avail_bytes * 100 ) / node_filesystem_size_bytes) < 10 and ON (instance,device, mountpoint) predict_linear(node_filesystem_avail_bytes[1h], 24*3600) < 0 and ON (instance,device, mountpoint) node_filesystem_readonly == 0
# 需符合以下條件
# 當前磁碟可用空間率 < 10%
# 預測24hr 後可用空間會小於0
# 且該磁區非唯讀磁區


```

### 標籤操作函數

#### label_replace()

主要用途是用當前特定標籤數值 新增一個新標籤

使用範例, 使用 up==1

``` 
up == 1
## 原輸出

up{instance="blackbox-exporter:9115", job="google_http2xx_probe"}
1
up{instance="blackbox-exporter:9115", job="prometheus_http2xx_probe"}
1
up{instance="localhost:9090", job="prometheus"}
1
up{instance="test-node-linux", job="node1"}
1

```

``` 

label_replace(up==1, "host", "$1", "instance", "(.*):.*")
# up==1 會輸出 1 或 0 , 1表示true, 0表示false
# instance 後面規則為regex , 批配字元中含有 : ,的字串
# instance 標籤數值符合 regex ""(.*):.*" , 會被當成新編籤host的輸出值,$1 表示regex 的第一組回傳值 


up{host="blackbox-exporter", instance="blackbox-exporter:9115", job="google_http2xx_probe"}
1
up{host="blackbox-exporter", instance="blackbox-exporter:9115", job="prometheus_http2xx_probe"}
1
up{host="localhost", instance="localhost:9090", job="prometheus"}
1
up{instance="test-node-linux", job="node1"}

```

### 多變數標籤操作

``` 
label_replace(up==1, "host", "$1 -- $2", "instance", "(.*):(.*)")
## $1 $2 表示regex 的 第一組 第二組 回傳值

up{host="blackbox-exporter -- 9115", instance="blackbox-exporter:9115", job="google_http2xx_probe"}
1
up{host="blackbox-exporter -- 9115", instance="blackbox-exporter:9115", job="prometheus_http2xx_probe"}
1
up{host="localhost -- 9090", instance="localhost:9090", job="prometheus"}
1
up{instance="test-node-linux", job="node1"}
```

#### label_join()

顛單說就是連接當前存在的標籤, 再一個新標籤輸出

若但若用不存在的標籤 則不作用

``` 
label_join(up==1, "new_join_label", "-", "instance", "job")


up{instance="blackbox-exporter:9115", job="google_http2xx_probe", new_join_label="blackbox-exporter:9115-google_http2xx_probe"}
1
up{instance="blackbox-exporter:9115", job="prometheus_http2xx_probe", new_join_label="blackbox-exporter:9115-prometheus_http2xx_probe"}
1
up{instance="localhost:9090", job="prometheus", new_join_label="localhost:9090-prometheus"}
1
up{instance="test-node-linux", job="node1", new_join_label="test-node-linux-node1"}

```

其他可參考  
[prometheus document](https://prometheus.io/docs/prometheus/latest/querying/functions/)  
[prometheus 中文文檔](https://prometheus.fuckcloudnative.io/di-san-zhang-prometheus/di-4-jie-cha-xun/functions)

### 補充說明

``` 
((node_filesystem_avail_bytes * 100 ) / node_filesystem_size_bytes)
## 磁碟可用空間率 (%)
```

``` 
ON (instance,device, mountpoint)
# 指定對應標籤的資料才運算
```

``` 
predict_linear(node_filesystem_avail_bytes[1h], 24*3600)
## 利用前一小時的增長率, 預測往後24hr 的增長
## 輸出單位為byte timestamp, 表示當前時間點可使用空間  , 會有多筆資料
```

``` 
                                                                   
predict_linear(node_filesystem_avail_bytes[1h], 24*3600)
```

``` 
node_filesystem_readonly
# 表示該磁碟是否為唯讀 , 0 = false, 1 = true
```

``` 
deriv(node_memory_MemAvailable_bytes[5m])
# deriv()
# 只能gauge類型, 返回一個瞬時向量, 使用簡單線性回歸計算區 單位時間內的變化量
```

