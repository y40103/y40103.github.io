---
title: "[筆記] 手動設定Swagger至Web UI"
categories:
  - 筆記
tags:
  - Swagger
  - Golang
toc: true
toc_label: Index
---

## 手動設定Swagger至Web UI

不依賴套件,手動設定Swagger設定檔, 輸出成web頁面


至 [swagger](https://github.com/swagger-api/swagger-ui) clone 最新版本, 並將dist資料夾複製到專案中

並將swagger的設定檔 檔案放到dist資料夾中, 這邊範例檔名為testswagger.yaml, 最後需套用該設定檔

### testswagger.yaml

```yaml
openapi: 3.0.0
info:
  contact:
    name: "test swagger UI"
  title: "Swagger Demo"
  version: "1.0"
servers:
  - url: http://localhost:8080
    description: demo
paths:
  "/hello-world":
    get:
      responses:
        "200":
          description: OK
          content:
            text/plain:
              schema:
                type: string
      tags:
        - DemoSwagger
```


### dist目錄結構

```
dist
├── favicon-16x16.png
├── favicon-32x32.png
├── index.css
├── index.html
├── oauth2-redirect.html
├── swagger-initializer.js
├── swagger-ui-bundle.js
├── swagger-ui-bundle.js.map
├── swagger-ui.css
├── swagger-ui.css.map
├── swagger-ui-es-bundle-core.js
├── swagger-ui-es-bundle-core.js.map
├── swagger-ui-es-bundle.js
├── swagger-ui-es-bundle.js.map
├── swagger-ui.js
├── swagger-ui.js.map
├── swagger-ui-standalone-preset.js
├── swagger-ui-standalone-preset.js.map
└── testswagger.yaml
```

### dist套用swagger設定檔

需將dist中的swagger-initializer.js中讀取的swagger設定檔指向 testswagger.yaml

```js
window.onload = function() {
  //<editor-fold desc="Changeable Configuration Block">

  // the following lines will be replaced by docker/configurator, when it runs in a docker-container
  window.ui = SwaggerUIBundle({
    url: "./testswagger.yaml",  // 設定檔位置
    dom_id: '#swagger-ui',
    deepLinking: true,
    presets: [
      SwaggerUIBundle.presets.apis,
      SwaggerUIStandalonePreset
    ],
//    plugins: [  // 這邊是想關掉topbar 不關不影響
//      SwaggerUIBundle.plugins.DownloadUrl
//    ],
//    layout: "StandaloneLayout"
  });

  //</editor-fold>
};
```

### 啟動web server

最後啟動一個web server, 並將dist資料夾放到web server中
這邊簡單使用golang的http server來啟動web server
啟動後可透過 http://localhost:8080/docs/ 來訪問Swagger UI

```golang
package main

import (
	"fmt"
	"net/http"
)

func HelloWorld(rw http.ResponseWriter, r *http.Request) {
	content := fmt.Sprintf("hello world!")
	fmt.Fprint(rw, content)
}

func main() {
	http.HandleFunc("/hello-world", HelloWorld)

	fs := http.FileServer(http.Dir("./dist")) // Swagger UI files handler
	http.Handle("/docs/", http.StripPrefix("/docs/", fs))

	http.ListenAndServe(":8080", nil)
}
```


