---
title: "[筆記] Terragrunt 基礎使用"
categories:
  - 筆記
tags:
  - terraform
  - terragrunt
toc: true
toc_label: Index
---

也運行了幾個月了, 紀錄一下理解與基礎用法

- terraform 擴充工具, 可定義一份tf 創建管理多環境資源

- 概念是把terraform main.tf 把每個資源都變成一個小的main.tf 但是以hcl去定義

- environment下每個目錄都是一個獨立資源, 都會有一個terragrunt.hcl, 通常是引入module, 並定義輸入參數

- 為了避免重複設定，會將共通邏輯（如 backend、provider、region 等）抽出放在上層目錄，並透過 `include` 或 `dependency` 引入。

- 可自動創建存放狀態檔的資源 e.g. s3 , DynamoDB

---

## 🧱 目錄結構與基礎設定範例

```bash
.
├── environment # 資源目錄 實際是定義一些metadata 還有輸入 特定module參數
│   ├── dev # 各環境 可選部份需要的modoule
│   │   ├── alb_proxy
│   │   └── vpc_env
│   └── prod
│       ├── alb_proxy
│       ├── asg
│       ├── codebuild
│       ├── codedeploy
│       ├── codepipeline
│       ├── route53
│       ├── security_group
│       └── vpc_env
├── modules # modules目錄 實際定義terraform 語法, 類似打包 一個function
│   ├── alb_proxy
│   │   ├── main.tf
│   │   ├── output.tf
│   │   └── variable.tf
│   ├── asg
│   │   ├── main.tf
│   │   ├── output.tf
│   │   └── variable.tf
│   ├── codebuild
│   │   ├── main.tf
│   │   └── variable.tf
│   ├── codedeploy
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variable.tf
│   ├── codepipeline
│   │   ├── main.tf
│   │   └── variable.tf
│   ├── route53
│   │   ├── main.tf
│   │   ├── output.tf
│   │   └── variable.tf
│   ├── security_group
│   │   ├── main.tf
│   │   ├── output.tf
│   │   └── variable.tf
│   └── vpc_env
│       ├── main.tf
│       ├── output.tf
│       └── variable.tf

└── terragrunt.hcl


```

### root.hcl 範本

這邊是根目錄資源, 基本上這個hcl是每個子hcl都會include進去的, 語法如下

```hcl
include "root" {
  path = find_in_parent_folders()
}
# 遞迴往上層目錄尋找, 找到的第一個hcl, include近來
```

通常主要會

- 定義狀態檔案存放位置, terragrunt是把每個module產生的instance 當作獨立資源管理, 因此都需要各別定義狀態檔
- 定義provider, e.g. 定義該資源是使用哪個 provider 的物件, e.g. aws , aws provider也會需要定義 region, 這邊創建的資源就會在該region, 無特別定義似乎就是以環境變數AWS_REGION為主

以下範例是定義將state file放在s3, s3 region 位置預設是在ap-northeast-1, 寫鎖是會在dynamodb 創建一個table作為lock

generate 的意思是會在資源目錄下 運行時 暫時創建, 非運行狀態不會保留

```hcl
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${get_env("AWS_REGION", "ap-east-2")}"
}
EOF
} # 產生provider 的 tf file

remote_state {
  backend = "s3"
  config = {
    encrypt        = true
    bucket         = "houseplus-tw-test4-terraform-state"
    key            = "${path_relative_to_include()}/tf.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "houseplus-tw-terraform-locks"
  }
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
} # 產生 狀態檔存放位置的 tf file

```

他的實做方式是在執行時,會在資源目錄 e.g. environment/dev/alb_proxy/ 下產生 tf file

如backend.tf , 內容大概為

```tf
terraform {
  backend "s3" {
    bucket         = "formatted-address-terraform-state"
    key            = "dev/vpc/terraform.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "formatted-address-terraform-locks"
    encrypt        = true
  }
}
```

如 provider.tf , 內容大概為

```tf
provider "aws" {
  region = var.aws_region
}
```

總之就是為每個資源都定義一個狀態存儲設定, provider ...

若是寫raw terraform 全部塞成一大包 main.tf , 就會有一組以上設定, terragrunt只是把一大包拆成小單位, 每個小單位也都會需要一組

### 資源檔 terragrunt.hcl 範本

```hcl
terraform {
  source = "../../../modules/alb_proxy"
}


include {
  path = find_in_parent_folders()
}


dependency "vpc_env" {
  config_path = "../vpc_env"
}


inputs = {
  vpc_id = dependency.vpc_env.outputs.vpc.id  # vpc name
  env_prefix = "formatted-address-prod" # 專案前綴
  listener_reverse_proxy_rule = "anfu-formatted-address.houseplus.com.tw" # load balancer 反向代理的 hostname
  service_port = 80 # 代理目標主機的service port
  https_listener_arn = "arn:aws:elasticloadbalancing:ap-northeast-1:537127459413:listener/app/houseplus-elb/5040da885994f0ed/1a285f2b0de68d8b"
  rule_priority = 1500  # 該 load balancer rule 的優先順序, 不能與已存在的priority 重複
}
```

### modules 範例

- main.tf 可理解為主物件

```tf
resource "aws_lb_target_group" "alb_target_group" {
  name        = "${var.env_prefix}-tg"
  port        = var.service_port
  protocol    = "HTTP"
  target_type = "instance"
  vpc_id      = var.vpc_id
  tags = { Name : "${var.env_prefix}" }
}




resource "aws_lb_listener_rule" "reverse_proxy_rule" {
  listener_arn = var.https_listener_arn
  priority     = var.rule_priority
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.alb_target_group.arn
  }
  condition {
    host_header {
      values = [var.listener_reverse_proxy_rule]
    }
  }
}

```

- variable.tf 創建主物件所需的輸入

```tf
variable "env_prefix" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "listener_reverse_proxy_rule" {
  type = string
}


variable "service_port" {
  type = number
}

variable "rule_priority" {
  type = number
}

variable "https_listener_arn" {
  type = string
}⏎
```

- output.tf 主物件的屬性, 其他地方調用該物件作為dependency時, 可以從中讀取這些屬性

```tf
output "alb_target_group_arn" {
  value = aws_lb_target_group.alb_target_group.arn
}

output "forward_service_port" {
  value = var.service_port
}
```

## 基本命令

通常會在 environment/dev/ 這目錄下執行, 意思是創建 or 刪除 dev的infra資源

```bash
terragrunt run-all apply
# 遞迴執行當前目錄下所有hcl 所創建的資源

terragrunt run-all apply -destroy

# 遞迴刪除當前目錄下所有hcl 所創建的資源
```

通常會在 environment/dev/alb_proxy/ 下執行, 創建 or 刪除 當前資源 (若有dependency需注意, 需先刪除 or 創建 dependency 才能操作)

```bash
terragrunt apply
# 創建當前資源

terragrunt apply -destroy
# 刪除當前資源
```
