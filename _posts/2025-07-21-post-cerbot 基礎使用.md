---
title: "[筆記] cerbot 基礎使用"
categories:
  - 筆記
tags:
  - SSL
  - cerbot
toc: true
toc_label: Index
---

免費仔的SSL好夥伴

## Install

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo apt install nginx
```

## 方案a. 只想為某domain取得憑證檔案

```bash
sudo certbot certonly --manual --preferred-challenges=dns --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d <domain name>
#sudo certbot certonly --manual --preferred-challenges=dns --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d *.coco77.online
```

之後會需要輸入email,

- 驗證
  之後會產生一串value, 這邊的意思是他們請求 \_acme-challenge.coco77.online 這個網址 需要返回 TEa50cNcj5GKLvMQdnttJ9k9NKSQb5E1xyBYq50IxGo

這邊domain驗證是直接在 dns 設定

```bash
Please deploy a DNS TXT record under the name:

_acme-challenge.coco77.online.

with the following value:

TEa50cNcj5GKLvMQdnttJ9k9NKSQb5E1xyBYq50IxGo

Before continuing, verify the TXT record has been deployed. Depending on the DNS
provider, this may take some time, from a few seconds to multiple minutes. You can
check if it has finished deploying with aid of online tools, such as the Google
Admin Toolbox: https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.coco77.online.
Look for one or more bolded line(s) below the line ';ANSWER'. It should show the
value(s) you've just added.
```

基本上就是去dns 設置 TXT record,

\_acme-challenge TEa50cNcj5GKLvMQdnttJ9k9NKSQb5E1xyBYq50IxGo

輸出的證書位置

`/etc/letsencrypt/archive/<domain name>`

```bash
root@ip-172-31-25-36:/etc/letsencrypt/archive/coco77.online# ls
cert1.pem  chain1.pem  fullchain1.pem  privkey1.pem
```

- nginx 憑證使用參考

```text
server {
        listen80;
        listen [::]:80;
        server_name example.com;
        return301 https://$host$request_uri;
}
server {
       listen443 ssl http2;
       listen [::]:443 ssl http2;
       server_name example.com;
       ssl_certificate /etc/letsencrypt/archive/example.com/fullchain1.pem;
       ssl_certificate_key /etc/letsencrypt/archive/example.com/privkey1.pem;
       # Other SSL config
       ...
}
```

## 方案b. 直接在該臺機器設定憑證,並自動renew

假設要申請 proxy.houseminer.com.tw proxy2.houseminer.com.tw

```bash
sudo cat > /etc/nginx/sites-available/houseminer.com.tw << EOF
server_name proxy.houseminer.com.tw proxy2.houseminer.com.tw;
EOF
```

- 開防火牆

```bash
sudo ufw allow 80/tcp
# 驗證domain用
sudo ufw allow 443/tcp
# 驗證完後續這個domain若走https 對外就需要
```

- 驗證domain

這邊驗證是server side, 作法是直接在目錄下創建指定的檔案內容, 會有驗證方來request這台機器檢查內容是否符合

```bash
sudo certbot --nginx -d proxy.houseminer.com.tw -d proxy2.houseminer.com.tw
```

- 驗證結果

```bash
sudo systemctl status certbot.timer

sudo certbot renew --dry-run
```

```bash
sudo systemctl status certbot.timer
#● certbot.timer - Run certbot twice daily
#     Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset: enabled)
#     Active: active (waiting) since Thu 2025-07-17 13:27:42 CST; 18min ago
#    Trigger: Fri 2025-07-18 01:34:21 CST; 11h left
#   Triggers: ● certbot.service
#
#Jul 17 13:27:42 chttl-9984d3f75155e09b systemd[1]: Started Run certbot twice daily.
sudo certbot renew --dry-run
#Saving debug log to /var/log/letsencrypt/letsencrypt.log
#
#- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
#Processing /etc/letsencrypt/renewal/proxy.houseminer.com.tw.conf
#- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
#Account registered.
#Simulating renewal of an existing certificate for proxy.houseminer.com.tw
#
#- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
#Congratulations, all simulated renewals succeeded:
#  /etc/letsencrypt/live/proxy.houseminer.com.tw/fullchain.pem (success)
#- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

- nginx 設定檔參考

這個驗證成功後, 它會有輸出一個設定檔在 `/etc/nginx/sites-available/default`

這邊是配合squid的設定檔(只是需要SSL), nginx只是給cerbot自動renew用的

```text
server {
 listen 80 default_server;
 listen [::]:80 default_server;

 # SSL configuration
 root /var/www/html;

 # Add index.php to the list if you are using PHP
 index index.html index.htm index.nginx-debian.html;

 server_name _;

 location / {
  # First attempt to serve request as file, then
  # as directory, then fall back to displaying a 404.
  try_files $uri $uri/ =404;
 }


#server {
#
#  root /var/www/html;
#
# # Add index.php to the list if you are using PHP
#  index index.html index.htm index.nginx-debian.html;
#  server_name proxy.houseminer.com.tw; # managed by Certbot
#
#  location / {
#   # First attempt to serve request as file, then
#   # as directory, then fall back to displaying a 404.
#   try_files $uri $uri/ =404;
#  }
#
#
#  listen [::]:443 ssl ipv6only=on; # managed by Certbot
#  listen 443 ssl; # managed by Certbot
#  ssl_certificate /etc/letsencrypt/live/proxy.houseminer.com.tw/fullchain.pem; # managed by Certbot
#  ssl_certificate_key /etc/letsencrypt/live/proxy.houseminer.com.tw/privkey.pem; # managed by Certbot
#  include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
#  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
#
#}

### 以上443 這邊直接取消, 因希望443直接導向 squid


server {
    if ($host = proxy.houseminer.com.tw) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


 listen 80 ;
 listen [::]:80 ;
    server_name proxy.houseminer.com.tw;
    return 404; # managed by Certbot


}

### 以上80

```
