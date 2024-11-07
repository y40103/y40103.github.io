---
title: "[筆記] ssh config tunnel"
categories:
  - 筆記
tags:
  - Linux
toc: true
toc_label: Index
mermaid: true
---


可直接把常需要使用跳板機的機器直接寫在~/.ssh/config , 之後可直接使用alias連線


## config example

`~/.ssh/config`

```
Host auto
  ProxyJump bastion
  User ubuntu
  Hostname 10.0.55.xxx
  IdentityFile /home/hccuse/.ssh/auto.pem  # the key that tunnel machine connect to target machine

Host bastion
  HostName 13.230.199.ooo
  User ubuntu
# IdentityFile:        # the key that mypc connect to tunnel machine 
```

- auto: remote target machine alias

- bastion: tunnel machine

- 13.230.199.ooo: tunnel machine host

- 10.0.55.xxx: remote target machine host



```bash
hccuse@hccuse-GEM12 ~> ssh auto
Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 6.5.0-1018-aws x86_64)
ubuntu@ip-10-0-55-xxx:~$ 

```
