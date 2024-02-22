---
title: "[筆記] Prometheus PromQL 範例"
categories:
  - 筆記
tags:
  - Prometheus
toc: true
toc_label: Index
mermaid: true
---

# PromQL 範例  


## cadvisor  

### 檢測容器是否超過5分鐘沒有連線

```
time() - container_last_seen > 300 
```
當前時間戳 - 容器最後連線時間戳 超過300s  

<br/> 

### 檢測容器是否全部停止

```
absent(container_last_seen) 
```

absent函數用於 檢測輸入直有無返回向量, 若有返回empty, 若沒有則是返回 {} = 1
這邊表示container_last_seen 沒有返回向量, 也就是沒有容器在運行
表示服務已經全部都停止了  
  
<br/>

### 檢視 過去三分鐘內 各容器每秒平均調用CPU core是否超過300% 

```
(sum(rate(container_cpu_usage_seconds_total{name!=""}[3m])) BY (instance, name) * 100) > 300   
```

container_cpu_usage_seconds_total 是 counter類型, 代表各容器使用量統計,各CPU Core分開統計, 表示一個容器會有多組數值       
{name!=""} label 取 name != "", 可以說是所有存在容器   
[3m] 三分鐘內採集的所有樣本點, 各容器 各CPU Core 三分鐘內的所有樣本點   
rate 取以上時間平均變化量, 三分鐘內 各容器 (末直-初始直)/經過秒數, 一個容器會有CPU Core數量的樣本點 , 表示 各CPU平均每秒 被該容器調度多少秒, 也可理解成各核心被該容器每秒平均使用率      
sum BY (instance, name) 用instance,name 同label進行 總和, 表示同一台主機上 各容器 CPU各核心使用率總和  
*100 表示轉為%     
value >300 表示特定容器占用滿 CPU 3core 以上的資源
<br/>

### 容器文件系統 IO使用量   

```
(sum(container_fs_io_current{name!=""}) BY (instance, name) * 100) > 80
```
container_fs_io_current 是 gauge類型, 代表各容器文件系統 IO使用量, 每個容器依照性質, 有可能有多個輸出, 如有持續讀取的業務, 
{name!=""} label 取 name != "", 可以說是所有存在容器    
sum BY (instance, name) 用instance,name 同label進行 總和, 表示同一台主機上 各容器 IO使用量總和 
*100 待釐清,  80 也是  
<br/>

### 檢視 各容器記憶體當前使用量是否超過可使用量的80%

```
(sum(container_memory_working_set_bytes{name!=""}) BY (instance, name) / sum(container_spec_memory_limit_bytes > 0) BY (instance, name) * 100) > 80  
```
container_memory_working_set_bytes 是gauge類型, 代表個容器記憶體使用量, 各容器只會有一個輸出  
{name!=""} label 取 name != "", 可以說是所有存在容器  
sum BY (instance, name) 用instance,name 同label進行 總和, 表示同一台主機上 各容器 記憶體使用量總和  
<br/>
container_spec_memory_limit_bytes 表示容器記憶體可使用上限  
sum(container_spec_memory_limit_bytes > 0) BY (instance, name) 用instance,name 同label進行 總和, 表示同一台主機上 各容器 記憶體可使用上限總和  
<br/>
100 *　使用量/可使用量 , 為百分比  
<br/>  


## node_exporter  

### 檢測 主機記憶體可用量是否小於10% 

```
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10   
```
node_memory_MemAvailable_bytes 是gauge類型, 代表主機記憶體可用量, 只會有一個輸出  
node_memory_MemTotal_bytes 是gauge類型, 代表主機記憶體總量, 只會有一個輸出  
100 *　使用量/可使用量 , 為百分比  
**檢測 主機記憶體可用量是否小於10%**  
<br/>

### 檢測 各主機2m內網路流入量是否大於100M*
```
sum by (instance) (rate(node_network_receive_bytes_total[2m])) / 1024 / 1024 > 100
```
node_network_receive_bytes_total 網路流入 bytes 統計, counter類型, 會有兩個輸出, 一為網路介面, 一為內部虛擬網路 
[2m] 表示兩分鐘內所有採集點  
sum by (instance) 用instance label進行聚合, 簡單說計算出特定主機的總和  
/1024 = KB , 再/1024 = MB   
大於 100 表示網路流入速度超過 100MB/s    
<br/>  

### 檢測 各主機2m內網路流出量是否大於100M
```
sum by (instance) (rate(node_network_transmit_bytes_total[2m])) / 1024 / 1024 > 100
```
node_network_transmit_bytes_total 網路流出 bytes 統計, counter類型, 會有兩個輸出, 一為網路介面, 一為內部虛擬網路 
[2m] 表示兩分鐘內所有採集點  
sum by (instance) 用instance label進行聚合, 簡單說計算出特定主機的總和  
/1024 = KB , 再/1024 = MB   
大於 100 表示網路流出速度超過 100MB/s    
<br/>  

### 檢測 各主機2m內硬碟讀取量是否大於50M* 
```
sum by (instance) (rate(node_disk_read_bytes_total[2m])) / 1024 / 1024 > 50
```
node_disk_read_bytes_total 磁碟讀取 bytes 統計, counter類型, 會有多個輸出, 表示多個磁碟分區   
[2m] 表示兩分鐘內所有採集點  
sum by (instance) 用instance label進行聚合, 簡單說計算出特定主機讀取的總和  
/1024 = KB , 再/1024 = MB  
<br/>

### 檢測 各主機2m內硬碟寫入量是否大於50M 

```
sum by (instance) (rate(node_disk_written_bytes_total[2m])) / 1024 / 1024 > 50
```

node_disk_written_bytes_total 磁碟寫入 bytes 統計, counter類型, 會有多個輸出, 表示多個磁碟分區   
[2m] 表示兩分鐘內所有採集點  
sum by (instance) 用instance label進行聚合, 簡單說計算出特定主機寫入的總和  
/1024 = KB , 再/1024 = MB  
<br/>

### 檢測 主機連線數量是否超過上限的80%* 
```
node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8
```
node_nf_conntrack_entries 是gauge類型, 代表linux kernel Netfilter框架的當前活動連線數, 只會有一個輸出   
node_nf_conntrack_entries_limit 是gauge類型, 表示linux kernel Netfilter框架的連線數處理上限, 只會有一個輸出    
<br/>

### 檢測 過去一分鐘NTP同步狀態,與最大誤差是否大於等於10秒* 
```
min_over_time(node_timex_sync_status[1m]) == 0 and node_timex_maxerror_seconds >= 10
```
node_timex_sync_status 是gauge類型, 代表linux kernel NTP 是否有與可信賴的時間伺服器同步, 1 yes, 0 no, 只會有一個輸出  
[1m] 表示一分鐘內所有採集點,  
min_over_time 表示一組 區間向量 的最小值,  
node_timex_maxerror_seconds 是gauge類型, 代表linux kernel NTP 與可信賴的時間伺服器最大誤差秒數, 只會有一個輸出  
<br/>

### 檢測 當前檔案描述符使用量是否超過上限的80%*  
```
node_filefd_allocated / node_filefd_maximum * 100 > 80  
```
node_filefd_allocated 是gauge類型, 代表linux kernel 當前檔案描述符使用量, 只會有一個輸出  
node_filefd_maximum 是gauge類型, 代表linux kernel 檔案描述符使用上限, 只會有一個輸出
100 *　使用量/可使用量 , 為百分比  
<br/>

### 檢測 過去一分鐘內各磁區平均IO時間是否大於0.1秒

```
increase(node_disk_read_time_seconds_total[1m]) /  increase(node_disk_reads_completed_total[1m]) > 0.1 and increase(node_disk_reads_completed_total[1m]) > 0
```
node_disk_read_time_seconds_total 是counter類型, 代表linux kernel 磁碟讀取時間統計, 會有多個輸出, 表示多個磁碟分區  
[1m] 表示一分鐘內所有採集點
increase 表示一組 區間向量 的增量, 計算該時間區間中, 總共使用多少秒該硬碟
<br/>
node_disk_reads_completed_total 是counter類型, 代表linux kernel 磁碟讀取次數統計, 會有多個輸出, 表示多個磁碟分區   
[1m] 表示一分鐘內所有採集點  
increase 表示一組 區間向量 的增量, 計算該時間區間中, 總共讀取多少次該硬碟  
<br/>
單位時間/單位次數 = 多少秒/次 = 每次硬碟平均IO時間
<br/>
increase(node_disk_reads_completed_total[1m]) > 0
且該段時間內 磁區是有被讀取過    
<br/>

### 檢測 過去兩分鐘內各主機CPU 使用率是否大於80% 

```
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80  
```
node_cpu_seconds_total 是counter類型, 代表linux kernel CPU使用量統計, 會有多個輸出(instance,cpu,mode), 表示各主機 多個CPU Core    
{mode="idle"} label 取 mode = "idle", 可以說是所有閒置CPU Core  
[2m] 表示兩分鐘內所有採集點  
rate 取以上時間平均變化量, 兩分鐘內 各CPU Core (末直-初始直)/經過秒數 , 表示 各CPU Core平均每秒 閒置多少秒, 也可理解成各核心被該容器每秒平均閒置率  
avg by(instance) 用instance label進行聚合, 簡單說計算出特定主機的平均值   
*100 表示轉為% 
<br/>  
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80   
表示CPU Core 非閒置時間 > 80%   
<br/>

### 檢測 過去五分鐘內各主機CPU 被其他VM調度使用率是否大於10%* 

```
avg by(instance) (rate(node_cpu_seconds_total{mode="steal"}[5m])) * 100 > 10  
```
node_cpu_seconds_total 是counter類型, 代表linux kernel CPU使用量統計, 會有多個輸出(instance,cpu,mode), 表示各主機 多個CPU Core    
{mode="steal"} label 取 mode = "steal", steal表示其他VM在使用該CPU Core  
[5m] 表示五分鐘內所有採集點   
rate 取以上時間平均變化量, 五分鐘內 各CPU Core (末直-初始直)/經過秒數, 表示 各CPU Core平均每秒 被調度多少秒, 也可理解成各核心被該容器每秒平均使用率   
avg by(instance) 用instance label進行聚合, 簡單說計算出特定主機的平均值   
*100 表示轉為%
<br/>

### 檢測 過去兩分鐘各網路介面接收封包錯誤率是否大於 1%  
```
100* (rate(node_network_receive_errs_total[2m]) /rate(node_network_receive_packets_total[2m])) > 1
```
node_network_receive_errs_total 是counter類型, 代表接收封包錯誤統計, 會有多個輸出, 表示多個網路介面  
node_network_packets_total 是counter類型, 代表接收封包統計, 會有多個輸出, 表示多個網路介面    
[2m] 表示兩分鐘內所有採集點  
rate(node_network_receive_errs_total[2m]) /rate(node_network_receive_packets_total[2m]) 表示錯誤封包數/總封包數 = 錯誤率  
*100 表示轉為%
<br/>

### 檢測 過去兩分鐘內各網路介面送出封包錯誤率是否大於 1% 
```
100*(rate(node_network_transmit_errs_total[2m]) / rate(node_network_transmit_packets_total[2m])) > 1  
```
node_network_transmit_errs_total 是counter類型, 代表發送封包錯誤統計, 會有多個輸出, 表示多個網路介面  
node_network_transmit_packets_total 是counter類型, 代表發送封包統計, 會有多個輸出, 表示多個網路介面      
[2m] 表示兩分鐘內所有採集點    
rate(node_network_transmit_errs_total[2m]) / rate(node_network_transmit_packets_total[2m]) 表示錯誤封包數/總封包數 = 錯誤率   
*100 表示轉為%  
<br/>

### 檢測 當前 Netfilter conntrack 是否超過上限值的 80%*
```
node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8
```
Netfilter是實現防火牆的Linux內核模組, 當封包傳入/出防火牆,  nf_conntrack 會追蹤這些封包連接訊息(內外部網路轉換,防火牆層...),  
node_nf_conntrack_entries 表示當前追蹤項目數量, 只會有一個輸出  
node_nf_conntrack_entries_limit 表示追蹤項目數量上限, 只會有一個輸出  
node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8 表示conntrack數量超過上限的80%    
<br/>

## blackbox_exporter

### 檢測 ssl證書最早到期時間是否小於14天
```
probe_ssl_earliest_cert_expiry - time() < 86400 * 14
```
probe_ssl_earliest_cert_expiry 是gauge類型, 代表ssl證書最早到期時間, 只會有一個輸出, 為timestamp  
time() 當前時間戳  
86400*14 ,14 天的秒數  
<br/>

### 檢測 過去五分鐘內http探測耗時是否大於0.5秒
```
avg_over_time(probe_http_duration_seconds[5m]) > 1s
```
probe_http_duration_seconds 是gauge類型, 代表http探測耗時, 只會有一個輸出  
[5m] 表示五分鐘內所有採集點  
avg_over_time 表示一組 區間向量 的平均值  
<br/>

### 檢測 http探測是否成功
```
probe_http_success == 0
```
probe_http_success 是gauge類型, 代表http探測是否成功, 1 yes, 0 no, 只會有一個輸出  
<br/>

### 檢測 http探測返回的狀態碼是否小於等於199 或 大於等於400
```
probe_http_status_code <= 199 OR probe_http_status_code >= 400  
```
probe_http_status_code 是gauge類型, 代表http探測返回的狀態碼, 只會有一個輸出   
<br/>


## postgres_exporter

### 檢測 各資料庫當前連線數是否超過該主機最大連線數的80%
```
sum by (datname,instance) (pg_stat_activity_count{datname!~"template.*|postgres"}) > on (instance) (sum by (instance) (pg_settings_max_connections)*0.8 )  
```
pg_stat_activity_count 是gauge類型, 代表當前活動連線數, 會有多個輸出, 表示多個資料庫  
{datname!~"template.*|postgres"} label 取 datname 不包含 template.*|postgres, 可以說是所有非系統資料庫  
sum by (datname,instance) 用datname,instance label進行聚合, 簡單說計算出特定主機的連線數總和  
on (instance) 表示限定label, 因前後ㄒ樣label不同, 唯一重疊為 instance, 需相同label才能進行運算, 這邊主要是指定label  
on (instance) (sum by (instance) (pg_settings_max_connections)*0.8 ) 跟最大值*0.8 作為threshold    
<br/>

### 檢測 過去五分鐘內所有節點各資料庫每秒平均transaction數量是否大於3000
```
avg(irate(pg_stat_database_xact_commit{datname!~"template.*"}[5m]) + irate(pg_stat_database_xact_rollback{datname!~"template.*"}[5m])) by (datname) > 3000   
```
template0,為系統內建的預設系統template, 此資料庫不可被修改 , template1 為可自訂template資料庫, 初始時 兩資料庫都相同   
pg_stat_database_xact_commit{datname!~"template.*"}[5m], 5分鐘內 非template資料庫中的所有commit成功的transaction, 有多組輸出,為每個資料庫的commit   
irate(pg_stat_database_xact_commit{datname!~"template.*"}[5m]) ,跟rate類似 但irate是計算所有採集點的倒數最後兩筆,相較rate 會更敏感, 容易顯示極值, 計算每秒內平均commit數量    
pg_stat_database_xact_rollback{datname!~"template.*"}[5m], 5分鐘內 非template資料庫中的所有被rollback的transaction, 有多組輸出,為每個資料庫的rollback     
irate(pg_stat_database_xact_rollback{datname!~"template.*"}[5m])), 跟rate類似 但irate是計算所有採集點的倒數最後兩筆,相較rate 會更敏感, 容易顯示極值, 計算每秒內平均rollback數量    
以上兩者相加 表示5分鐘內 非template資料庫中的所有成功與失敗的commit, 每秒時間內transaction總數量 , 有多組輸出,為每個資料庫的每秒平均transaction    
avg(irate(pg_stat_database_xact_commit{datname!~"template.*"}[5m]) + irate(pg_stat_database_xact_rollback{datname!~"template.*"}[5m])) by (datname) 用datname label進行聚合, 表示計算同一資料庫 (若多節點, 每個節點都會有相同名稱資料庫,單一節點沒差)  
<br/>

## redis_exporter  

### 檢測 當前連線數是否大於1000 
```
redis_connected_clients > 1000    
```
redis_connected_clients 是gauge類型, 代表當前連線數, 只會有一個輸出  
<br/>

### 檢測 當前記憶體使用量是否大於2GB
```
redis_memory_used_bytes > 2048\*1000\*1000    
```
redis_memory_used_bytes 是gauge類型, 代表當前記憶體使用量, 只會有一個輸出  
2048*1000*1000 bytes = 2GB  
<br/>  



