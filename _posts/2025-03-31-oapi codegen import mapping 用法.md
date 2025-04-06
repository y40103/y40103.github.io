---
title: "[筆記] oapi-codegen import mapping"
categories:
  - 筆記
tags:
  - Swagger
  - Golang
  - oapi-codegen
toc: true
toc_label: Index
mermaid: true
---

這邊是openapi的文檔拆分功能, 主要是避免單一openapi文檔過大

拆分後的文檔, 可以codegen 輸出成同一 package, 也可以輸出成不同的package  

在codegen中 主要目的是利用這個特性把openapi codegen的輸出拆成多成package,  
這樣可以針對不同package使用不同的middleware, 如部份 handler 加上jwt, 部份不使用jwt

## 目錄說明

- openapi spec config

  - admin/api.yaml: 主要api文檔
  - common/api.yaml: 這邊是只有api response schema, 在主要api文檔中會引用該檔案

- opai-codegen command config

  - 輸出為相同package

    - samepackage/cfg-api.yaml: oapi-codegen command 輸出 主要api文檔的codegen 設定檔
    - samepackage/cfg-user.yaml: oapi-codegen command 輸出 被引用部份的codegen 設定檔

  - 輸出為不同package

    - multiplepackages/admin/cfg.yaml: oapi-codegen command 輸出 主要api文檔的codegen 設定檔
    - multiplepackages/user/cfg.yaml: oapi-codegen command 輸出 被引用部份的codegen 設定檔

```bash

cd import-mapping
tree
├── admin
│   └── api.yaml
├── common
│   └── api.yaml
├── multiplepackages
│   ├── admin
│   │   ├── cfg.yaml
│   │   └── server.gen.go
│   └── common
│       ├── cfg.yaml
│       └── types.gen.go
├── readme.md
└── samepackage
    ├── cfg-api.yaml
    ├── cfg-user.yaml
    ├── server.gen.go
    └── user.gen.go
```

```bash
go mod init import-mapping
go mod tidy
```

### admin/api.yaml

該檔案為主要api文檔, 但response的schema是引用外部的openapi文檔

```yaml
openapi: "3.0.0"
info:
  version: 1.0.0
  title: Admin API
  description: The admin-only portion of the API, which has its own separate OpenAPI spec
tags:
  - name: admin
    description: Admin API endpoints
  - name: user
    description: API endpoint that pertains to user data
paths:
  /admin/user/{id}:
    get:
      tags:
        - admin
        - user
      summary: Get a user's details
      operationId: getUserById
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        200:
          description: Success
          content:
            application/json:
              schema:
                $ref: "../common/api.yaml#/components/schemas/User"  ## 引用外部openapi config
```

### common/api.yaml

```yaml
components:
  schemas:
    User:
      type: object
      additionalProperties: false
      properties:
        name:
          type: string
      required:
        - name
```

### samepackge/cfg-api.yaml

```yaml
package: samepackage
output: server.gen.go
generate:
  models: true
  echo-server: true
output-options:
  # to make sure that all types are generated
  skip-prune: true
import-mapping:
  ../common/api.yaml: "-"
```

### samepackage/cfg-user.yaml

```yaml
package: samepackage
output: user.gen.go
generate:
  #  echo-server: true
  models: true
output-options:
  # to make sure that all types are generated
  skip-prune: true
```

### multiplepackages/admin/cfg.yaml

```yaml
# yaml-language-server: $schema=../../../configuration-schema.json
package: admin
output: server.gen.go
generate:
  models: true
  chi-server: true
output-options:
  # to make sure that all types are generated
  skip-prune: true
import-mapping:
  ../common/api.yaml: "import-mapping/multiplepackages"
```

### multiplepackages/user/cfg.yaml

```yaml
# yaml-language-server: $schema=../../../configuration-schema.json
package: common
output: types.gen.go
generate:
  models: true
output-options:
  # to make sure that all types are generated
  skip-prune: true
```

## samepackage codegen command

分別對兩組檔案產生code

```bash

cd samepackage

oapi-codegen -config cfg-api.yaml ../admin/api.yaml
# 這邊是主要api codegen, 需注意 雖然引用外部openapi yaml, 但並不會將外部引用的部份輸出codegen, 輸出package為 samepackage

oapi-codegen -config cfg-user.yaml ../common/api.yaml
# 這邊是被引用的 openapi yaml, 需單獨產生 codegen, 輸出package為 samepackage
```

## multiplepackages codegen command

```bash

cd multiplepackages/admin/

oapi-codegen -config cfg.yaml ../../admin/api.yaml
# 這邊是主要api codegen, 需注意 雖然引用外部openapi yaml, 但並不會將外部引用的部份輸出codegen , 輸出package為 admin

cd ../../multiplepackages/common/
oapi-codegen -config cfg.yaml ../../common/api.yaml
# 這邊是被引用的 openapi yaml, 需單獨產生 codegen, 輸出package為 common
```
