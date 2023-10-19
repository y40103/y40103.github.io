---
title: "Django 基本架構/操作"
categories:
  - Python
tags:
  - Django
  - ORM
toc: true
toc_label: Index
---

Django 基本使用與ORM操作隨筆 





## 初始化設定

初始化django

```bash

django-admin startproject mysite

```

啟動server

```bash

python3 manage.py runserver

```

創建app

```bash

python3 manage.py startapp

```

model映射至db

```bash

python3 manage.py migrate

```

---

## URL

```text

protocol:hostname:<port>/<path>?<queryKey1>=<queryValue1>#<fragment>

e.g.

https://www.abc.com.tw/book?page=1#subject

protocal=https
hostname=www.abc.com.tw
path=book
queryKey1=page
queryValue1=1
fragment=subject 
#錨點 可理解成頁面的位置標籤 你點進去該往業 會直接移動到該錨點標籤的地方
# e.g.
# https://docs.djangoproject.com/en/3.2/
# https://docs.djangoproject.com/en/3.2/#performance-and-optimization
```

request 進來後 django處理該 request url 方式

1. 從django root_url 找 e.g mysite/urls.py
2. 載入 urlpatterns 變數
3. 批配 urlpatterns path
4. 批配成功 調用對應的view處理請求 返回response
5. 批配失敗 返回404

需注意 urlpatterns 會由上至下批配 若有同時滿足的path, 上面的優先級會比較高

舉例 (注意 path 結尾必須為/)

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('page/2003/',views.page_2003),
]

```

---

## 路徑參數

於url.py中設定 , 也可設定多個變數

```python

urlpatterns = [
    path('admin/', admin.site.urls),
    path("page/<int:page_num>", views.page_view)
]


```

於views.py中

```python
def page_view(request, page: int):
    msg = f"Hello World {page}"
    return HttpResponse(msg)

```

[測試](http://127.0.0.1:8000/page/5)

類型

1. str: 批配/之外的非空字串 e.g. page/index
2. int: 批配0 or 任何整數 返回int e.g. page/1
3. slug: 批配任意ascii字母 數字 連字符號 e.g. page/this-is-django
4. path: 批配非空字段 包含"/" e.g. page/a/b/c 會批配/a/b/c

## regex路徑參數

urls.py

```python\

    urlpatterns = [
        path('admin/', admin.site.urls),
        path("page/<int:page_num>", views.page_view),
        re_path(r"regex_test\/(?P<regex_get>\d\d\d\d_123)", views.regex_page_view),
    ]


```

views.py

```python
def regex_page_view(request, regex_get: str):
    msg = f"Hello World {regex_get}"
    return HttpResponse(msg)
```

(?P<var_name>\d\d\d\d_123)

regex中
()表示group
P<別名> 表示group中抓出來的值 為 別名=該值

/需要\轉譯, 單/有其他意義 因此需要\轉譯 e.g. \/ = '/'

?P<regex_get_var_name> 可以先當作沒看到 只是把regex批配的group給個變數名稱

實際上就是批配 regex pattern: regex_test\/(\d\d\d\d_123)

測試範例: regex_test/1111_123

實際上就是抓到 1111_123的部份,

批配結果為 regex_get=1111_123 傳入views

---

## Request 屬性(查詢參數,BODY)

views.py 中 傳入的request 屬性解析

```python

def req_test(request: HttpRequest):
    print("path info is ",request.path_info)
    print("method is ",request.method)
    print("querystring is ",request.GET)
    print("full path is ",request.get_full_path())
    # print("metadata is ",request.META) ## 超大字典
    print("client ip",request.META["REMOTE_ADDR"])

    return HttpResponse("test request field")

## 輸出

# (path info is  /req/1)

# (method is  GET)

# (querystring is  <QueryDict: {'a': ['1'], 'b': ['2']}>)

# (full path is  /req/1?a=1&b=2)

# (client ip 127.0.0.1)

    
```

[測試](http://127.0.0.1:8000/req/1?a=1&b=2)

path info: host後面的路徑
method: 請求種類
GET: 為檢視查詢參數 會用字典的形式返回 ## 若查詢參數有複數個相同key 可用 GET.getlist('<key>') 會返回list
POST: 為檢視Content-Type: application/x-www-form-urlencoded , body的內容

e.g. username=johndoe&password=secretpassword

```bash

curl -X POST http://your-django-app-url/your-view-url/ -d "username=myusername&password=mypassword" -H "Content-Type: application/x-www-form-urlencoded"

```

FILES: 回傳類似字典, 包含所有上傳文件的訊息
COOKIES: 回傳字典 key:value
body: 字符串 body的內容
scheme: 請求協議 http or https
request.get_full_path(): 請求的完整路徑

---

## Response

格式也是

1. 起始行
2. headers
3. body

常見狀態碼

- 200: 請求成功
- 301: 永久重定向
- 302: 臨時重定向
- 404: 請求的資源不存在 (參數錯誤就有可能導致)
- 500: server錯誤

Django response格式

HttpResponse(content=<body>,content_type=<body格式>,status=<狀態碼>)

常見content_type

- text/html (html文件)
- text/plain (純文本)
- text/css (css文件)
- text/javascrpt (js文件)
- multipart/form-data (文件提交)
- application/json (json傳輸)
- application/xml (xml文件)

## Application

新增django application

```bash

python manage.py startapp <app_name>

```

會在django目錄下 創建一個子目錄

```bash

python manage.py startapp <app_name>

bookstore
├── admin.py
├── apps.py
├── __init__.py
├── migrations
│   └── __init__.py
├── models.py
├── tests.py
└── views.py
```

---

## Router

main router 有include的方法

mysite/urls.py

```python

urlpatterns = [
    path('admin/', admin.site.urls),
    path("page/<int:page_num>", views.page_view),
    re_path(r"regex_test\/(?P<regex_get>\d\d\d\d_123)", views.regex_page_view),
    path("req/1", views.req_test),

    ## 分佈式
    path("app1/",include("app1.urls")),
    path("app2/", include("app2.urls"))

]
```

app1/urls.py

```python

urlpatterns = [
    path("views/<int:num>", views.app_views)
]

## <int:num> 為路徑參數 可為任藝數字
```

實際urls為
localhost:8000/app1/views/100


---

## Template

各app目錄下的 templates/ 會預設就是 template的預設路徑之一
若要新增自定義template目錄 可至mysite/settings.py 新增 DIRS
這邊自創一個目錄在最外層

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR,'templates')], ## 在這邊加入template目錄 預設為'DIRS': [], 這邊自創一個目錄在最外層
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

若各app中的預設templates目錄中  
有與自己新增的template目錄有相同名稱的html檔案
會以自己新增的為優先
e.g. mysite/templates/ 與 mysite/app2/templates/ 前者是自定義 後者是系統預設 因此會以前者為主

http://localhost:8000/app2/views/100

template/test_template.html 與 test_template1.html
預設子目錄已經有test_template.html 可在外層來回修改測試
可發現 會以外層為主

---

## django-extensions

補充一個django的擴充套件, 可以方便查看後面orm操作, 底層sql 實際運行內容

```bash

pip install django-extensions

```

於主app註冊

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    "app1",
    "app2",
    "bookstore",
    'django_extensions', ## <<<<
]
```
進入Django shell
```python
python3 manage.py shell_plus --print-sql
```

---

## ORM

### 優點

- 資料庫種類解耦
- 不需要使用SQL, 可用python lib開發 開發效率高

### 缺點

- 複雜業務 性能成本高
- python lib 轉化 SQL, 會有性能損失

### 映射架構 ORM vs DataBase

- model_class <-> table
- object <-> row
- attribute <-> field

---

### 映射schema

創建model(等同db中的table)  
於models.py下新增

```python

class Book(models.Model):
    title = models.CharField("title", max_length=128, default="")
    price = models.DecimalField("price", max_digits=7, decimal_places=2)

```

利用model產生lib 並保存於migrations目錄

```python

python3 manage.py makemigrations

```

調用lib 將我們自定義的model 真實映射於資料庫

```python

python3 manage.py migrate

```

---

### 資料庫與model field 對應

- BooleanField() <> tinyint(1) 實際上資料庫是存 0 or 1
- CharField() <> varchar 必須指定max_length
- DateField() <> date 參數: auto_now: 每次更新 自動設置當前時間(true/false), auto_now_add:
  當地一次被創建,自動設置當前時間(true/false), default: 設置當前時間 格式為 YYYY-mm-dd ### 以上參數只能三選一
- EmailField() <> varchar 內部會用regex做限制
- IntegerField() <> int 整數
- FloatField() <> double 浮點數
- DecimalField() <> decimal(x,y) 需要精準數字的類型 e.g.金錢, 參數: max_digit: 位數總數 包含小數點部份
  該值於大於decimal_place, decimal_places: 小數點後的位數
- ImageField() <> varchar(100) 保存圖片存儲路徑 非圖片本身
- TextField <> longtext 內容長度不固定的字串

---

### field 其他屬性

- primary_key (true,false) 設置為true 則是該field為主健 ## 需注意 若無設置primary key, django會自動新增一個id field,
  作為primary key
- blank (true,false) 設置為true 表示該field可以為空 , false表示不能為空, 預設就是false
- null (true,false) 設置為true 表示可以為null, 預設false, 若為true, 建議用default=null配合, ## 建議不要使用null,
  對資料庫操作會有性能影響
- db_index (true,false) 設置為true, 表示設置為索引
- unique 設置true, 表示該field的值 皆為唯一
- db_column 指定column名稱 不指定的話 預設為 field_name
- verbose_name 這邊是指定後台admin的顯示名稱

```python
name = modol.CharField(max_length=30,unique=True,null=False,db_index=True)
```

```python

class Book(models.Model):
    title = models.CharField("title", max_length=128, default="", unique=True)  ## 第一個field 是admin後台顯示的名稱
    price = models.DecimalField("price", max_digits=7, decimal_places=2)  ## 定價
    publish = models.CharField("jump", max_length=100, default="")  ## 出版社
    market_price = models.DecimalField("market_price", max_digits=7, decimal_places=2, default=0.0)  ## 實際售價
    author = models.ForeignKey("Author", on_delete=models.CASCADE, default=1)


class Author(models.Model):
    name = models.CharField("name", max_length=50)
    age = models.IntegerField("age", default=1)
    email = models.EmailField("email", null=True)

```

---

### model Meta屬性

通常是table本身的屬性

e.g. table_name, 預設是 <app>_<model_name> 若要自定義 則可修改 db_table = XXXX

```python

class Book(models.Model):
    title = models.CharField("title", max_length=128, default="", unique=True)  ## 第一個field 是admin後台顯示的名稱
    price = models.DecimalField("price", max_digits=7, decimal_places=2)  ## 定價
    publish = models.CharField("jump", max_length=100, default="")  ## 出版社
    market_price = models.DecimalField("market_price", max_digits=7, decimal_places=2, default=0.0)  ## 實際售價
    author = models.ForeignKey("Author", on_delete=models.CASCADE, default=1)

    class Meta:
        db_table = "book" ## 預設為 <app_name>_Book 這邊修改為 book

    def __str__(self):  ## print 可視化
        return f"title:{self.title} price:{self.price} publish:{self.publish} market_price:{self.market_price} author:{self.author}"


class Author(models.Model):
    name = models.CharField("name", max_length=50)
    age = models.IntegerField("age", default=1)
    email = models.EmailField("email", null=True)

    class Meta:
        db_table = "author" ## 預設為 <app_name>_Author 這邊修改為 author

    def __str__(self):  ## print 可視化
        return f"author_id: {self.pk} name: {self.name} age: {self.age} email: {self.email}"

```

---

### 新增資料

1. create

```python
<MyModel>.object.create(field1=XXX,field2=OOO...) 
```

成功: 返回創建資料
失敗: 異常

進入 python

```bash
python3 manage.py shell
```

```python
# 需逐row 分開輸入
from bookstore.models import Book
from bookstore.models import Author

Author.objects.create(name="foo",age=20,email="123@google.com")

Book.objects.create(title="harry potter", publish="jump",price=100,market_price=80,author=Author.objects.get(name="foo"))
Book.objects.create(title="harry potter2", publish="jump",price=100,market_price=80,author=Author.objects.get(name="foo2"))

```

2. obj

```python
obj = <MyModel>(field1=XXX,field=OOOO....)
obj.save()
```

進入 python

```bash
python3 manage.py shell
```

```python

b = Book(title="harry potter3", publish="jump",price=100,market_price=80,author=Author.objects.get(name="foo"))

b.save()

```

需注意 若有 foreign key的field , 需帶入 model的 instance, 如上方author field

```python
>>> type(Author.objects.get(name="foo2"))
<class 'bookstore.models.Author'>
```

---

### 全查詢(無任何條件篩選)

這邊返回結果都是 querySet

querySet 用python比較接近的比喻概念 就是list

```python
## QuerySet 雖然不等於 但類似List , 為可迭代物件

返回類似等於 List[model_obj] or List[ tuple[model_info] ]

django 查詢還有一種返回方式 為單一 model_obj
若返回內容大於1 則直接報錯

簡單說 查詢後 返回為 複數row 必須是 QuerySet 的查詢方式
若用返回 model_obj的查詢方式 必須確定滿足條件的只有一個 row 否則報錯

```

1. all()  
   等同 select * from \<table\> 的效果
   返回 QuerySet 內部為自定義的Model物件 屬性就是 model的field
   model物件 可以理解成 table的row, 屬性為 所有field name

```python
# 需逐row 分開輸入
from bookstore.models import Book
books = Book.objects.all()
print(books.query)
for book in books:
  print(book)

```

輸出

```python
## orm to raw sql
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book"

title:harry potter price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
title:harry potter2 price:100.00 publish:jump market_price:80.00 author:author_id: 2 name: foo2 age: 20 email: 123@google.com
title:harry potter3 price:60.00 publish:jump market_price:50.00 author:author_id: 1 name: foo age: 20 email: 123@google.com

```

2. values()  
   等同 select field1,field2.. from \<table\>  
   返回 QuerySet 內部為自定義的Model物件 屬性就是 model的field
   model物件 可以理解成 table的row, 屬性為 所有field name

```python
# 需逐row 分開輸入
from bookstore.models import Book
books = Book.objects.values('title','publish')
print(books.query)
for book in books:
  print(book)

```

輸出

```python
## orm to raw sql
SELECT "book"."title", "book"."publish" FROM "book"

{'title': 'harry potter', 'publish': 'jump'}
{'title': 'harry potter2', 'publish': 'jump'}
```

3. values_list()  
   等同 select field1,field2.. from \<table\>  
   返回 QuerySet 內部為 tuple

```python
# 需逐row 分開輸入
from bookstore.models import Book
books2 = Book.objects.values_list('title','publish')
print(books2.query)
for book in books2:
  print(book)
```

輸出

```python
# orm to raw sql
SELECT "book"."title", "book"."publish" FROM "book"

('harry potter', 'jump')
('harry potter2', 'jump')

```

4. order_by()

會對查詢結果做 order_by , 預設是升序, 降序則需再前面加上 -  
等同 select * from \<table\> order by field1  
加上 -  
等同 select * from \<table\> order by field2 desc

```python
# 需逐row 分開輸入
from bookstore.models import Book
books3 = Book.objects.order_by('-price') #降序
print(books3.query)
for book in books3:
  print(book)
```

輸出

```python
# orm to raw sql
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book" ORDER BY "book"."price" DESC

title:harry potter price:100.00 publish:jump market_price:80.00 author:Author object (1)
title:harry potter2 price:100.00 publish:jump market_price:80.00 author:Author object (2)
title:harry potter3 price:60.00 publish:jump market_price:50.00 author:Author object (1)
```

也可以只用query特定field 組合 order by 只是實際上sql 還是select *
性能上沒幫助  
等同select from \<table\> order by field1

```python
# 需逐row 分開輸入
from bookstore.models import Book
books = Book.objects.values('title','publish').order_by("title")
# 也可以 books = Book.objects.order_by("title").values('title','publish')
# 順序不影響 實際query都是 select * 


print(books3.query)

for book in books3:
  print(book)
```

```python
# orm to raw sql
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book" ORDER BY "book"."price" DESC

title:harry potter price:100.00 publish:jump market_price:80.00 author:Author object (1)
title:harry potter2 price:100.00 publish:jump market_price:80.00 author:Author object (2)
title:harry potter3 price:60.00 publish:jump market_price:50.00 author:Author object (1)

```

---

### 條件查詢 - QuerySet

這邊為返回類型為 QuerySet的條件查詢 可返回複數row的內容

1. filter()

```python
<MyModel>.objects.filter(<field1>=XXX,<field2>=OOO)
```

返回querySet
多個field, 為 and 連接, 會返回滿足所有條件的對象

```python
# 需逐row 分開輸入
from bookstore.models import Book
books = Book.objects.filter(price="100")
print(books.query)
for book in books:
    print(book)

from bookstore.models import Author
authors = Author.objects.filter(name="foo2",age=20)
print(authors.query)
for author in authors:
    print(author)
```

輸出

```python
## orm to raw sql
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book" WHERE "book"."price" = 100

title:harry potter price:100.00 publish:jump market_price:80.00 author:Author object (1)
title:harry potter2 price:100.00 publish:jump market_price:80.00 author:Author object (2)

## orm to raw sql
SELECT "author"."id", "author"."name", "author"."age", "author"."email" FROM "author" WHERE ("author"."age" = 20 AND "author"."name" = foo2)

name: foo2 age: 20 email: 123@google.com
```

2. filter() 非等值查詢

#### 等值查詢

```python
# 需逐row 分開輸入
from bookstore.models import Book
books1 = Book.objects.filter(price__exact="100")
print(books1.query)
# 等同 price="100"
```

輸出

```python
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book" WHERE "book"."price" = 100
```

#### 模糊查詢

```python
# 需逐row 分開輸入
from bookstore.models import Book
books2 = Book.objects.filter(title__contains="potter")
print(books2.query)
# 等同  select * from book where title like '%pottet%'
# 名字包含 potter的內容

## __startswith = "xx" 以xx開頭 
## __endswith = 'oo' 已oo結尾
```

輸出

```python
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book" WHERE "book"."title"::text LIKE %potter%

```

#### 大於

```python
# 需逐row 分開輸入
from bookstore.models import Author
authors = Author.objects.filter(age__gt=50)
print(authors.query)

## __gt=$$  , 表示大於$$      ## get greater than
## __gte=OO , 表示大於等於OO   ## gte = greater than or equal
## __lt=XX  , 表示小於xx      ## lt = less than
## __lte=@@ , 表示小於等於@@   ## lte = less than or equal

```

輸出

```python
SELECT "author"."id", "author"."name", "author"."age", "author"."email" FROM "author" WHERE "author"."age" > 50
```

#### in 列表中條件

```python
# 需逐row 分開輸入
from bookstore.models import Book
books3 = Book.objects.filter(title__in=["harry potter","harry potter2"])
print(books3.query)
```

輸出

```python
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book" WHERE "book"."title" IN (harry potter, harry potter2)
```

#### range 數字範圍區間條件

```python
# 需逐row 分開輸入
from bookstore.models import Book
from decimal import Decimal
books4 = Book.objects.filter(price__range=(Decimal("50"),Decimal("100")))
print(books4.query)
```

輸出

```python
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book" WHERE "book"."price" BETWEEN 50 AND 100
```

3. exclude

```python
<MyModel>.objects.exclude(<field1>=XXX,<field2>=OOO)
```

等同
select * from \<table\> where field1 != XXXXX

```python
# 需逐row 分開輸入

from bookstore.models import Author
authors = Author.objects.all()
for author in authors:
    print(author)


from bookstore.models import Book
books2 = Book.objects.exclude(author=1)
print(books2.query)
for book in books2:
    print(book)

```

```python

## all author
author_id: 1 name: foo age: 20 email: 123@google.com
author_id: 2 name: foo2 age: 20 email: 123@google.com


## orm to raw sql
SELECT "book"."id", "book"."title", "book"."price", "book"."publish", "book"."market_price", "book"."author_id" FROM "book" WHERE NOT ("book"."author_id" = 1)
## 查找 author_id != 1, 表示找foo2

title:harry potter2 price:100.00 publish:jump market_price:80.00 author:name: foo2 age: 20 email: 123@google.com

```

---

### 條件查詢 - model_obj

這邊為返回類型為 model_obj的條件查詢  
由於model_obj只能代表一個row的內容  
若返回多個row或是結果為空 會直接報錯 , 可用 try except 捕捉

1. Get()

```python
<MyModel>.objects.get(field1=XXXX)
```

```python
from bookstore.models import Book
book = Book.objects.get(title="harry potter")
print(book)
type(book)

```

輸出

```python
title:harry potter price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com

<class 'bookstore.models.Book'>
```

可以發現 返回就是之前 QuerySet中的單位物件


---

### 更新

django更新操作步驟為

查詢>修改>儲存 三個步驟來操作

- #### 單row更新

有點類似 obj.save() 新增的方式  
只是這邊是 先撈出之前的某個row, 直接通過field修改
最後再儲存 映射於實際資料庫

單row 就是撈出 model物件類型 進行修改

```python
# 需逐row 分開輸入
from bookstore.models import Book
book = Book.objects.get(title="harry potter3") 
## 這邊返回是 model instance , 若是filter 取得 QuerySet 則要索引其中之一進行修改
book.title = "harry potter33"
book.save()
```

```python
# 需逐row 分開輸入
from bookstore.models import Book
books = Book.objects.all()
for book in books:
  print(book)
```

輸出

```python
title:harry potter price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
title:harry potter2 price:100.00 publish:jump market_price:80.00 author:author_id: 2 name: foo2 age: 20 email: 123@google.com
title:harry potter4 price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
title:harry potter33 price:60.00 publish:jump market_price:50.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
```

可以看到已經修改成功

#### 多row更新

多個row, 就是撈出 QuerySet 統一進行修改

一樣也是先撈出

這邊修改與儲存會是一起的 QuerySet利用update指令進行更新

```python
# 需逐row 分開輸入
from bookstore.models import Book
books = Book.objects.filter(price__gte=100)
for book in books:
  print(book)
```

輸出

```python
title:harry potter price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
title:harry potter2 price:100.00 publish:jump market_price:80.00 author:author_id: 2 name: foo2 age: 20 email: 123@google.com
title:harry potter4 price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
```

```python
books.update(price=120)
books = Book.objects.filter(price__gte=100)
for book in books:
  print(book)
```

輸出

```python
3 ## update 返回被更新的row 數目

title:harry potter price:120.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
title:harry potter2 price:120.00 publish:jump market_price:80.00 author:author_id: 2 name: foo2 age: 20 email: 123@google.com
title:harry potter4 price:120.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
```

---

### 刪除

刪除也是依單row 或是 多個row刪除, 但實際用法是一樣的  
model物件與querySet都有delete()方法 可以直接刪除  
依然需注意get若返回多個row 或 空結果 會報錯 可用 try except處理

#### 單row刪除

```python

from bookstore.models import Book
book = Book.objects.get(title="harry potter4")
book.delete()
```

輸出

```python
(1, {'bookstore.Book': 1}) #返回刪除row數目
```

#### 多row刪除

```python
from bookstore.models import Book
book = Book.objects.filter(author=1)
book.delete()
```

輸出

```python
(2, {'bookstore.Book': 2}) #返回刪除row數目
```

---

### 映射關係

這邊主要是處理SQL中 foreign key 連動的方式  

假設A table(設置某field foreign key) 關係 B table, B 某row被刪除後 ,  A關聯數值怎麼處理
e.g. A = Book, B = Author

雖然在django有 一對一 多對一 一對多, 但實際上都是foreign key  

A table中 設置 foreign key, on_delete 參數選項
- models.CASCADE: B被刪除後 , A該關聯也會被跟著刪 (非db機制 而是django層自己實現)
- models.PROTECT: B有被其他關聯 , 會無法被刪除, 需先將其他刪除 如 A被刪除 才能刪除B
- set_null: B被刪除 , A foreign key 欄位數值會變null
- set_default: B被刪除 , A foreign key 欄位數值會變預設值

#### 一對一

model.py 新增範例
```python

### 以下為 一對一 範例

# 讀者
class Reader(models.Model):
    name = models.CharField("讀者",max_length=50)

    class Meta:
        db_table = "reader" ## 預設為 <app_name>_Author 這邊修改為 author

# 老婆
class ReaderWife(models.Model):
    name = models.CharField("讀者老婆",max_length=50)
    husband = models.OneToOneField(Reader,on_delete=models.CASCADE) #

```

django foreign key 正逆向檢索操作
```python
from bookstore.models import Reader
from bookstore.models import ReaderWife
reader = Reader.objects.create(name="boo")
wife = ReaderWife.objects.create(name="boowifi",husband=reader)
print(wife.husband.name) ## 主表藉由外鍵訪問 關聯表  A > B  , 由foreign key field 訪問
print(reader.readerwife.name) ## 被關聯表 也可反向檢索  B > A ,通靈 主表名稱 由主表名稱作為field 反向訪問

```

#### 一對多

先創建一 再創建多

```python
### 以下為 一對多 範例

# 一
class School(models.Model):
    name = models.CharField("學校名稱",max_length=50)

    class Meta:
        db_table = "school"

# 多
class Student(models.Model):
    name = models.CharField("學生名稱",max_length=50)
    school = models.ForeignKey(School,on_delete=models.CASCADE)

    class Meta:
        db_table = "student"
    
    def __str__(self):
        return f"student_name: {self.name} school: {self.school}"
```

```python
from bookstore.models import School
from bookstore.models import Student
sch1 = School.objects.create(name="s1 school")
stu1 = Student.objects.create(name="s1_stu1",school=sch1)
stu2 = Student.objects.create(name="s1_stu2",school=sch1)
```
一對多 學校 > 學生 , 這邊由一的任何一個 檢索所有學生 藉由 <多>_set 的方法
類似group_by去檢索所有學生
```python
from bookstore.models import School
from bookstore.models import Student
sch = School.objects.get(name="s1 school")
stu_all = sch.student_set.all()
for stu in stu_all:
  print(stu)
```

#### 多對多

多對多 實際上無法用foreign key 直接關聯對方, 因兩者相互依賴  
django中會自動建立 第三個table 來維持相互關聯性  
兩者為多對多 建立多對多關係 隨便挑選一個建立就可以 效果相同

如銀行中 客戶與信用卡種類
一個客戶有可能有多張卡
一張卡種也可能被多個客戶申請

```python
class Client(models.Model):
    name = models.CharField("庫戶姓名",max_length=50)

    class Meta:
        db_table = "client"

class Credit_card(models.Model):
    name = models.CharField("信用卡種類",max_length=50)
    client = models.ManyToManyField(Client)

    class Meta:
        db_table = "credit_card"
        
## 實際db創建
## public | client                            | table    | postgres | permanent   | heap          | 8192 bytes | 
## public | credit_card                       | table    | postgres | permanent   | heap          | 8192 bytes | 
## public | credit_card_client                | table    | postgres | permanent   | heap          | 8192 bytes | 
```

可用客戶找到信用卡table 在創建信用卡
如下
```python
from bookstore.models import Client
from bookstore.models import Credit_card
cli1 = Client.objects.create(name="foo")
cli2 = Client.objects.create(name="boo")

## foo與boo 同時都有申辦 card1
## 下面建立透過client 建立 card1 

card1 = cli1.credit_card_set.create(name="世界卡")
## 藉由由 client 關聯信用卡,  再去 創建 信用卡
## 該操作會在credit_card創建 世界卡之外 還在 多對多的第三table創建對應關係
## client <> card to create card
## 返回創建的 model obj: <Credit_card: Credit_card object (1)>

cli2.credit_card_set.add(card1)
## cli2 直接用credit_card中的 該row model 創建
## 創建完後 也會在多對多的第三table創建對應關係

```
也可用信用卡找到客戶 在創建客戶

正反檢索 有放多對多field的為正

```python
from bookstore.models import Client
from bookstore.models import Credit_card

# 正向 卡>客戶
card1 = Credit_card.objects.get(name="世界卡")
## 用credit card 找到其中一個張卡 用該卡查找所有用戶
all_client = card1.client.all() ## card1.client為objects 可all() or filter()
for cli in all_client:
  print(cli.name)

# 反向 客戶 > 卡
client = Client.objects.get(name="foo")
# 用Client 找到其中一個客戶 查找該客戶有你些信用卡 
all_card = client.credit_card_set.all() ## client.credit_card_se為objects 可all() or filter()
for card in all_card:
  print(card.name)

```

---
### F

其實就是實現某個欄位數值 val=val+1 的資料庫操作

在django orm中 如果不靠F這個方式 去更新數值

則需要先query之前的數值

之後再用該數值進行運算

最後進行更新

但在併發的情況下 會產生競爭問題

雖然 innodb在用update的時候會上鎖 不會讓人同時更新某個row

但讀的時候並不會上鎖, 若有兩個請求同時近來讀值 假設 val=5 , 兩個請求都是把當前數值加1

A請求讀值後 +1 = 6, 並更新上去 val=6
B請求也是跟A同時進來 因此第一時間也是讀到5, 並+1 又再更新一次 val=6

此時 val 只被加到一次

orm更新操作為
select field from table where id = xxx;   
得到 field數值 = val   
new_val = val + 1   
update table set field = new_val from table where id = xxx.  

一般用sql處理 會直接 update table set field=field+1 where id = xxx; 直接一個SQL語句更新   

要執行這類 field=field+1 查詢前值 計算 並更新的 操作  
就需要用到F

```python
## 不用F 操作, 併發情況會有問題
from bookstore.models import Book
book = Book.objects.get(title="harry potter2") 
new_price = book.price+1 # 讀值 並最後續計算
book.price = new_price
book.save() # 更新
```

```python
## F 操作
from bookstore.models import Book
from django.db.models import F

book = Book.objects.get(title="harry potter2") 
book.price = F("price") + 1 # 此時數值並沒有固定
book.save() # update 時會用當前值去做+1

## 底層實際執行sql

### Postgres 權限概念

postgres 權限控制核心是基於RBAC, 利用角色(role)控制權限  

簡單說就是創建一個role, 設定該role可以做哪些事  

創建user後, 就把該role掛在user上 ,   

postresql中 user與role 實際上是同一種單位 只差別在有無登錄權限

以下無論創建user或是 role, 都是用role的指令

賦予某role權限的user, 所執行的動作等同該role

---
#### 範例情況

- 身為一個user, 若被創建時有createdb權限, 由你所創建的db的owner就是你自己
   對於該db來說, 你就與superuser相同

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

### 基本操作隨筆 

```SQL
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



### RBAC

創建/刪除/刪除/檢索 role,  且可定義基本系統權限 

#### 檢索user  

```SQL
\du
```

```SQL
select * from pg_user;
```
---

#### 檢索role 詳細權限


```SQL
select * from pg_roles
```

---
#### 創建role


```SQL
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
#### 修改role

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
#### 刪除role

只有superuser,createuser 可以刪除

刪除前 需先刪除相依於此role的對象或權限

```SQL
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
#### 賦予role

賦予後, 該user就可以擁有該role的權限

```SQL
grant <role> to <某user>
##　將賦予某user <role>權限
```

```
grant postgrs to test01
# 使user test01 擁有 user postgres權限
```

須注意 被賦於權限後 該user 權限不會馬上生效

該user需執行, 類似linux中 su操作

```SQL
set role <role>:
```


#### 撤銷賦予的role

將user test01的 role postgres 拔掉 
```SQL
revoke postgres from test01;
```




### 物件權限

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

---
#### Database 層

資料庫層級的授權, 是表示可以操作Database以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

- Schema
- Table
- Row

##### Create Database Authorization

create 一般內部資源控制都授權可以執行

```SQL
grant create on database <某db> to <userB>
```
##### Revoke Database Authorization

```SQL
revoke create on database <某db> from <userB>
```


---
#### Schema 層

Schema層級的授權, 是表示可以操作Schema以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

需注意 因無法檢索schema層級, 推薦把預設schema鎖在授權的schema, 這樣檢索內部資源才會有辦法看到


- Table
- Row

##### Create Schema Authorization

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

##### Revoke Schema Authorization

```SQL
revoke create on schema <某db> from <userB>
```


---
#### Table 層

簡單說是可以在某schema下 針對table row操作 給予權限

- row


##### Create Table Authorization

需注意schema檢索權限是必須的 否則無法看到table之類的 也等於無法指定 會沒辦法操作內部資源

```SQL
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

##### Revoke Table Authorization

```SQL
revoke <關鍵字> on <schema_name>.<table_name> from <user_name>
```


---

### Postgres 權限概念

postgres 權限控制核心是基於RBAC, 利用角色(role)控制權限  

簡單說就是創建一個role, 設定該role可以做哪些事  

創建user後, 就把該role掛在user上 ,   

postresql中 user與role 實際上是同一種單位 只差別在有無登錄權限

以下無論創建user或是 role, 都是用role的指令

賦予某role權限的user, 所執行的動作等同該role

---
#### 範例情況

- 身為一個user, 若被創建時有createdb權限, 由你所創建的db的owner就是你自己
   對於該db來說, 你就與superuser相同

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

### 基本操作隨筆 

```SQL
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



### RBAC

創建/刪除/刪除/檢索 role,  且可定義基本系統權限 

#### 檢索user  

```SQL
\du
```

```SQL
select * from pg_user;
```
---

#### 檢索role 詳細權限


```SQL
select * from pg_roles
```

---
#### 創建role


```SQL
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
#### 修改role

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
#### 刪除role

只有superuser,createuser 可以刪除

刪除前 需先刪除相依於此role的對象或權限

```SQL
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
#### 賦予role

賦予後, 該user就可以擁有該role的權限

```SQL
grant <role> to <某user>
##　將賦予某user <role>權限
```

```
grant postgrs to test01
# 使user test01 擁有 user postgres權限
```

須注意 被賦於權限後 該user 權限不會馬上生效

該user需執行, 類似linux中 su操作

```SQL
set role <role>:
```


#### 撤銷賦予的role

將user test01的 role postgres 拔掉 
```SQL
revoke postgres from test01;
```




### 物件權限

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

---
#### Database 層

資料庫層級的授權, 是表示可以操作Database以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

- Schema
- Table
- Row

##### Create Database Authorization

create 一般內部資源控制都授權可以執行

```SQL
grant create on database <某db> to <userB>
```
##### Revoke Database Authorization

```SQL
revoke create on database <某db> from <userB>
```


---
#### Schema 層

Schema層級的授權, 是表示可以操作Schema以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

需注意 因無法檢索schema層級, 推薦把預設schema鎖在授權的schema, 這樣檢索內部資源才會有辦法看到


- Table
- Row

##### Create Schema Authorization

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

##### Revoke Schema Authorization

```SQL
revoke create on schema <某db> from <userB>
```


---
#### Table 層

簡單說是可以在某schema下 針對table row操作 給予權限

- row


##### Create Table Authorization

需注意schema檢索權限是必須的 否則無法看到table之類的 也等於無法指定 會沒辦法操作內部資源

```SQL
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

##### Revoke Table Authorization

```SQL
revoke <關鍵字> on <schema_name>.<table_name> from <user_name>
```


---

### Postgres 權限概念

postgres 權限控制核心是基於RBAC, 利用角色(role)控制權限  

簡單說就是創建一個role, 設定該role可以做哪些事  

創建user後, 就把該role掛在user上 ,   

postresql中 user與role 實際上是同一種單位 只差別在有無登錄權限

以下無論創建user或是 role, 都是用role的指令

賦予某role權限的user, 所執行的動作等同該role

---
#### 範例情況

- 身為一個user, 若被創建時有createdb權限, 由你所創建的db的owner就是你自己
   對於該db來說, 你就與superuser相同

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

### 基本操作隨筆 

```SQL
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



### RBAC

創建/刪除/刪除/檢索 role,  且可定義基本系統權限 

#### 檢索user  

```SQL
\du
```

```SQL
select * from pg_user;
```
---

#### 檢索role 詳細權限


```SQL
select * from pg_roles
```

---
#### 創建role


```SQL
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
#### 修改role

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
#### 刪除role

只有superuser,createuser 可以刪除

刪除前 需先刪除相依於此role的對象或權限

```SQL
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
#### 賦予role

賦予後, 該user就可以擁有該role的權限

```SQL
grant <role> to <某user>
##　將賦予某user <role>權限
```

```
grant postgrs to test01
# 使user test01 擁有 user postgres權限
```

須注意 被賦於權限後 該user 權限不會馬上生效

該user需執行, 類似linux中 su操作

```SQL
set role <role>:
```


#### 撤銷賦予的role

將user test01的 role postgres 拔掉 
```SQL
revoke postgres from test01;
```




### 物件權限

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

---
#### Database 層

資料庫層級的授權, 是表示可以操作Database以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

- Schema
- Table
- Row

##### Create Database Authorization

create 一般內部資源控制都授權可以執行

```SQL
grant create on database <某db> to <userB>
```
##### Revoke Database Authorization

```SQL
revoke create on database <某db> from <userB>
```


---
#### Schema 層

Schema層級的授權, 是表示可以操作Schema以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

需注意 因無法檢索schema層級, 推薦把預設schema鎖在授權的schema, 這樣檢索內部資源才會有辦法看到


- Table
- Row

##### Create Schema Authorization

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

##### Revoke Schema Authorization

```SQL
revoke create on schema <某db> from <userB>
```


---
#### Table 層

簡單說是可以在某schema下 針對table row操作 給予權限

- row


##### Create Table Authorization

需注意schema檢索權限是必須的 否則無法看到table之類的 也等於無法指定 會沒辦法操作內部資源

```SQL
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

##### Revoke Table Authorization

```SQL
revoke <關鍵字> on <schema_name>.<table_name> from <user_name>
```


---

### Postgres 權限概念

postgres 權限控制核心是基於RBAC, 利用角色(role)控制權限  

簡單說就是創建一個role, 設定該role可以做哪些事  

創建user後, 就把該role掛在user上 ,   

postresql中 user與role 實際上是同一種單位 只差別在有無登錄權限

以下無論創建user或是 role, 都是用role的指令

賦予某role權限的user, 所執行的動作等同該role

---
#### 範例情況

- 身為一個user, 若被創建時有createdb權限, 由你所創建的db的owner就是你自己
   對於該db來說, 你就與superuser相同

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

### 基本操作隨筆 

```SQL
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



### RBAC

創建/刪除/刪除/檢索 role,  且可定義基本系統權限 

#### 檢索user  

```SQL
\du
```

```SQL
select * from pg_user;
```
---

#### 檢索role 詳細權限


```SQL
select * from pg_roles
```

---
#### 創建role


```SQL
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
#### 修改role

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
#### 刪除role

只有superuser,createuser 可以刪除

刪除前 需先刪除相依於此role的對象或權限

```SQL
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
#### 賦予role

賦予後, 該user就可以擁有該role的權限

```SQL
grant <role> to <某user>
##　將賦予某user <role>權限
```

```
grant postgrs to test01
# 使user test01 擁有 user postgres權限
```

須注意 被賦於權限後 該user 權限不會馬上生效

該user需執行, 類似linux中 su操作

```SQL
set role <role>:
```


#### 撤銷賦予的role

將user test01的 role postgres 拔掉 
```SQL
revoke postgres from test01;
```




### 物件權限

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

---
#### Database 層

資料庫層級的授權, 是表示可以操作Database以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

- Schema
- Table
- Row

##### Create Database Authorization

create 一般內部資源控制都授權可以執行

```SQL
grant create on database <某db> to <userB>
```
##### Revoke Database Authorization

```SQL
revoke create on database <某db> from <userB>
```


---
#### Schema 層

Schema層級的授權, 是表示可以操作Schema以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

需注意 因無法檢索schema層級, 推薦把預設schema鎖在授權的schema, 這樣檢索內部資源才會有辦法看到


- Table
- Row

##### Create Schema Authorization

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

##### Revoke Schema Authorization

```SQL
revoke create on schema <某db> from <userB>
```


---
#### Table 層

簡單說是可以在某schema下 針對table row操作 給予權限

- row


##### Create Table Authorization

需注意schema檢索權限是必須的 否則無法看到table之類的 也等於無法指定 會沒辦法操作內部資源

```SQL
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

##### Revoke Table Authorization

```SQL
revoke <關鍵字> on <schema_name>.<table_name> from <user_name>
```


---

### Postgres 權限概念

postgres 權限控制核心是基於RBAC, 利用角色(role)控制權限  

簡單說就是創建一個role, 設定該role可以做哪些事  

創建user後, 就把該role掛在user上 ,   

postresql中 user與role 實際上是同一種單位 只差別在有無登錄權限

以下無論創建user或是 role, 都是用role的指令

賦予某role權限的user, 所執行的動作等同該role

---
#### 範例情況

- 身為一個user, 若被創建時有createdb權限, 由你所創建的db的owner就是你自己
   對於該db來說, 你就與superuser相同

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

### 基本操作隨筆 

```SQL
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



### RBAC

創建/刪除/刪除/檢索 role,  且可定義基本系統權限 

#### 檢索user  

```SQL
\du
```

```SQL
select * from pg_user;
```
---

#### 檢索role 詳細權限


```SQL
select * from pg_roles
```

---
#### 創建role


```SQL
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
#### 修改role

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
#### 刪除role

只有superuser,createuser 可以刪除

刪除前 需先刪除相依於此role的對象或權限

```SQL
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
#### 賦予role

賦予後, 該user就可以擁有該role的權限

```SQL
grant <role> to <某user>
##　將賦予某user <role>權限
```

```
grant postgrs to test01
# 使user test01 擁有 user postgres權限
```

須注意 被賦於權限後 該user 權限不會馬上生效

該user需執行, 類似linux中 su操作

```SQL
set role <role>:
```


#### 撤銷賦予的role

將user test01的 role postgres 拔掉 
```SQL
revoke postgres from test01;
```




### 物件權限

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

---
#### Database 層

資料庫層級的授權, 是表示可以操作Database以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

- Schema
- Table
- Row

##### Create Database Authorization

create 一般內部資源控制都授權可以執行

```SQL
grant create on database <某db> to <userB>
```
##### Revoke Database Authorization

```SQL
revoke create on database <某db> from <userB>
```


---
#### Schema 層

Schema層級的授權, 是表示可以操作Schema以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

需注意 因無法檢索schema層級, 推薦把預設schema鎖在授權的schema, 這樣檢索內部資源才會有辦法看到


- Table
- Row

##### Create Schema Authorization

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

##### Revoke Schema Authorization

```SQL
revoke create on schema <某db> from <userB>
```


---
#### Table 層

簡單說是可以在某schema下 針對table row操作 給予權限

- row


##### Create Table Authorization

需注意schema檢索權限是必須的 否則無法看到table之類的 也等於無法指定 會沒辦法操作內部資源

```SQL
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

##### Revoke Table Authorization

```SQL
revoke <關鍵字> on <schema_name>.<table_name> from <user_name>
```


---
# (UPDATE "book")


### Postgres 權限概念

postgres 權限控制核心是基於RBAC, 利用角色(role)控制權限  

簡單說就是創建一個role, 設定該role可以做哪些事  

創建user後, 就把該role掛在user上 ,   

postresql中 user與role 實際上是同一種單位 只差別在有無登錄權限

以下無論創建user或是 role, 都是用role的指令

賦予某role權限的user, 所執行的動作等同該role

---
#### 範例情況

- 身為一個user, 若被創建時有createdb權限, 由你所創建的db的owner就是你自己
   對於該db來說, 你就與superuser相同

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

### 基本操作隨筆 

```SQL
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



### RBAC

創建/刪除/刪除/檢索 role,  且可定義基本系統權限 

#### 檢索user  

```SQL
\du
```

```SQL
select * from pg_user;
```
---

#### 檢索role 詳細權限


```SQL
select * from pg_roles
```

---
#### 創建role


```SQL
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
#### 修改role

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
#### 刪除role

只有superuser,createuser 可以刪除

刪除前 需先刪除相依於此role的對象或權限

```SQL
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
#### 賦予role

賦予後, 該user就可以擁有該role的權限

```SQL
grant <role> to <某user>
##　將賦予某user <role>權限
```

```
grant postgrs to test01
# 使user test01 擁有 user postgres權限
```

須注意 被賦於權限後 該user 權限不會馬上生效

該user需執行, 類似linux中 su操作

```SQL
set role <role>:
```


#### 撤銷賦予的role

將user test01的 role postgres 拔掉 
```SQL
revoke postgres from test01;
```




### 物件權限

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

---
#### Database 層

資料庫層級的授權, 是表示可以操作Database以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

- Schema
- Table
- Row

##### Create Database Authorization

create 一般內部資源控制都授權可以執行

```SQL
grant create on database <某db> to <userB>
```
##### Revoke Database Authorization

```SQL
revoke create on database <某db> from <userB>
```


---
#### Schema 層

Schema層級的授權, 是表示可以操作Schema以內自己的所有單位 (僅能控制owner是自己的單位,或是新建)

需注意 因無法檢索schema層級, 推薦把預設schema鎖在授權的schema, 這樣檢索內部資源才會有辦法看到


- Table
- Row

##### Create Schema Authorization

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

##### Revoke Schema Authorization

```SQL
revoke create on schema <某db> from <userB>
```


---
#### Table 層

簡單說是可以在某schema下 針對table row操作 給予權限

- row


##### Create Table Authorization

需注意schema檢索權限是必須的 否則無法看到table之類的 也等於無法指定 會沒辦法操作內部資源

```SQL
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

##### Revoke Table Authorization

```SQL
revoke <關鍵字> on <schema_name>.<table_name> from <user_name>
```


---
# (   SET "title" = 'harry potter2',)

# (       "price" = ("book"."price" + 1),)

# (       "publish" = 'jump',)

# (       "market_price" = 80.00,)

# (       "author_id" = 2)

# ( WHERE "book"."id" = 4)

# (Execution time: 0.002023s [Database: default])
```
---
### Q

django filter 條件查詢 多條件 只能用 and 做連接

Q可以處理 or 與 not

支援運算符

1. & : 與
2. | : 或
3. ~ : not


```python
from django.db.models import Q
from bookstore.models import Book
books = Book.objects.filter( Q(price__gt=100) | Q(publish="康宣")) 
# 查找 price > 100 或 publush = "康宣" 

```


```python
from django.db.models import Q
from bookstore.models import Book

books = Book.objects.filter( Q(price__gt=100) & ~ Q(publish="康宣")) 
# 查找 price > 100 且 非 publush = "康宣" 

```
---

### 聚合查詢

#### 一般聚合查詢

聚合函數 Sum,Avg,Count,Max,Min  

等同sql中 select Sum(field1) as 自定義別名 from table 用法  

就是將查詢出來的結果 做一些統計 e.g. 計算總和/平均/最大值/最小值/ 輸出值總數目  

輸出方式為字典形式 { 自定義別名 : 統計數值 }

```python
from django.db.models import Count
from bookstore.models import Book

res = Book.objects.aggregate( res = Count('title'))

## 實際執行
# (SELECT COUNT("book"."title") AS "res")

# (  FROM "book")

# (Execution time: 0.001583s [Database: default])

```

輸出
```python

{'res': 1} # 因該table只剩 一個row , count出來結果為 1

```


#### 分組聚合查詢

其實就是group by

用法跟上面相同 只是會用某個field的數值做分類  
e.g. 假設field name為組別, 總共有 A B C,  
就會將field數值為為A的所有結果做統計 B的所有結果做統計  C的所有結果做統計 
最後輸出格式為為
```python
List[ {group_by_field_name: field_val1, 聚合輸出別名: 聚合統計結果},{group_by_field_name: field_val2, 聚合輸出別名: 聚合統計結果} ... ]
```

```python
## 先檢視所有

from django.db.models import Count
from bookstore.models import Book

books = Book.objects.all().select_related("author") 
##該表有參考 foreignkey, select_related 避免n+1 query (底層sql會用join的)

for book in books:
  print(book)
```
輸出
```python
title:harry potter2 price:123.00 publish:jump market_price:80.00 author:author_id: 2 name: foo2 age: 20 email: 123@google.com
title:harry potter price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
title:harry potter3 price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
```

用author group by 再 count數目
```python
from django.db.models import Count
from bookstore.models import Book

# foo與foo2的類群 各有幾個 , 因為count是統計row數目 price是隨便挑的 沒特別意義 只要能統計row數目 用哪個欄位都可以
books = Book.objects.values('author') #同 select price from table
q = books.annotate(res= Count('price')) 
## group by author 

print(q)

## 實際執行
# (SELECT "book"."author_id",)

# (       COUNT("book"."price") AS "res")

# (  FROM "book")

# ( GROUP BY "book"."author_id")

# ( LIMIT 21)

# (Execution time: 0.000439s [Database: default])
```
輸出
```django

<QuerySet [{'author': 1, 'res': 2}, {'author': 2, 'res': 1}]>
# author_id 為1的 count結果為2,  author_id 為2的 count結果為1, 

```

---

### 執行 raw SQL

用django封裝raw sql 語法, 這邊用法需嚴謹 避免被SQL injection

需注意 返回的是 RawQuerySet 非 QuerySet, 類似 也可用for迭代
```python
from bookstore.models import Book

books = Book.objects.raw("select * from book")
## 等同 Books.objects.all() 效果

print(books)
```

輸出

```python
<RawQuerySet: select * from book>
```

raw sql有變數的 , 不能用python的語法去組合 e.g f-string or % format  
會有被injection風險 需用django的方式做組合

正確方式
```python
from bookstore.models import Book

books = Book.objects.raw("select * from book where publish = %s", ['jump',])

print(books)

```
輸出

```python
<RawQuerySet: select * from book where publish = jump>
```

#### injection 測試

錯誤舉例 沒有符合jump2 的選項 依然可以查找出來他的
```python
from bookstore.models import Book
books = Book.objects.raw("select * from book where publish = %s" % ("'jump2' or 1=1"))
for book in books:
  print(book)
```

輸出

```python
title:harry potter2 price:123.00 publish:jump market_price:80.00 author:author_id: 2 name: foo2 age: 20 email: 123@google.com
title:harry potter price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
title:harry potter3 price:100.00 publish:jump market_price:80.00 author:author_id: 1 name: foo age: 20 email: 123@google.com
```


django方式 injection測試

```python
from bookstore.models import Book
books = Book.objects.raw("select * from book where publish = %s",["'jump2' or 1=1"])
for book in books:
  print(book)
```
輸出

```python
# 輸出為空 , 無法查找出結果
```
---

## 後台系統

創建管理者帳戶  

```bash
python3 manage.py createsuperuser
```
app_name_/admin.py  

```python
## 將model註冊至後台
## 因 djang model <> db table , 簡單說就是將db的某個table 註冊到後台 可以透過後台ui CRUD

admin.site.register(Book)
```

可自定義model顯示, 如有5個field 只想顯示4個, 哪些field可以被修改  

app_name_/admin.py  

```python

## 自訂自 model: Book 在後台的功能
class BookManager(admin.ModelAdmin):
    # Book這個model 在後台顯示的 field有哪些
    list_display = ["id", "title", "publish", "price"]
    # 控制後台顯示field 哪些為 link url, 可轉跳至修改頁面, 被需在list_display中 否則會看不到
    list_display_links = ["price"]
    # 在後台 哪個field數值為為link url 可以轉跳至修改頁面, 但點進去 該row所有field都數值都可以修改, 若沒有設定 則都不可自由編輯該row所有field數值
    # 若不設定 但想限制修改哪些欄位 可以用 list_editable 控制特定field數值 才可以被編輯

    list_filter = ["publish"]
    # 有邊跑出一個控制器 可以類似group by 的功能去filter 特定 field,  有 All , value1 , value2 e.g.假設性別  就會有 All,男,女 若在該控制器選男 就只會顯示男的 row,

    search_fields = ["title","publish"]
    # 模糊查詢選項 可設定哪些field會被批配關鍵字, 可以用模糊查詢這些field數值含有該關鍵字的row
    # e.g. 後台UI search欄位檢索  'harry', 就會顯示出 所有title,publish含有harry的row

    list_editable = ["title"]
    # 設定row 的哪些特定field可被編輯, 不可與list_display_links 同時存在 , 以功能來說有衝突 有設定 list_display_links , 設定這邊就沒意義 因為links 可以修改所有field數值

    # ... 其他可至django文檔 admin 查看

```

後台顯示名稱 有些在Model.py Meta設定  

```python

class Meta:
    db_table =  "book" # 在實際資料庫中的table 名稱
    verbose_name = "書" # 後台管理頁面中 book 顯示名稱

```

---


## cookie

cookie 設置 讀取 刪除

views.py
```python

def set_cookie(request: HttpRequest)-> HttpResponse:
    resp = HttpResponse("set cookie")
    resp.set_cookie("my_cookie", "123", 500)
    return resp
                                                             
                                                             
def get_cookie(request: HttpRequest) -> HttpResponse:    
    val = request.COOKIES.get("my_cookie", "No Cookie Value")```
                                                             
    return HttpResponse(f"cookie value is {val}")
                                                             
                                                             
def delete_cookie(request: HttpRequest) -> HttpResponse:
    resp = HttpResponse("delete cookie")
    resp.delete_cookie("my_cookie")
                                                             
    return resp
```
參考測試url:
http://localhost:8000/bookstore/set_cookie/
http://localhost:8000/bookstore/get_cookie/
http://localhost:8000/bookstore/delete_cookie/

---

## Session

這邊利用 middleware 實現 sessionID  
需先確認middleware有開啟  
main_app_/settings.py   
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',  ## <<<<<<<<<<<<<<<<<<
    'django.contrib.messages',
    'django.contrib.staticfiles',
    "app1",
    "app2",
    "bookstore",
    'django_extensions',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware', ## <<<<<<<<<<<<<<<<<
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

session 新增 讀取 刪除  
實際上django中 session被包裝類似dict 用法基本上相同  
views.py  
```python
def set_session(request: HttpRequest) -> HttpResponse:
    request.session["my_session"] = "456"
    return HttpResponse("set session")


def get_session(request: HttpRequest) -> HttpResponse:
    session_value = request.session.get("my_session", "No session")
    resp = HttpResponse(f"session value is {session_value}")
    return resp


def delete_session(request: HttpRequest) -> HttpResponse:
    resp = HttpResponse("delete session")
    del request.session["my_session"]
    return resp
```

參考測試url:
http://localhost:8000/bookstore/set_session/
http://localhost:8000/bookstore/get_session/
http://localhost:8000/bookstore/delete_session/

---
### 基本屬性設置

main_app_/settings.py

- SESSION_COOKIE_AGE: SESSION COOKIE 預設壽命 , 預設 2 week
- SESSION_EXPIRE_AT_BROWSER_CLOSE: 關閉瀏覽器 cookie and session 是否失效

```python
SESSION_COOKIE_AGE = 60 * 60 * 24 * 7 * 2
SESSION_EXPIRE_AT_BROWSER_CLOSE: True # 預設False
```

預設為session_store為當前資料庫, 使用session前要先migrate 產生session table

---
### 修改SessionStore

假設修改至redis

```bash
pip install django-redis
```

main_app_/settings.py

```python

SESSION_ENGINE = "django.contrib.sessions.backends.cache"

SESSION_CACHE_ALIAS = "default"

CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",  # 适应你的 Redis 配置
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}

```

設置完需先重新映射資料庫
```bash

python manage.py makemigrations
python manage.py migrate

```

清除session store 紀錄

```bash
python manage.py clearsessions
```





