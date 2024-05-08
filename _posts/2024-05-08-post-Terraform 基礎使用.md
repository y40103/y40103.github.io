---
title: "[筆記] Terraform 基礎使用"
categories:
  - 筆記
tags:
  - Terraform
  - AWS
toc: true
toc_label: Index
---

雲端版的基礎設施佈署工具,配合ansible使用可滿足大多基礎需求  


## 初始化

定義 provider

```
provider "aws" {
  region     = "OOO"
  access_key = "XXX"
  secret_key = "@@@"
}
```

```bash
terraform init
## 利用 terraform config 初始化至provider  
## 會建立該provider所需的工具包  
```

編寫main.tf, 主要使用HCL 

## AWS 基礎操作

terraform AWS的基本資源調用,  

### resource

主要用來定義資源單位 , 後續可用指令對資源進行操作

```hcl
resource "aws_vpc" "development_vpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"
  tags             = {
    Name = "development_vpc"
  }
}
# 定義VPC  
```

### data

主要用來檢索已存在的資源

```
data "aws_vpc" "default" {
  default = true
}
# 檢索default VPC

```

### plan

用於確認config(main.tf) 與當前實際佈署狀態差距

```bash
terraform plan
```

### apply

套用資源至provider, 具冪等性

```bash

terraform apply
## 套用資源

terraform apply -destroy
## 卸除資源

terraform apply -target=$資源類型.$資源名稱
## 卸除特定資源

```

### auto-approve

若不希望執行時會跳出prompt 詢問是否執行

可增加 -auto-approve

```bash
terraform apply -auto-approve
```

### destroy

撤銷config中內容

```bash
terraform destroy
```

## State

terraform 是利用 terraform.tfstate 來記錄佈署狀態  
只要成功 套用config後, 就會把實際狀態寫至 該檔案

可以藉由command對當前狀態做檢索  


***狀態檔案非常重要,之後若想繼續編輯該組設定都必須相依於這些狀態檔,若狀態檔對不上,可能會變成創建新的資源***

```bash
terraform state list
## 顯示目前所有單位資源

terraform state $資源名稱
## 可列出特定資源詳細訊息, 資源名稱可用 list 檢視  

```

## Output

主要是在config佈署後, 可以輸出佈署資源的特定屬性至stdout

```
output "dev-vpc-id" {
  value = aws_vpc.development_vpc.id
}
```

- dev-vpc-id 為輸出的key名稱
- value 則是輸出的數值, aws_vpc為資源類型 development_vpc為資源名稱 id為屬性名稱

## Variable

定義變數基本訊息

| field       | explain                |
|-------------|------------------------|
| description | 變數說明                   |
| type        | 宣告變數類型, Optional, 可不定義 |

```
variable "subnet_cidr_block" {
  description = "test for variable"
  type        = string
}
```

### Variable Type

- string
- number
- bool
- list
- map
- object

#### example

define

```hcl
variable "example_string" {
  type    = string
  default = "Hello, Terraform!"
}

variable "example_number" {
  type    = number
  default = 42
}

variable "example_boolean" {
  type    = bool
  default = true
}

variable "example_list" {
  type    = list(string)
  default = ["value1", "value2", "value3"]
}

variable "example_map" {
  type    = map(string)
  default = {
    key1 = "value1"
    key2 = "value2"
    key3 = "value3"
  }
}

variable "example_object" {
  type = object({
    name    = string
    age     = number
    enabled = bool
  })
  default = {
    name    = "John"
    age     = 30
    enabled = true
  }
}

```

call variable

```hcl
my_string = var.example_string

my_number = var.example_number

my_bool = var.example_boolean

my_list = var.example_list[0]

my_map = var.example_map["key1"]

my_obj = var.example_object.name
```

### variable in string

字串中調用變數  

```hcl

test_in_string = "${var.example_string} in string"
```

### Nested


```hcl
variable "cidr_blocks" {
  description = "test nested var"
  type        = list(
    object( {
      cidr_block = string
      name       = string
    }
    )
  )
}
```

```hcl
my_cidr_block = cidr_blocks[0].cidr_block

```

### Input Value

有三種輸入數值的方法

#### Prompt

直接套用含有定義變數的config時, 會自動提示輸入

```
root@76d867dbc089 /h/r/tutorial# terraform apply
var.subnet_cidr_block
  test for variable

  Enter a value: 10.0.2.0/24

data.aws_vpc.existing_vpc: Reading...
data.aws_vpc.existing_vpc: Read complete after 1s [id=vpc-096a8b318ea4a71c8]

```

#### Arg

flag: -var 直接輸入變數

```bash
terraform apply -var "subnet_cidr_block=10.0.2.0/24"
```

#### Variable file

較推薦此方式,預設自動載入變數檔案 terraform.tfvars, 若需使用其他名稱 可用 -var-file

目錄

```
.
|-- main.tf
|-- terraform.tfstate
|-- terraform.tfstate.backup
`-- terraform.tfvars

0 directories, 4 files
```

定義value

```hcl
subnet_cidr_block="10.0.2.0/24"
```

```
root@76d867dbc089 /h/r/tutorial [1]# terraform apply 
data.aws_vpc.existing_vpc: Reading...
data.aws_vpc.existing_vpc: Read complete after 1s [id=vpc-096a8b318ea4a71c8]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_subnet.dev_subnet-1 will be created
  + resource "aws_subnet" "dev_subnet-1" {
      + arn                                            = (known after apply)
      + assign_ipv6_address_on_creation                = false
      + availability_zone                              = "ap-northeast-1a"
      + availability_zone_id                           = (known after apply)
      + cidr_block                                     = "10.0.10.0/24"
      + enable_dns64                                   = false
      + enable_resource_name_dns_a_record_on_launch    = false
```

使用其他variable file,

```bash
terraform apply -var-file $variable_file
```

### Default value

為變數設定預設值, 若設有預設值時, 則不會prompt輸入, 若要使用新輸入的數值則需使用 Arg 或是 Variable file ,   
無使用任何輸入數值手段, 一律使用預設值

```hcl
variable "subnet_cidr_block" {
  description = "test for variable"
  default     = "10.0.1/24"
}
```


## Environment Variable

環境變數大致分成兩種, 一種是預設的環境變數 , 另一種則是自行定義的環境變數  

### 預設環境變數  

此類型 若該環境變已設定數值, config無需另外定義  

e.g. AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION   

若無設定env, 則需   
```
provider "aws" {
  region     = "OOO"
  access_key = "XXX"
  secret_key = "@@@"
}
```

若使用env  

```
export AWS_ACCESS_KEY_ID=OOO
export AWS_SECRET_ACCESS_KEY=XXX
export ArS_REGION=@@@
```
僅需  

```hcl
provider "aws" {}
```

關於AWS auth, 若已於~/.aws 定義, 也可省略這些變數的設定  


### 自定義環境變數

若要在terraform中使用自定義的環境變數, 該變數prefix必須為 TF_VAR,  
之後直接定義變署名稱, 之後該變數就會直接調用 TF_VAR prefix, 名稱相同的環境變數自動帶入  

```bash

export TF_VAR_avail_zone="ap-northeast-1"
```

```hcl
variable avail_zone {}

output test_echo1 {
  value = "${var.avail_zone}_test_var_in_string"
}

output test_echo2 {
  value = var.avail_zone
}

```

```
root@76d867dbc089 /h/r/tutorial# terraform apply
data.aws_vpc.existing_vpc: Reading...
data.aws_vpc.existing_vpc: Read complete after 0s [id=vpc-096a8b318ea4a71c8]
.......


Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:


test_echo1 = "ap-northeast-1_test_var_in_string"
test_echo2 = "ap-northeast-1"


```

## .gitignore

過濾terraform不需進版控的檔案  

.gitignore  
```
# local .terraform dir
.terraform/*

# tf state files
*.tfstate
*.tfstate.*

# tf variable
*.tfvars

```

## modules

模組化, 用於將config拆分成多個檔案, 實際上拆分為module後, 使用上是類似函數  
module底下的子目錄就與一般HCL相同, 只是可以用多個 \*.tf檔案做分類


- 若有需要將apply之後才會知道的數值輸出給其他module引用, 需使用output輸出該變數   
- 定義的變數會作為之後調用該module時的輸入 類似 f(x,y) , 要調用該模組f 需輸入x,y, 
- 通常module調用會在最外層的main, 語法範例如下, source為module的目錄路徑, 其他就是在module中定義的變數 , 這邊需輸入這些變數值  
- 若在最外側 terraform.tfvars 定義相同名稱打的變數名稱 , 則會直接帶入  



```hcl
module "provider" {
  source = "./modules/provider"
}


module "vpc_env" {
  source          = "./modules/vpc_env"
  CIDRs           = var.CIDRs
  env_prefix      = var.env_prefix
  available_zone1 = var.available_zone1
  available_zone2 = var.available_zone2
  tt_ip = var.tt_ip
  eip_tag_Name_value = var.eip_tag_Name_value
}

module "security_group" {
  source = "./modules/security_group"
  vpc_id = module.vpc_env.etc-vpc.id
  env_prefix = var.env_prefix
  tt_ip = var.tt_ip
  ap-ip = module.vpc_env.ap-ip.public_ip
}
```





## 結構化目錄參考

會自動載入當前目錄所有 \*.tf的檔案

常見為 main.tf, variable.tf, output.tf, provider.tf ....


```
├── main.tf
├── modules
       ├── alb
       │   ├── main.tf
       │   ├── output.tf
       │   └── variable.tf
       ├── backup
       │   ├── main.tf
       │   └── variable.tf
       ├── ec2
       │   ├── main.tf
       │   ├── output.tf
       │   ├── user_data.sh
       │   └── variable.tf
       ├── monitor_alert
       │   ├── main.tf
       │   └── variable.tf
       ├── provider
       │   └── main.tf
       ├── s3_backup
       │   ├── main.tf
       │   └── variable.tf
       ├── security_group
       │   ├── main.tf
       │   ├── output.tf
       │   └── variable.tf
       ├── vpc
       ├── vpc_env
       │   ├── main.tf
       │   ├── output.tf
       │   └── variable.tf
       └── waf
           ├── main.tf
           └── variable.tf
```


