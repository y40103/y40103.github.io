---
title: "[筆記] EC2 Service Discovery and Relabel 基礎使用"
categories:
  - 筆記
tags:
  - Prometheus
toc: true
toc_label: Index
mermaid: true
---

Prometheus ec2 服務發現 與 relabel 用法

## autoscaling service discovery

ec2_sd_configs 為 ec2 service discovery 的設定模塊

access_key, secret_key 必須確保需要有 `ec2:DescribeInstance` 的權限

```
scrape_configs:
  - job_name: 'ec2-autoscaling'
    ec2_sd_configs:
      - region: ap-northeast-1    # 您的 AWS 區域
        access_key: ''    # AWS 訪問金鑰
        secret_key: ''    # AWS 秘密金鑰
        port: 9100    # 設定該主機exppoter預設port, 目前測試並實際功能, 還是需要重寫 __address__ 自己加上port, 才會找的到exporter  
        filters:
          - name: tag:prometheus
            values:
              - enabled
              
    relabel_configs:
     - target_label: __metrics_path__
       replacement: /metrics   # 默認的 Prometheus 指標路徑
     - source_labels: [__meta_ec2_private_ip]
       regex: '(.*)'
       replacement: '${1}:9100'
       target_label: __address__
```

filters 是會抓取 ec2 tag 含有 prometheus:enabled 的 instance

## relabel

relabel_configs 主要是重寫標籤的數值,

- target_label: 這邊是輸入標籤名稱, 主要是定位標籤, 下面得action 決定對該label的名稱或數值 做操作  (通常是數值)
- replacement: 這邊是取代該定位標籤的數值

這邊意思是 把 __metrics_path_ 的 label name , value部份替換成 /metrics

```
- target_label: __metrics_path__
  replacement: /metrics   # 默認的 Prometheus 指標路徑
```

- source_labels: 主要是用來新增 label , 這邊主要用途是抓取 該 list 中 label的數值 來做組合

__meta_ec2_instance_id  __meta_ec2_instance_type 為兩 label name, 會將這兩 label name的 數值 進行組合
利用target_label 定位到 instance這個label, 數值會用 `${1}-${2}` 來組合 = `數值1-數值2`

```
relabel_configs:
  - source_labels: [__meta_ec2_instance_id, __meta_ec2_instance_type]
    target_label: instance
    replacement: "${1}-${2}"
```

- regex: 主要用來批配 source_labels 抓到的 label value, 如果批配的話 才會被拿來使用

以下是若  __meta_ec2_instance_state 為 running 的情況, 會新增一個 label , instance_state=running

```
relabel_configs:
  - source_labels: [__meta_ec2_instance_state]
    target_label: instance_state
    regex: "running"
    action: keep
```

以下是 將 meta_ec2_private_ip 的value 抓出來, 並用regex批配,   
這邊是 取任意數值都批配 目的是藉由 regex 抓出數值,  
在組合常數 利用 replacement 替換 __address__ 這個 label name的 數值   
最後輸出範例 可能會是  __address__: 10.0.2.12:9100

```
- source_labels: [__meta_ec2_private_ip]
  regex: '(.*)'
  replacement: '${1}:9100'
  target_label: __address__
```

以下為 抓取的instance 新增label, 因prometheus 新增標籤 會將原instance複製一份 再加上新 label, 若使用action: replacement, 會直接新增在原instance上  
```
- target_label: node_name
  replacement: test-playground
  action: replacement
```

### meta label

一般 instance 基本的 meta label

| 標籤名稱                             | 描述                                  | 範例值                              |
|----------------------------------|-------------------------------------|----------------------------------|
| `__address__`                    | Prometheus 用來抓取指標的目標地址（主機:端口）       | `10.0.0.1:9100`                  |
| `__meta_<service_name>_<label>`  | 服務發現的元數據（例如 EC2、Kubernetes）         | `__meta_ec2_instance_id=abcd123` |
| `__metrics_path__`               | 抓取指標的路徑（默認為 `/metrics`）             | `/metrics`                       |
| `__param_<param_name>`           | 在抓取配置中添加的 URL 參數                    | `__param_job=myjob`              |
| `__scheme__`                     | 抓取時使用的協議（`http` 或 `https`）          | `http`                           |
| `__meta_<instance_name>_<label>` | 服務發現中的實例級別元數據（例如 Kubernetes pod 名稱） | `__meta_k8s_pod_name=my-pod`     |

### ec2 sd config meta label

ec2 service discovery instance meta label

| 標籤名稱                                | 說明                                                   |
|-------------------------------------|------------------------------------------------------|
| `__meta_ec2_ami`                    | EC2 的 Amazon Machine Image（AMI）。                     |
| `__meta_ec2_architecture`           | 實例的架構。                                               |
| `__meta_ec2_availability_zone`      | 實例所在的可用區域。                                           |
| `__meta_ec2_availability_zone_id`   | 實例所在的可用區域 ID（需要 `ec2:DescribeAvailabilityZones` 權限） |
| `__meta_ec2_instance_id`            | EC2 實例的 ID                                          |
| `__meta_ec2_instance_lifecycle`     | EC2 實例的生命周期，只對 "spot" 或 "scheduled" 實例設置，對其他實例無此標籤  |
| `__meta_ec2_instance_state`         | EC2 實例的狀態                                           |
| `__meta_ec2_instance_type`          | EC2 實例的類型                                           |
| `__meta_ec2_ipv6_addresses`         | 實例網路介面分配的 IPv6 地址列表（逗號分隔），如果存在                      |
| `__meta_ec2_owner_id`               | 擁有 EC2 實例的 AWS 帳號 ID                                |
| `__meta_ec2_platform`               | 操作系統平台，對於 Windows 伺服器設置為 "windows"，其他情況下不設置此標籤      |
| `__meta_ec2_primary_ipv6_addresses` | 實例的主 IPv6 地址列表（逗號分隔），如果存在。列表根據每個對應網路介面在附加順序中的位置排列   |
| `__meta_ec2_primary_subnet_id`      | 主網路介面的子網 ID（如果可用）                                   |
| `__meta_ec2_private_dns_name`       | 實例的私有 DNS 名稱（如果可用）                                  |
| `__meta_ec2_private_ip`             | 實例的私有 IP 地址（如果存在）                                   |
| `__meta_ec2_public_dns_name`        | 實例的公共 DNS 名稱（如果可用）                                  |
| `__meta_ec2_public_ip`              | 實例的公共 IP 地址（如果可用                                   |
| `__meta_ec2_region`                 | 實例所在的區域                                             |
| `__meta_ec2_subnet_id`              | 實例所在的子網 ID 列表（逗號分隔，如果可用）                            |
| `__meta_ec2_tag_<tagkey>`           | 實例的每個標籤值                                            |
| `__meta_ec2_vpc_id`                 | 實例所在 VPC 的 ID（如果可用）                                 |
