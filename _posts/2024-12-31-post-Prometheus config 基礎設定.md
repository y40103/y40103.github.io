---
title: "[筆記] Prometheus config 基礎設定"
categories:
  - 筆記
tags:
  - Prometheus
toc: true
toc_label: Index
mermaid: true
---


簡單紀錄主設定檔目前有用到的區塊   

/etc/prometheus/prometheus.yml  

### 區塊說明

- global: 該區塊定義全局設定  
  - scrape_interval: 資料採集的頻率, 如每15秒採一次  
  - evaluation_interval: 評估alert rule 的頻率，如每15秒評估一次  


- alerting: 該區塊定義 alert 相關設定  
  - alertmanagers: alertmanager 相關設定  
    - static_configs: alertmanager 的靜態主機設定  
      - targets: alertmanager 主機, host:port , 若prometheus 的metrics超過 警戒數值 就會推送給alertmanager 發送通知  

- rule_files: 該區塊定義 alert rule 檔案路徑, 超過警戒數值 就會推送給alertmanager 發送通知   

- scrape_configs: 該區塊定義 資料採集相關設定  

- scrape_config_files: 該區塊定義 scrape_configs include 檔案, 可將列表中 符合的檔案路徑檔案中 scrape_configs 內容抓進來   

### 設定檔範例

```
global:
  scrape_interval: 15s 
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "alert.yaml"
  - "rules/*.yaml"

scrape_config_files:
  - /etc/prometheus/scrape_configs/*.yaml
  - /etc/prometheus/scrape_configs/*.yml

scrape_configs:
  - job_name: 'monitor-node'
    ec2_sd_configs:
      - region: ap-northeast-1    # AWS 區域
        access_key: ''    # AWS 訪問金鑰
        secret_key: ''    # AWS 秘密金鑰
        port: 9100    # EC2 實例暴露的指標端口 同dockerfile expose , 無實際用途
        filters:
          - name: tag:Name
            values:
              - monitor
    relabel_configs:
      - target_label: __metrics_path__
        replacement: /metrics   # 默認的 Prometheus 指標路徑

      - source_labels: [__meta_ec2_private_ip]
        regex: '(.*)'
        replacement: '${1}:9100'
        target_label: __address__

      - source_labels: [ __meta_ec2_private_ip ]
        regex: '(.*)'
        target_label: instance
        replacement: $1

      - target_label: instance_tag
        replacement: monitor-instance
        action: replace

  - job_name: 'monitor-cadvisor'
    ec2_sd_configs:
      - region: ap-northeast-1    # 您的 AWS 區域
        access_key: ''    # AWS 訪問金鑰
        secret_key: ''    # AWS 秘密金鑰
        port: 8080
        filters:
          - name: tag:Name
            values:
              - monitor

    relabel_configs:
      - target_label: __metrics_path__
        replacement: /metrics   # 默認的 Prometheus 指標路徑

      - source_labels: [__meta_ec2_private_ip]
        regex: '(.*)'
        replacement: '${1}:8080'
        target_label: __address__

      - source_labels: [ __meta_ec2_private_ip ]
        regex: '(.*)'
        target_label: instance
        replacement: $1

      - target_label: instance_tag
        replacement: monitor-instance
        action: replace
```

### scrape_config_files 範例

/etc/prometheus/scrape_configs/example.yaml

實際上同主設定檔的 scrape_configs 區塊  

```
scrape_configs:
  - job_name: 'example-staging-node'
    ec2_sd_configs:
      - region: ap-northeast-1    # AWS 區域
        access_key: ''    # AWS 訪問金鑰
        secret_key: ''    # AWS 秘密金鑰
        port: 9100    # EC2 實例暴露的指標端口 同dockerfile expose , 無實際用途
        filters:
          - name: tag:Name
            values:
              - example-staging-auto-scaling-instance

    relabel_configs:
      - target_label: __metrics_path__
        replacement: /metrics   # 默認的 Prometheus 指標路徑

      - source_labels: [ __meta_ec2_private_ip ]
        regex: '(.*)'
        replacement: '${1}:9100'
        target_label: __address__

      - source_labels: [ __meta_ec2_private_ip ]
        regex: '(.*)'
        target_label: instance
        replacement: $1

      - target_label: instance_tag
        replacement: example-staging-instance
        action: replace

  - job_name: 'example-staging-cadvisor'
    ec2_sd_configs:
      - region: ap-northeast-1    # 您的 AWS 區域
        access_key: ''    # AWS 訪問金鑰
        secret_key: ''    # AWS 秘密金鑰
        port: 8080
        filters:
          - name: tag:Name
            values:
              - example-staging-auto-scaling-instance

    relabel_configs:
      - target_label: __metrics_path__
        replacement: /metrics   # 默認的 Prometheus 指標路徑

      - source_labels: [ __meta_ec2_private_ip ]
        regex: '(.*)'
        replacement: '${1}:8080'
        target_label: __address__

      - source_labels: [ __meta_ec2_private_ip ]
        regex: '(.*)'
        target_label: instance
        replacement: $1

      - target_label: instance_tag
        replacement: example-staging-instance
        action: replace
```
