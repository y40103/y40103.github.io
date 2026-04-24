---
title: "[筆記] AWS mTLS 基本說明"
categories:
  - 筆記
tags:
  - mTLS
  - AWS
  - API-Gateway
toc: true
toc_label: Index
---

客戶目前透過 AWS API Gateway 使用我們的服務，走 mTLS 做雙向驗證。現行架構下 server cert 用 ACM 發、client cert 也是我們的 CA 簽的，整套都是我們自己管。

後來客戶提出想用自己的 CA 憑證，所以我們需要研究可行的配置方式。主要會碰到幾個問題：

- API Gateway 的 TrustStore 要加入客戶的 CA，才能驗證客戶自帶的 client cert
- Server cert 要繼續用 ACM 還是也讓客戶的 CA 簽，需要評估
- AWS API Gateway 要求 domain 所有權驗證，這會影響誰要處理 DNS CNAME

這份筆記先整理 mTLS 的基本原理，再針對 AWS API Gateway 的限制列出可行的方案。

---

## 一、什麼是 mTLS？

**mTLS (Mutual TLS)** 是一種雙向身分驗證機制。

| 類型     | 說明                                        |
| -------- | ------------------------------------------- |
| **TLS**  | Server 出示憑證，Client 驗證 Server（單向） |
| **mTLS** | 雙方都出示憑證，互相驗證對方（雙向）        |

---

## 二、核心元件

### 2.1 憑證階層

```
                    ┌─────────────────────┐
                    │      TrustStore     │
                    │  ┌───────────────┐  │
                    │  │  CA_A (root)  │  │
                    │  ├───────────────┤  │
                    │  │  CA_B         │  │
                    │  ├───────────────┤  │
                    │  │  CA_C         │  │
                    │  └───────────────┘  │
                    └─────────────────────┘
```

**TrustStore（信任庫）**：放信任的 CA 憑證，可以放多個。

### 2.2 Client 與 Server 各自需要的東西

```
┌─────────────────┐                        ┌─────────────────┐
│     CLIENT      │                        │     SERVER      │
│                 │                        │                 │
│  TrustStore     │                        │  TrustStore     │
│  (信任多個CA)    │                        │  (信任多個CA)    │
│                 │                        │                 │
│  ● Key (自己的)  │                        │  ● Key (自己的)  │
│  ● CRT (自己的)  │                        │  ● CRT (自己的)  │
└─────────────────┘                        └─────────────────┘
```

| 元件            | 說明                             |
| --------------- | -------------------------------- |
| **TrustStore**  | 放信任的 CA 憑證（用來驗證對方） |
| **Key（私鑰）** | 自己保管，絕不外傳               |
| **CRT（憑證）** | 含公鑰，可公開，由 CA 簽發       |

---

## 三、mTLS 握手流程

### 3.1 流程總覽

```
    Client                                          Server
      │                                               │
      │─────── 1. ClientHello ──────────────────────►│
      │         (我支援的 cipher suite)               │
      │                                               │
      │◄────── 2. ServerHello ───────────────────────│
      │         (選定的 cipher suite, random)         │
      │                                               │
      │◄────── 3. Certificate ────────────────────────│
      │         (Server 的 CRT，含公鑰)               │
      │                                               │
      │◄────── 4. CertificateRequest ────────────────│
      │         (要求 Client 提供憑證)                │
      │                                               │
      │─────── 5. Certificate ───────────────────────►│
      │         (Client 的 CRT，含公鑰)               │
      │                                               │
      │─────── 6. CertificateVerify ─────────────────►│
      │         (Client 用私鑰簽名)                   │
      │                                               │
      │◄────── 7. Finished ─────────────────────────│
      │                                               │
      │─────── 8. Finished ─────────────────────────►│
      │                                               │
      ════════════════ 握手完成，開始加密通訊 ════════════════
```

### 3.2 各步驟詳細說明

#### Step 1: ClientHello

- Client 告訴 Server 自己支援哪些加密演算法（cipher suite）

#### Step 2: ServerHello

- Server 選定一組 cipher suite，並產生隨機數

#### Step 3: Server 出示憑證

```
         │◄────── Certificate ────────────────────────│
         │         Server CRT                          │
         │         Issuer: CA_X                         │
         │                                              │
         │         【Client 驗證階段】                   │
         │         1. 看 Issuer = "CA_X"                │
         │         2. 在 TrustStore 找 CA_X              │
         │         3. 用 CA_X 公鑰驗證簽名               │
         │         4. 檢查有效期、CN                     │
         │         5. 通過 → 信任這是 Server             │
```

#### Step 4: Server 要求 Client 憑證

- Server 發送 CertificateRequest，要求 Client 提供憑證

#### Step 5: Client 出示憑證

```
         │─────── Certificate ────────────────────────►│
         │         Client CRT                          │
         │         Issuer: CA_Y                         │
         │                                              │
         │         【Server 驗證階段】                   │
         │         1. 看 Issuer = "CA_Y"                │
         │         2. 在 TrustStore 找 CA_Y              │
         │         3. 用 CA_Y 公鑰驗證簽名               │
         │         4. 檢查有效期、CN                     │
         │         5. 通過 → 信任這是 Client             │
```

#### Step 6: Client 證明持有私鑰

```
         │─────── CertificateVerify ──────────────────►│
         │         Client 用私鑰簽名                    │
         │                                              │
         │         【Server 再次確認】                   │
         │         用 Client CRT 裡的公鑰驗證簽名        │
         │         → 確認 Client 真的持有私鑰           │
```

#### Step 7-8: Finished

- 雙方交換 Finished 訊息，握手完成
- 建立對稱金鑰，開始加密通訊

### 3.3 驗證邏輯總結

| 階段   | 動作                    | 目的                      |
| ------ | ----------------------- | ------------------------- |
| Step 3 | Client 驗證 Server CRT  | 確認 CRT 由信任的 CA 簽發 |
| Step 5 | Server 驗證 Client CRT  | 確認 CRT 由信任的 CA 簽發 |
| Step 6 | Server 驗證 Client 簽名 | 確認 Client 真的持有私鑰  |

---

## 四、信任機制說明

### 4.1 基本原則

1. **TrustStore 可以放多個 CA** — 收到憑證時，Issuer 匹配到任一個 CA 就算過關
2. **雙方各自維護 TrustStore** — 只要信任簽發對方憑證的 CA 就好
3. **不同 CA 也能互通** — 前提是雙方 TrustStore 都有放對方的 CA

### 4.2 圖解信任關係

```
    Client                                          Server

    TrustStore:                                     TrustStore:
    ┌─────────┐                                     ┌─────────┐
    │ CA_A    │                                     │ CA_X    │
    │ CA_B    │                                     │ CA_Y    │
    │ CA_C    │                                     │ CA_Z    │
    └─────────┘                                     └─────────┘

    Client CRT (CA_B 簽發)                          Server CRT (CA_X 簽發)

    → Server 用 CA_B 公鑰驗證 Client                → Client 用 CA_X 公鑰驗證 Server
      (前提：Server 的 TrustStore 有 CA_B)            (前提：Client 的 TrustStore 有 CA_X)
```

---

## 五、AWS API Gateway mTLS 配置方案

### 5.1 方案比較

| 方案 | Domain | Server Cert  | Client Cert  | Ownership 驗證 |
| ---- | ------ | ------------ | ------------ | -------------- |
| A    | 業主的 | ACM 公開 CA  | 業主 CA 自簽 | 業主加 CNAME   |
| B    | 業主的 | 業主 CA 自簽 | 業主 CA 自簽 | 業主加 CNAME   |
| C    | 你的   | 業主 CA 簽   | 業主 CA 自簽 | 你自己加 CNAME |

### 5.2 方案 A：全用 ACM（比較省事）

```
Domain: 業主的
Server cert: ACM 發（公開 CA，自動續期）
Client cert: 業主 CA 自簽

                    業主                              你的 API Gateway
              ┌──────────────┐                   ┌──────────────────────┐
              │ Client        │                   │ Server               │
              │               │                   │                      │
              │ Truststore:   │                   │ Truststore:          │
              │ 公開 CA root  │                   │ 業主 CA cert         │
              │ (系統內建)     │                   │ (S3)                 │
              └───────────────┘                   └──────────────────────┘
```

**好處**：Server cert 自動續期，維運成本低。

**業主需提供**：

- CA cert（放 S3 truststore）

**業主需配合**：

- DNS 加 CNAME（證明 domain 所有權）
- 用自家 CA 簽 client cert

### 5.3 方案 B：業主自簽 Server Cert

```
Domain: 業主的
Server cert: 業主 CA 自簽（手動續期）
Client cert: 業主 CA 自簽
```

**AWS 額外要求**：

```
業主 DNS ──── CNAME ────▶ ACM DNS-validated cert（只證明所有權，不參與 TLS）
```

**業主需提供**：

- Server cert + key + CA cert

**缺點**：Server cert 要手動續期

### 5.4 方案 C：你的 Domain + 業主簽 Server Cert

```
Domain: 你的
Server cert: 業主 CA 簽你的 domain（手動續期）
Client cert: 業主 CA 自簽
```

**Ownership 驗證**：

```
你的 DNS ──── CNAME ────▶ ACM DNS-validated cert（只證明所有權，不參與 TLS）
```

**業主需提供**：

- Server cert（CN=你的domain）+ key + CA cert

**你自行處理**：

- DNS 加 CNAME（ownership 驗證）
