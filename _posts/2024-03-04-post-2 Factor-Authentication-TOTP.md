---
title: "[筆記] 2 Factor-Authentication TOTP"
categories:
  - 筆記
tags:
  - Authentication
toc: true
toc_label: Index
mermaid: true
---


## 概念

簡單來說需要兩組的驗證機制  

方法有很多種, 這邊採用

- 帳密驗證
- Time-base One Time Password

主要紀錄TOTP   

概念上很簡單 就是利用timestamp與一組secret 產生一組驗證碼     

為了給該驗證碼有期限, 該timestamp會用一個算法讓他在一段時間內可以輸出同一個結果   

用該結果跟secret 就可以在一段時間產出相同驗證碼    

### sudo code  

```

base_time = now().timestamp

## wait few seconds

t =  floor( (now().timestamp() -  base_time) / 30 )

## expired time = 30 s ,   if  over 30 seconds , output will change

TOTP = hash(mysecret,t)

```


以Google Authenticator舉例  

Server 產生secret 並以QRCODE方式傳給使用者  

使用者用 Google Authenticator 掃描該QRCODE 取得secret   

Google Authenticator 就可以用該secret與當前時間計算出驗證碼  

同時Server端也在利用該secret計算驗證碼,  

因此使用者輸入驗證碼後, 就可以進行驗證  




## 實作範例  


### Python

```python
import pyotp
import qrcode

secret = pyotp.random_base32()

totp = pyotp.TOTP(secret)

validation_code = totp.now()

uri = totp.provisioning_uri(name="myapp",issuer_name="foo-company")

img = qrcode.make(uri)

img.save("qrcode.png")

```


### Go

```golang

package main  
  
import (  
    "bytes"  
    "github.com/pquerna/otp/totp"    "image/png"    "log"    "os")  
  
func main() {  
    key, err := totp.Generate(totp.GenerateOpts{  
       Issuer:      "Example.com",  
       AccountName: "alice@example.com",  
    })  
    if err != nil {  
       panic(err)  
    }  
    var buf bytes.Buffer  
    img, err := key.Image(200, 200)  
    if err != nil {  
       panic(err)  
    }  
    png.Encode(&buf, img)  
  
    file, err := os.Create("qrcode.png")  
    if err != nil {  
       log.Fatal(err)  
    }  
    defer file.Close()  
    _, err = buf.WriteTo(file)  
  
    if err != nil {  
       log.Fatal(err)  
    }  
}
```
