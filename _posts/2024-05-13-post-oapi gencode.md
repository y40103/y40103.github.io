---
title: "[筆記] oapi gencode"
categories:
  - 筆記
tags:
  - Swagger
  - Golang
toc: true
toc_label: Index
---

Golang Swagger3.0 方案之一  

- 為Swagger 3.0 config first的 Golang codegen工具, [oapi-codegen](https://github.com/deepmap/oapi-codegen/tree/master)  
- 主要是藉由swagger config生成API的輸入輸出類型相關的程式碼(request參數輸入,response輸出的 marshal/unmarshal)  
<br/>
優點為  
- 可以保持API文件與程式碼的一致性
- 節省mashalling/unmashalling的時間
- 只需要開發者專注於handler的實現

## 支援

- Chi
- Echo
- Fiber
- Gin
- gorilla/mux
- Iris
- net/http (Go 1.22+)
 


## 差異

之前用過gin-swagger這類的code first工具來產生swagger文件,  
除了只支援swagger2.0之外, 藉由撰寫註釋的方式來生成swagger文檔維護上沒有想像中的好用,所有的異動都需要靠自己來維護  
唯一的優點是比較直覺一些,對於初次使用者來說比較容易上手  


## 安裝

```bash
go install github.com/deepmap/oapi-codegen/v2/cmd/oapi-codegen@latest
oapi-codegen -version
```
## gencode 指令

```bash
oapi-codegen --package=<package name> <swagger config yaml/json>
## 之後會返回生成的程式碼
```

實際範例
```bash
oapi-codegen --package=api dist/openapi_test.yaml > api/gencode.go
```


## 開發流程

- 編寫openapi文件
- 產生程式碼
- 實作API的handler (request input/response output類型可直接調用上面產生的程式碼)
- 實做server端的main函數, 將handler與路由綁定, 啟動server

## 使用範例1

這邊是使用 oapi-codegen的範例, 實做一個最簡單的API Demo  
oapi-gencode 編寫的swagger文件需依賴Swagger的編輯器才能瀏覽效果, 這邊會新增一個swagger的WebUI來方便瀏覽效果


### 初始化

```bash
go mod init mininal_echo

go mod tidy

mkdir api
## 也是之後gencode的package name, 之後會將生成的程式碼放在這個目錄下
```

需先去[swagger](https://github.com/swagger-api/swagger-ui) clone下來, 並將dist目錄複製到專案目錄下


### openapi文件

編寫新增 dist/openapi_test.yaml

```yaml
openapi: "3.0.0"
info:
  version: 1.0.0
  title: Minimal ping API server
paths:
  /ping:
    get:
      responses:
        '200':
          description: pet response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Pong'
      tags:
        - PingDemo
components:
  schemas:
    # base types
    Pong:
      type: object
      required:
        - ping
      properties:
        ping:
          type: string
          example: pong
```

```
oapi-codegen --package=api dist/openapi_test.yaml > api/gencode.go
```


### 修改swagger dist config目標

需把swagger dist套用的設定檔改成剛剛寫的openapi_test.yaml  
dist/swagger-initializer.js

```javascript
window.onload = function() {
  //<editor-fold desc="Changeable Configuration Block">

  // the following lines will be replaced by docker/configurator, when it runs in a docker-container
  window.ui = SwaggerUIBundle({
    url: "./openapi_test.yaml", //這邊改成剛設定的swagger config
    dom_id: '#swagger-ui',
    deepLinking: true,
    presets: [
      SwaggerUIBundle.presets.apis,
      SwaggerUIStandalonePreset
    ],
//    plugins: [   // 這邊是關閉swagger ui的topbar, 不影響使用
//      SwaggerUIBundle.plugins.DownloadUrl
//    ],
//    layout: "StandaloneLayout"
  });

  //</editor-fold>
};
```


### handler

藉由oapi-codegen 生成程式碼(request input,response output), 編寫新增api/impl.go

```golang
package api

import (
	"net/http"

	"github.com/labstack/echo/v4"
)

// ensure that we've conformed to the `ServerInterface` with a compile-time check
var _ ServerInterface = (*Server)(nil)

type Server struct{}

func NewServer() Server {
	return Server{}
}

// (GET /ping)
func (Server) GetPing(ctx echo.Context) error {
	resp := Pong{
		Ping: "pong",
	}

	return ctx.JSON(http.StatusOK, resp)
}
```

### main.go

編寫新增 ./main.go, 實作server端的main函數, 將handler與路由綁定, 啟動server  
這邊用echo 來演示, 使用不同api框架, 只會在這邊的實作有所不同

```golang
package main

import (
	"github.com/labstack/echo/v4"
	"log"
	"net/http"

	"minimal_echo/api"
)

func main() {
	// create a type that satisfies the `api.ServerInterface`, which contains an implementation of every operation from the generated code
	server := api.NewServer()

	e := echo.New()

	// Serve the Swagger UI files and the dist folder
	e.GET("/docs/*", echo.WrapHandler(http.StripPrefix("/docs/", http.FileServer(http.Dir("./dist")))))

	api.RegisterHandlers(e, server)

	// And we serve HTTP until the world ends.
	log.Fatal(e.Start("0.0.0.0:8080"))
}

````


### 目錄結構

此時目錄結構如下

```
minimal_echo/
├── api
│   ├── gencode.go
│   └── impl.go
├── dist
│   ├── favicon-16x16.png
│   ├── favicon-32x32.png
│   ├── index.css
│   ├── index.html
│   ├── oauth2-redirect.html
│   ├── openapi_test.yaml
│   ├── swagger-initializer.js
│   ├── swagger-ui-bundle.js
│   ├── swagger-ui-bundle.js.map
│   ├── swagger-ui.css
│   ├── swagger-ui.css.map
│   ├── swagger-ui-es-bundle-core.js
│   ├── swagger-ui-es-bundle-core.js.map
│   ├── swagger-ui-es-bundle.js
│   ├── swagger-ui-es-bundle.js.map
│   ├── swagger-ui.js
│   ├── swagger-ui.js.map
│   ├── swagger-ui-standalone-preset.js
│   └── swagger-ui-standalone-preset.js.map
├── go.mod
├── go.sum
└── main.go
```

### 啟動server

```bash
go build main.go
./main
```

此時可訪問 [http://localhost:8080/docs/](http://localhost:8080/docs/) 來瀏覽swagger文檔   

