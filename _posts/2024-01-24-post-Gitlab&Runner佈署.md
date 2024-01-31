---
title: "[筆記] Gitlab & Runner 佈署"
categories:
  - 筆記
tags:
  - Gitlab
toc: true
toc_label: Index
---


## Gitlab 佈署
用container啟動, 簡化佈署複雜度  
這邊須注意 external_url需與docker network 分配的相同, 設定上直接採用靜態IP
或是直接使用宿主機IP,

```yaml
version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ee:latest'
    restart: always
    hostname: 'hccuse'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://10.20.0.2'
        # Add any other gitlab.rb configuration here, each on its own line
    ports:
      - '80:80'
      - '10443:443'
      - '10022:22'
    volumes:
      - './gitlab/config:/etc/gitlab'
      - './gitlab/logs:/var/log/gitlab'
      - './gitlab/data:/var/opt/gitlab'
    shm_size: '256m'
    networks:
      gitlab_net:
        ipv4_address: 10.20.0.2


networks:
  gitlab_net:
    name: "gitlab_net"
    driver: bridge
    ipam:
      config:
        - subnet: 10.20.0.0/16
          gateway: 10.20.0.1
```

等待初始化完成, 大概需要等5分鐘左右 status: start -> healthy

```bash
CONTAINER ID   IMAGE                     COMMAND             CREATED          STATUS                    PORTS                                                                                                                   NAMES
412f93c68da6   gitlab/gitlab-ee:latest   "/assets/wrapper"   12 minutes ago   Up 12 minutes (healthy)   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:10022->22/tcp, :::10022->22/tcp, 0.0.0.0:10443->443/tcp, :::10443->443/tcp   tutorial-web-1
```

預設username: root  
查看預設密碼

```bash
docker exec -it 41 cat /etc/gitlab/initial_root_password
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

# Password: fUqiMaEMlMXP0ofNgrBemlA5mKQcspciCJmaPv/ZLSY=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

[login](http://localhost)

Overview -> User -> Edit 修改密碼

## runner

增加runner    
佈署local測試用runner   

```yaml
version: '3.6'
services:
  gitlab:
    image: 'gitlab/gitlab-ee:latest'
    restart: always
    hostname: 'hccuse'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://172.24.141.200'
        # Add any other gitlab.rb configuration here, each on its own line
    ports:
      - '80:80'
      - '10443:443'
      - '10022:22'
    volumes:
      - './gitlab/config:/etc/gitlab'
      - './gitlab/logs:/var/log/gitlab'
      - './gitlab/data:/var/opt/gitlab'
    shm_size: '256m'
    networks:
      gitlab_net:
        ipv4_address: 10.20.0.2

  runner:
    image: gitlab/gitlab-runner:latest
    restart: always
    volumes:
      - './gitlab-runner/:/etc/gitlab-runner/'
      - '/var/run/docker.sock:/var/run/docker.sock'
    networks:
      gitlab_net:
        ipv4_address: 10.20.0.3





networks:
  gitlab_net:
    name: "gitlab_net"
    driver: bridge
    ipam:
      config:
        - subnet: 10.20.0.0/16
          gateway: 10.20.0.1
```


### register

```bash
gitlab-runner register  --url http://172.24.141.200  --token glrt-i_Hioib-mPsxZRTmd6U4
```

### delete

需先從gitlab UI刪除, 再執行以下指令


