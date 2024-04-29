---
title: "[筆記] PostreSQL 權限控制"
categories:
  - 筆記
tags:
  - Postgres
toc: true
toc_label: Index
---

Postgres 權限控制隨筆


## Postgres 權限概念

postgres 權限控制核心是基於RBAC, 利用角色(role)控制權限  

簡單說就是創建一個role, 設定該role可以做哪些事  

創建user後, 就把該role掛在user上 ,   

postgres中 user與role 實際上是同一種單位 只差別在有無登錄權限

以下無論創建user或是 role, 都是用role的指令

賦予某role權限的user, 所執行的動作等同該role

---

## 基本操作隨筆 

```
set search_path to <schema_name>;
# 切換當前schema


\du+
# 可以顯示 role的系統權限

\dn
# 檢視當前database下 所有schema

\d+
# 檢視當前 schema下所有 table

\dt <schema_name>.*
# 檢視某 schema下 所有table



drop scheam <schema> (cascade)
# 刪除schema
# 無 cascade 若schema中有table 會無法刪除
# 有 cascade 會把底下相依 table 全部刪除

SELECT table_name,table_schema
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema');
# 檢索當前db 所有table 與其 schema

SELECT * FROM information_schema.table_privileges where grantor='test02';
# 查看某user 對各table權限

```



## RBAC

創建/修改/刪除/檢索 role,  且可定義基本系統權限 

### 檢索user  

```
\du
```

```
select * from pg_user;
```
---

### 檢索role 詳細權限


```
select * from pg_roles
```

---
### 創建role


```
create role <role_name>
<授權選項1>
<授權選項2>
...

```


```
create role test01 LOGIN ENCRYPTED password 'test01'
```


常用授權選項

|權限|說明|預設|
|:--:|:--:|:--:|
|SUPERUSER|超級用戶 任何操作皆不需被鑒權|NOSUPERUSER
|CREATEDB|可以創建資料庫|NOCREATEDB
|CREATEROLE|可以創建role權限,謹慎使用,除了superuser授權,其他都可創建|NOCREATEROLE
|LOGIN|可否被登錄|NOLOGIN
|ENCRYPTED PAWSSWORD '<pass_word>'| 設定密碼, 需要有login權限才能被設定|NULL|

預設是指不設定時 該權限狀態

[其他可以參考](https://docs.postgresql.tw/reference/sql-commands/create-role)

---
### 修改role

createuser,superuser 權限可以修改

```
alter user test01 rename to test02;
## 修改用戶名稱

alter user test01 password 'test01_password';
# 修改密碼

alter user test01 createrole;
# 修改用戶權限

```

---
### 刪除role

只有superuser,createuser 可以刪除

刪除前 需先刪除相依於此role的對象或權限

```
drop role if exists <role_name>;
## 登陸
```


```bash
dropuser -U postgres -h 127.0.0.1 test01
## 未登錄terminal
# test01 被刪除user
# postgres root帳號
# 127.0.0.1 db host

```

---
### 賦予role

賦予後, 該user就可以擁有該role的權限

```
grant <role> to <某user>
##　將賦予某user <role>權限
```

```
grant postgrs to test01
# 使user test01 擁有 user postgres權限
```

須注意 被賦於權限後 該user 權限不會馬上生效

該user需執行, 類似linux中 su操作

```
set role <role>:
```


### 撤銷賦予的role

將user test01的 role postgres 拔掉 
```
revoke postgres from test01;
```




## 物件權限

Postgres的物件單位由上至下可分

- Database
- Schema
- Table
- Row

物件層級的授權, 是表示可以操作該物件層級以下的所有單位, 但不包含自己

e.g.   
Schema 層級授權 , 表示你可以操作該schema下  
table 創建或刪除,  
table中的所有row的CRUD,  
但不能操作其他schema, 只能在授權的schema中活動

詳細物件授權可參考 [grant](https://docs.postgresql.tw/reference/sql-commands/grant)

---
### Database 層

資料庫層級的授權, 是表示可以操作Database以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

- Schema
- Table
- Row

#### Create Database Authorization

create 一般內部資源控制都授權可以執行

```
grant create on database <某db> to <userB>
```
#### Revoke Database Authorization

```
revoke create on database <某db> from <userB>
```


---
### Schema 層

Schema層級的授權, 是表示可以操作Schema以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

需注意 因無法檢索schema層級, 推薦把預設schema鎖在授權的schema, 這樣檢索內部資源才會有辦法看到


- Table
- Row

#### Create Schema Authorization

推薦all,  沒有用 usage 會無法檢索到table, 看不到 有操作權限也會無法操作

```
grant <關鍵字> on schema <schema_name> to <user_name>
```

|關鍵字|功能|
|:--:|:--:|
|usage|檢索|
|temportary|可創建暫存table, 但在該Session結束會被刪除|
|create|創建|
|all|所有權限|

#### Revoke Schema Authorization

```
revoke create on schema <某db> from <userB>
```


---
### Table 層

簡單說是可以在某schema下 針對table row操作 給予權限

- row


#### Create Table Authorization

需注意schema檢索權限是必須的 否則無法看到table之類的 也等於無法指定 會沒辦法操作內部資源

```
grant usage on schema <schema_name> to <user_name>
## 需給於schema 檢索權限 , 蠻反人類的 即使你有權力操作table內容 但你看不到schema下的東西 一樣不能操作

grant <關鍵字> on <schema_name>.<table_name> to <user_name>
## 給予某user 某schema中 已存在的某table 關鍵字 權限

```

|關鍵字|功能|
|:--:|:--:|
|INSERT|創建|
|SELECT|讀取|
|UPDATE|更新|
|DELETE|刪除|

#### Revoke Table Authorization

```
revoke <關鍵字> on <schema_name>.<table_name> from <user_name>
```


---

## 實際情況

- 若你這個user 並沒有createdb權限, 假設userB有createdb權限,  你被賦予userB這個role  
   你使用指令創建db, 實際上owner依然是userB, 但因你有userB的role,  身份上 該user=userB  

- role被創建的時候, 可以被賦於基本的系統權限, 如 可否登陸, 可否創建role, 可否創建db 等等....  

- database內的細部權限控制是屬於 物件層級的權限控制  
   如A 有crestedb的基本系統權限, B只有登陸權限 啥都沒有  
   A創建一個名為 A_DB的資料庫 , 他只想給B操作這個A_DB內的內容  
   就可以用物件層級的授權指令 給B A_DB的 db物件層級 權限,  
   他就可以操作 該db 中 schema, table , row 的操作 

- 若有一個C 你想要給他某個schema內的操作權限  
   就可以用物件層級的指令 給C 某個schema的 schema物件層級all權限  
   他就可以對schema內的 table, row 操作  

- 今天有一個D 你想要給他跟C相同的權限 , 可以找一個user是有 createrole or superuser 權限, 將role C 直接給 D,   

- 有一個E 你只想給他讀取某table中的row  
   你就可以用物件層級的指令 給C某schema 物件層級的檢索權限 table物件層級的讀權限, 他就只能讀某table的內容  

- 然而今天有兩個人 E,F 用物件層級權限給使其可操作 A_DB  
   但實際上兩者控制是無法相互干涉的 E在該DB下創建的資源 owner為E, F創建的為F  
   若非該onwer 是無法控制該單位的  類似線上遊戲概念 大家登陸同一款遊戲, 各玩各的 他人是沒有權力干涉你的帳號的  

- 有最上層物件權限不代表有權限可以訪問下層其他人物件, 如G有 G_DB為 onwer 但給 H database物件權限, 他可以在G_DB創建 schema/table/row,  他建了一個 H_schema ,  H又給了I H_schema的權限, I 在 H schema下 創建  I_table,   此時H與G都是無法訪問操作 I_table的 ,  需要 I 給 H I_table權限, H 才能訪問操作 I_table,  而G要操作 I_table 則需要滿足   H給 H_schema權限, I也給 I_table權限  

- superuser = 不會觸發鑒權機制 因此可以做任何操作 
 




## 參考範例

### Create Database
```
create database <database name>


REVOKE ALL PRIVILEGES ON DATABASE <database> FROM public;
```



### Create Read-Only User
```
create role <username> login encrypted password '<mypassword>'

Grant connect on database <database name >to <username>

GRANT pg_read_all_data TO <username>;

```


1
### Create Dev User



```
create role <username> login encrypted password '<mypassword>'

Grant all on database <database name> to <username>

```


```
grant usage on schema <schema_name> to <user_name>

grant all on <schema_name>.<table_name> to <user_name>

```


