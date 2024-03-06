---
title: "[筆記] SSH Two-Factor-Authentication TOTP"
categories:
  - 筆記
tags:
  - Authentication
  - Linux
toc: true
toc_label: Index
mermaid: true
---

## 系統環境

```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04.6 LTS"
NAME="Ubuntu"
VERSION="20.04.6 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.6 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```

##  系統環境

```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04.6 LTS"
NAME="Ubuntu"
VERSION="20.04.6 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.6 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```

## 安裝 google-authentication pam

```
apt-get install libpam-google-authenticator -y
```

## 驗證組合
###  方案一 Password + TOTP

#### 設置驗證方式

ChallengeResponseAuthentication

- no 表示只用 密碼或publicKey 驗證
- yes  prompt 其他驗證方式 

```
sed -i 's/ChallengeResponseAuthentication\ no/ChallengeResponseAuthentication\ yes/g' /etc/ssh/sshd_config

## no 取代為 yes
```

#### 設置 pam_google_authenticator 驗證

```
echo "Auth required pam_google_authenticator.so" >> /etc/pam.d/sshd

## 設置使用pam_google_authenticator
```

### 方案二 Password + PublicKey + TOTP

#### 設置驗證方式

ChallengeResponseAuthentication

- no 表示只用 密碼或publicKey 驗證
- yes  使用其他驗證方式 

```
sed -i 's/ChallengeResponseAuthentication\ no/ChallengeResponseAuthentication\ yes\nAuthenticationMethods publickey,keyboard-interactive/g' /etc/ssh/sshd_config

## no 取代為 yes
##　AuthenticationMethods publickey,keyboard-interactive　指定驗證方式
## keyboard-interactove 通常表示用戶名稱 與 密碼 
```

#### 設置 pam_google_authenticator 驗證

```
echo "Auth required pam_google_authenticator.so" >> /etc/pam.d/sshd

## 設置使用pam_google_authenticator
```

### 方案三 PublicKey + TOTP

#### 設置驗證方式

ChallengeResponseAuthentication

- no 表示只用 密碼或publicKey 驗證
- yes  使用其他驗證方式 

```
sed -i 's/ChallengeResponseAuthentication\ no/ChallengeResponseAuthentication\ yes\nAuthenticationMethods publickey,keyboard-interactive\nPasswordAuthentication no\n/g' /etc/ssh/sshd_config

## 同方案二
## 但多新增關閉密碼選項  
```

#### 設置 pam_google_authenticator 驗證

```
sed -i 's/@include\ common-auth/\#\ @include\ common-auth/g' /etc/pam.d/sshd
## 關閉include /etc/pam.d/common-auth

echo "Auth required pam_google_authenticator.so" >> /etc/pam.d/sshd
## 設置使用pam_google_authenticator

```



## 重啟service

```
service ssh restart 
# 重啟ssh server
```


## 設定TOTP SECRET


***登錄至要開啟totp的user***


```
google-authenticator -t -f -d -r 3 -R 30 -w 2

#-t ：使用 time-based verification
#-f : save ~/.google_authenticator
#-d : disable totp token reuse
#-e : 3組 備用碼, default 1, 使用會被消耗
#-w : 設置驗證碼的可容許度, 若網路或時間有稍微誤差, 可容許前後2組數值  

# 登陸限制 30s內 3次
#-r : set N ,limit login N per every M seconds
#-R : set N ,limit login N per every M seconds


```


會取得qrcode 與 url,    
可使用 totp驗證碼產生的app 藉由qrcode輸入 secret   
之後即可用該secret產生的驗證碼登陸  




## 補充


### ubuntu 20.04

#### Dockerfile

ssh user defaul: user/password:  hcc/example

```
FROM ubuntu:20.04
RUN apt update && apt install -y openssh-server
RUN apt-get install -y vim
RUN useradd -m -s /bin/bash hcc
RUN echo "hcc:example" | chpasswd
USER hcc
RUN ssh-keygen -m PEM -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
RUN mkdir -p /home/hcc/.ssh/
RUN touch /home/hcc/.ssh/authorized_keys
USER root
EXPOSE 22
ENTRYPOINT service ssh start && bash
```

#### docker compose.yaml 

```
version: '3.8'
services:
  ssh1:
    restart: always
    image: ssh_test:v1.0
    tty: true
    container_name: ssh1
    build:
      context: .
      dockerfile: ./Dockerfile
    volumes:
      - ./root_run.sh:/root/root_run.sh
      - ./run.sh:/home/hcc/run.sh
    networks:
      ssh_net:
        ipv4_address: 120.20.0.2

  ssh2:
    restart: always
    image: ssh_test:v1.0
    tty: true
    container_name: ssh2
    volumes:
      - ./root_run.sh:/root/root_run.sh
      - ./run.sh:/home/hcc/run.sh
    build:
      context: .
      dockerfile: ./Dockerfile
    networks:
      ssh_net:
        ipv4_address: 120.20.0.3

networks:
  ssh_net:
    name: "ssh_net"
    driver: bridge
    ipam:
      config:
        - subnet: 120.20.0.0/16
          gateway: 120.20.0.1


```


#### root_run.sh

```
#!/usr/bin/bash
apt-get install libpam-google-authenticator -y
sed -i 's/ChallengeResponseAuthentication\ no/ChallengeResponseAuthentication\ yes\nAuthenticationMethods publickey,keyboard-interactive\nPasswordAuthentication no\n/g' /etc/ssh/sshd_config
sed -i 's/@include\ common-auth/\#\ @include\ common-auth/g' /etc/pam.d/sshd
echo "Auth required pam_google_authenticator.so" >> /etc/pam.d/sshd
service ssh restart
```

#### run.sh

```
#!/usr/bin/bash
if [ ! -f /tmp/"$USER"_totp_auth_info ]; then
 google-authenticator -t -f -d -r 3 -R 30 -w 2 >> /tmp/"$USER"_totp_auth_info
 grep -o 'http[s]\?://[^"]\+' /tmp/"$USER"_totp_auth_info | wget -i - -O /tmp/"$USER"_qrcode.jpg
fi

```


#### 懶人包

這邊測試用 public key + totp

```
docker compose up -d
## 將ssh1 public key 丟到 ssh2

docker exec -it ssh2 bash

cd /root

bash root_run.sh

su hcc

cd /home/hcc

bash run.sh

exit

docker cp ssh1:/home/hcc/id_rsa .

ssh -i id_rsa hcc@ssh2
# 輸入驗證碼

## 登入ssh2成功

```



### ubuntu22.04

#### ubuntu 22.04異動

22.04之後 /etc/ssh/sshd_config 中的ChallengeResponseAuthentication已被KbdInteractiveAuthentication取代

```
#直接取代就可以了
如
sed -i 's/ChallengeResponseAuthentication\ no/ChallengeResponseAuthentication\ yes\nAuthenticationMethods publickey,keyboard-interactive\nPasswordAuthentication no\n/g' /etc/ssh/sshd_config

轉為

sed -i 's/KbdInteractiveAuthentication\ no/KbdInteractiveAuthentication\ yes\nAuthenticationMethods publickey,keyboard-interactive\nPasswordAuthentication no\n/g' /etc/ssh/sshd_config
```


```
google-authenticator 多新增一驗證碼確認的prompt  
若用腳本 可直接利用 -C 跳過
```

#### Dockerfile

ssh user defaul: user/password:  hcc/example

```
FROM ubuntu:22.04
RUN apt update && apt install -y openssh-server
RUN apt-get install -y vim
RUN useradd -m -s /bin/bash hcc
RUN echo "hcc:example" | chpasswd
USER hcc
RUN ssh-keygen -m PEM -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
RUN mkdir -p /home/hcc/.ssh/
RUN touch /home/hcc/.ssh/authorized_keys
USER root
EXPOSE 22
ENTRYPOINT service ssh start && bash
```

#### docker compose.yaml 

```
version: '3.8'
services:
  ssh1:
    restart: always
    image: ssh_test:v1.0
    tty: true
    container_name: ssh1
    build:
      context: .
      dockerfile: ./Dockerfile
    volumes:
      - ./root_run.sh:/root/root_run.sh
      - ./run.sh:/home/hcc/run.sh
    networks:
      ssh_net:
        ipv4_address: 120.20.0.2

  ssh2:
    restart: always
    image: ssh_test:v1.0
    tty: true
    container_name: ssh2
    volumes:
      - ./root_run.sh:/root/root_run.sh
      - ./run.sh:/home/hcc/run.sh
    build:
      context: .
      dockerfile: ./Dockerfile
    networks:
      ssh_net:
        ipv4_address: 120.20.0.3

networks:
  ssh_net:
    name: "ssh_net"
    driver: bridge
    ipam:
      config:
        - subnet: 120.20.0.0/16
          gateway: 120.20.0.1


```


#### root_run.sh

```
#!/usr/bin/bash  
apt-get install libpam-google-authenticator -y  
sed -i 's/KbdInteractiveAuthentication\ no/KbdInteractiveAuthentication\ yes\nAuthenticationMethods publickey,keyboard-interactive\nPasswordAuthentication no\n/g' /etc/ssh/sshd_config  
sed -i 's/@include\ common-auth/\#\ @include\ common-auth/g' /etc/pam.d/sshd  
echo "Auth required pam_google_authenticator.so" >> /etc/pam.d/sshd  
service ssh restart
```

#### run.sh

```
#!/usr/bin/bash  
if [ ! -f /tmp/"$USER"_totp_auth_info ]; then  
 google-authenticator -t -f -d -r 3 -R 30 -w 2 -C >> /tmp/"$USER"_totp_auth_info  
 grep -o 'http[s]\?://[^"]\+' /tmp/"$USER"_totp_auth_info | wget -i - -O /tmp/"$USER"_qrcode.jpg  
fi
```


#### 懶人包

這邊測試用 public key + totp

```
docker compose up -d
## 將ssh1 public key 丟到 ssh2

docker exec -it ssh2 bash

cd /root

bash root_run.sh

su hcc

cd /home/hcc

bash run.sh

exit

docker cp ssh1:/home/hcc/id_rsa .

ssh -i id_rsa hcc@ssh2
# 輸入驗證碼

## 登入ssh2成功

```

