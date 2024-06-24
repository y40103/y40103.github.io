---
title: "[筆記] Linux 系統校時"
categories:
  - 筆記
tags:
  - Linux 
toc: true
toc_label: Index
mermaid: true
---


查看一下資料 比較流行的套件  

- ntpdate 準備棄用  
- ntpd
- chrony
- timedatectl  


這邊記錄一下 chrony的作法  
實際上一開始是使用ntpd, 但使用預設的time server, 不知道為啥跟台北時間差好幾秒  
有些服務會依賴時間, 如果誤差太大蠻可怕的  
chrony裝起來後, 時間就是準的 記錄一下指令  



設定檔位置  
```
/etc/chrony.conf
```

### Quickstart


```bash
sudo apt install chrony
# 安裝

sudo systemctl restart chrony.service
# 重啟會自動重新校時

chronyc sources
# 查看同步狀態

chronyc sourcestats
# 同上
```

