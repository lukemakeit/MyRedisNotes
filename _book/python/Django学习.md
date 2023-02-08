### Django学习

创建新项目: `django-admin startproject myDjangoDemo`;

启动服务: `python manage.py runserver 端口号`,不传端口，默认是8000;

**`manage.py`包含项目管理的子命令**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220624064711555.png" alt="image-20220624064711555" style="zoom:50%;" />

**`__init__.py`**: Python包的初始化文件;

**`wsgi.py`**:WEB服务网关的配置文件——Django正式启动时 需要用到;

**`urls.py`**:项目的主路由配置——HTTP请求进入Django时，优先调用该文件;

**`settings.py`**:项目的配置文件, 包含项目启动时需要的配置;

#### settings.py

- 公有配置: Django 官方提供的配置项
- 自定义配置
- 格式，大写名字: `BASE_DIR = xxxx`
- `DEBUG=True`:
  - 检测到代码改动后,立刻重启服务
  - 更好的报错页面;
- `ALLOWED_HOSTS = [127.0.0.1]`:允许的请求头HOST。为空的时候，只允许127.0.0.1 和 localhost; 还可以配置 `*`代表任何请求头的host都能访问项目;
- `ROOT_URLCONF = 'myDjangoDemo.urls'`: 主路由文件配置;
- `DATABASES=`:数据库文件的保存地址;
- 配置引用方式: `from django.conf import settings`

#### 视图函数: views.py

语法:

```python
def xxx_view(request[,其他参数]):
    return HttpResponse对象
```

#### 路由配置: urls.py

示例:

```python
# 文件 urls.py
from . import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('hello',views.HelloWolrd),
]

# 文件 views.py
from django.http import HttpResponse

def HelloWolrd(request):
    html="<h1>hello world</h1>"
    return HttpResponse(html)
```

<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20220626223326911.png" alt="image-20220626223326911" style="zoom:50%;" />

例如:

```python
# 文件 urls.py
from . import views

urlpatterns = [
    path('page/<int:pgn>',views.pagen_view)
    path('<int:n>/<str:op>/<int:m>',views.cal_view)
]

# 文件 views.py
from django.http import HttpResponse

def pagen_view(request,pgn):
    html="<h2>设置编号为{}的页面</h2>".format(pgn)
    return HttpResponse(html)

def cal_view(request,n,op,m):
    if op not in ['add','sub','mul']:
        return HttpResponse("Your op:{} is valid".format(op))
    if op == 'add':
      return HttpResponse("{}+{}={}".format(n,m,n+m))
    elif op == 'sub':
      return HttpResponse("{}-{}={}".format(n,m,n-m))
    elif op == 'mul':
      return HttpResponse("{}*{}={}".format(n,m,n*m))
```

**`re_path()`:路径中用正则匹配**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220627063238050.png" alt="image-20220627063238050" style="zoom:50%;" />

````python
#操作的数字只允许两位
urlpatterns = [
    re_path(r'^(?P<x>\d{1,2}/(?P<op>\w+)/(?P<y>\d{1,2})$',views.cal2_view)
]

def cal2_view(request,n,op,m):
    if op not in ['add','sub','mul']:
        return HttpResponse("Your op:{} is valid".format(op))
    if op == 'add':
      return HttpResponse("{}+{}={}".format(n,m,n+m))
    elif op == 'sub':
      return HttpResponse("{}-{}={}".format(n,m,n-m))
    elif op == 'mul':
      return HttpResponse("{}*{}={}".format(n,m,n*m))
````

#### 请求和响应

**requests**

- `path_info`: URL 字符串;

- `method`: 字符串,表示HTTP请求方法，常用值: `GET`、`POST`

- `GET`: QueryDict查询字典中的对象，包含get请求方式的所有数据;

- `POST`: QueryDict查询字典的对象，包含post请求方式的所有数据;

- `FILES`:类似于字典的兑现，包含所有的上传文件信息

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220627064352972.png" alt="image-20220627064352972" style="zoom:50%;" />

```python
urlpatterns = [
    path('test_request',views.test_request)
]

def test_request(request):
    # 打印在终端
    print("path info is ",request.path_info)
    print("method is ",request.method)
    print("querystring is ",request.GET)

    return HttpResponse("test request ok")

http://127.0.0.1:8000/test_request?a=1&b=2
path info is  /test_request
method is  GET
querystring is  <QueryDict: {'a': ['1'], 'b': ['2']}>
```

**响应对象**

<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20220627065248929.png" alt="image-20220627065248929" style="zoom:25%;" />

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220627065336505.png" alt="image-20220627065336505" style="zoom:25%;" />

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220627065534511.png" alt="image-20220627065534511" style="zoom:25%;" />

例如:

````python
from django.http import HttpResponse, HttpResponseRedirect

return HttpResponseRedirect("/page/1")
````

#### GET 和 POST请求

**无论是GET还是POST,统一由视图函数接收请求，通过判断request.method区分具体的请求动作。**
样例:

```
if request.method == 'GET':
    处理GET请求时的业务逻辑
elif request.method == 'POST':
    处理POST请求时的夜晚逻辑
else:
    其他请求业务逻辑
```

```python
urlpatterns = [
    path('test_get_post',views.test_get_post)
]

def test_get_post(request):
    if request.method == 'GET':
        print(request.GET)
        print(request.GET['a'])
        print(request.GET.getlist('a'))
        print(request.GET.get('c','default c'))
    elif request.method == 'POST':
        x = request.POST['x']
        y = request.POST['y']
        op = request.POST['op']
        pass
    else:
        pass
    return HttpResponse("---test get post method ok")

http://127.0.0.1:8000/test_get_post?a=1&a=2
标准输出:
<QueryDict: {'a': ['1', '2']}>
2
['1', '2']
default c
```

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220627072030943.png" alt="image-20220627072030943" style="zoom:33%;" />

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220627072242781.png" alt="image-20220627072242781" style="zoom: 33%;" />

#### 设计模式及模版

**MVC**

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220627072515932.png" alt="image-20220627072515932" style="zoom: 33%;" />

**MTV**

<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20220627072649665.png" alt="image-20220627072649665" style="zoom:33%;" />

#### 模版配置

<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20220628061901247.png" alt="image-20220628061901247" style="zoom:33%;" />

创建templates

```python
tree
.
├── db.sqlite3
├── manage.py
├── myDjangoDemo
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── __init__.cpython-36.pyc
│   │   ├── settings.cpython-36.pyc
│   │   ├── urls.cpython-36.pyc
│   │   ├── views.cpython-36.pyc
│   │   └── wsgi.cpython-36.pyc
│   ├── settings.py
│   ├── urls.py
│   ├── views.py
│   └── wsgi.py
└── templates
```

修改`myDjangoDemo/settings.py`文件:

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR,"templates")], # 主要是这一行
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

**模版的加载方式**

**方案一: 通过loader获取模版，通过HttpResponse进行响应**

```python
from django.template import loader
# 1. 通过loader加载模版
t = loader.get_template("模版文件名,如 a.html")
# 2. 将t转换成HTML字符串
html = t.render(字典数据)
# 3. 用响应对象将转换的字符串内容返回给浏览器
return  HttpResponse(html)
```

**方案二: 使用`render()`直接加载并响应模版(推荐)**

视图函数中:

```python
from django.shortcuts import render
return render(request,"模版文件名",字典数据)
```

**视图层和模版层之间的交互**

1. 视图函数中可以将python变量封装到**字典(必须是字典)**中传递给模版

   ```python
   def xxx_view(request):
       dict = {
           "变量1":"值1",
           "变量2":"值2"
       }
       return render(request,'xxx.html',dict)
   ```

2. 模版中，我们可以用`{{变量名}}`的语法调用视图传进来的变量

   ```python
   def test_a(request):
       dict ={"username":"zhangsan","age":11}
       return render(request,"a.html",dict)
     
   <ul>
       <li>名字:{{username}}</li>
       <li>年龄:{{age}}</li>
   </ul>
   ```

3. 模版中使用变量的语法

   - `{{变量名}}`
   - `{{变量名.index}}`
   - `{{变量名.key}}`
   - `{{对象方法}}`
   - `{{函数名}}`

   示例:

   ```python
   def say_hi():
     return 'hahaha'
   
   class Dog:
     def say(self):
       return 'wangwang'
     
   dict = {}
   dict['int']=88
   dict['str']="guoxiaomao"
   dict['list']=['Tom','Jack','Lily']
   dict['dict']={'a':9,'b':8}
   dict['func']=say_hi #注意不是 say_hi()
   dict['class_obj']=Dog()
   return render(request,"param.html",dict)
   
   <h3> int 是 {{int}} </h3>
   <h3> str 是 {{str}} </h3>
   <h3> list 是 {{list}} </h3>
   <h3> list[0] 是 {{list.0}} </h3>
   <h3> dict 是 {{dict}} </h3>
   <h3> dict['a'] 是 {{dict.a}} </h3>
   <h3> function 是 {{func}} </h3>
   <h3> class_obj 是 {{class_obj.say}} </h3>
   ```

4. 模版标签

   作用: 将一些服务器端的功能嵌入到模版中，例如流程控制等。标签语法:
   ```python
   {% 标签 %}
   ...
   {% 结束标签 %}
   ```

   **if标签**

   语法:

   ```python
   {% if 条件表达式1 %}
   ...
   {% elif 条件表达式2 %}
   ...
   {% elif 条件表达式3 %}
   ...
   {% else %}
   ...
   {% endif %}
   ```

   if条件表达式中可以用`==`、`!=`、`<`、`>`、`<=`、`>=`、`in`、`not in`、`is`、`is not`、`and`、`or`等

   示例:

   ```html
   dict={"x":10}
   
   {% if x > 10 %}
   今天天气很好
   {% else %}
   今天天气非常好
   {% endif %}
   
   <option value="add"> {% if op == 'add' %} selected {% endif %} +加 </option>
   ```

   **for标签**

   语法:
   ```python
   {% for 变量 in 可迭代的对象 %}
       ...循环语句
   {% empty %}
       ...可迭代对象无数据时填充的语句
   {% endfor %}
   ```

   `empty`: 意思是在 可迭代对象无数据时，填充这条语句;

   <img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20220628072145658.png" alt="image-20220628072145658" style="zoom:33%;" />

示例:
```python
def test_if_for(request):
    dict={}
    dict["lst"]=["Tom","Jack","Lily"]
    return render(request,"a.html",dict)

    <ul>
        {% for name in lst %}
        {% if forloop.first %} start loop {% endif %}
        <li>{{forloop.counter}} {{name}}</li>
        {% if forloop.last %} end loop {% endif %}
        {% empty %}
            当前没数据
        {% endfor %}
    </ul>
    
结果:
  start loop
1 Tom
2 Jack
3 Lily
end loop
```

### django 应用和分布式路由

**应用是在django中一个完全独立的业务模块，可以包含自己的路由、视图、模版和模型**
**创建应用**

```shell
python3 manage.py startapp ${应用名}
```

**在setting.py的`INSTALLED_APPS`列表中配置安装此应用**

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    '...',
    '${自己的应用名}'
]
```

**分布式路由**

主路由配置文件可以做请求分发(分布式请求处理)。
<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20220716084912073.png" alt="image-20220716084912073" style="zoom:33%;" />

**配置分布式路由**
主路由配置:

```python
文件 urls.py
from django.urls import path,include
from . import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('test_a',views.test_a),
    ...
    path('music/',include('music.urls')), #注意是 music/
]
```

music应用的路由配置`urls.py`:
```python
文件 music/urls.py
from django.urls import  path
from . import views

urlpatterns =[
    path('index', views.index_view),
]

文件 music/views.py
from django.shortcuts import render
from django.http import HttpResponse
# Create your views here.

def index_view(request):
    html="<h2>欢迎进入music的主页</h2>"
    return HttpResponse(html)
```

**应用下的模版**

1. 在应用下手动创建`templates`文件夹;
2. `setting.py`中开启应用模版功能:
   - `TEMPLATE`配置项中的`APP_DIRS`值为`TRUE`即可;
3. 应用下`templates`和 外层`templates`都存在时, django得查找模版规则
   - **优先查找外层的`templates`目录下的模版**;
   - **按照`INSTALLED_APPS`配置下的应用程序逐层查找**;

```python
//文件: music/templates/index.html
<head>
    <meta charset="UTF-8">
    <title>music频道</title>
</head>
<body>
    <h2>我是music</h2>
</body>

//文件: music/views.py
from django.shortcuts import render
from django.http import HttpResponse
# Create your views here.

def index_view(request):
    return render(request,'index.html')
```

如果外层有同名`index.html`，会优先使用外层的`index.html`。应用之间有同名的`.html`文件也不行，搞不好用到别人的`.html`文件了。如何解决?

```python
搞成这样  music/templates/music/index.html

//文件: music/views.py
from django.shortcuts import render
from django.http import HttpResponse
# Create your views here.
def index_view(request):
    return render(request,'music/index.html')
```

### django ORM(Object Relational Mapping)

**环境**

- 先确认ubuntu是否安装了`python3-dev`和`default-libmysqlclient-dev`;

  ```shell
  sudo apt list --installed|grep -E "libmysqlclient-dev|python3-dev"
  ```

- 安装命令: 

  ```python
  sudo pip3 install mysqlclient
  pip freeze |grep -i "mysql"
  mysqlclient==2.1.1
  ```

- 创建数据库: `create database mysite3  default charset utf8mb4`

- 配置数据库`setting.py`:

  ```
  DATABASES = {
      'default': {
          'ENGINE': 'django.db.backends.mysql',
          'NAME': 'mysite3',
          'USER':'xx',
          'PASSWORD':'xxx',
          'HOST':'127.0.0.1',
          'PORT':'20002'
      }
  }
  ```

**模型**

- 模型是一个python类，他是由`django.db.models.Model`派生出的子类;
- 一个模型类代表数据库中的一张数据表
- 模型类中的每一个类属性代表数据库中的一个字段
- 模型是数据交互的接口，是表示和操作数据库的方法和方式;

代码:
```python
from django.db import models
# Create your models here.

class Music(models.Model):
    title = models.CharField('专辑名',max_length=50,null=False,default='')
    price = models.DecimalField('价格',max_digits=7,decimal_places=2)
```

**数据库迁移: 同步你对模型所做更改(添加字段,删除模型等)到数据库模式**

- 生成迁移文件:`python manage.py makemigrations`, 将应用下的`models.py`文件中生成一个中间文件，并保存在`migrations`文件夹中;

- 执行迁移脚本程序`python manage.py migrate`。此时表结构已同步创建到相关库中;

  ```sql
  show create table music_music\G
  *************************** 1. row ***************************
         Table: music_music
  Create Table: CREATE TABLE `music_music` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `title` varchar(50) NOT NULL,
    `price` decimal(7,2) NOT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  ```

**模型类定义**

```python
from django.db import models
class 模型类名(models.Model):
    字段名 = models.字段类型(字段选项)
```

<img src="/Users/lukexwang/Library/Application Support/typora-user-images/image-20220716111158465.png" alt="image-20220716111158465" style="zoom:50%;" />

**字段类型**

- `BooleanField()`
  - 数据库类型:`tinyint(1)`
  - 编程语言中: 使用True or False表示值;
  - 数据库中: 使用1 or 0 表示值;
- `CharField()`
  - 数据库类型: varchar
  - 注意: 必须要指定`max_length`参数的值;
- `DateField()`
  - 数据类型: date，表示日期
  - 参数:
    - `auto_now` 每次保存对象时，自动设置改字段为当前时间(取值:True/False);
    - `auto_now_add`: 当对象第一次被创建时自动设置当前时间(取值:True/False);
    - `default`: 设置当前时间(取值:字符串格式时间如`2019-6-1`);
    - 以上三个参数只能多选一;
- `DateTimeField()`
  - 数据库类型:`datetime(6)`，表示日期和时间;
  - 参数同`DateField`
- `FloatField()`
  - 数据库类型: double
- `DecimalField()`
  - 数据库类型:`decimal(x,y)`
  - Python语言中：使用小数表示该列的值;
  - 数据库语言中: 使用小数
  - 参数:
    - `max_digits`: 位数总数，包括小数点后的位数。该值必须大于等于`decimal_places`;
    - `decimal_places`:小数点后的数字数量
- `EmailFiled()`
  - 数据库类型: varchar
- `IntegerField()`
  - 数据库类型: int
- `ImageField()`
  - 数据库类型: varchar(100)
  - 作用: 在数据库中保存图片的路径
- `TextField()`
  - 数据库类型:`longtext`

**字段选项**

- `primary_key`: 如果设置喂True, 表示该列为主键；如果指定一个字段为主键，则数据表不会自动创建id字段;
- `blank`:设置为True,字段可以为空。设置为False，字段必须填写的。并不是数据库的`null`，而是在admin 后台编辑数据时，必须填写内容;
- `null`:设置为True, 表示MySQL的null; 默认为False，建议加入default 选项来设置默认值;
- `default`:设置列的默认值，和`null=False`一起使用;
- `db_index`:设置为True，表示为该列增加索引;
- `unique`:设置为True, 表示字段在数据库中值必须唯一;
- `db_column`: 指定列的名称，如果不指定的话则采用属性名作为列名;
- `verbose_name`:设置此字段在admin界面上显示的名称;

```python
# 创建一个属性, 表示用户名称，长度为30字符，必须是唯一的，不能为空，添加索引
name = models.CharField(max_length=30,unique=True,null=False,db_index=True)
```

**Meta类:和表相关的属性**

```python
class Music(models.Model):
    title = models.CharField('专辑名',max_length=50,default='')
    price = models.DecimalField('价格',max_digits=7,decimal_places=2)
    class Meta:
        db_table = "zhuanji" # 可改变当前模型类对应的表名
```

再次执行: `python manage.py makemigrations`、`python manage.py migrate`

#### 创建数据

每个继承自`models.Model`的模型类,都有一个objects对象被同样继承下来。这个对象称为管理对象。
数据库的增删查改可以通过模型的管理器实现:

```python
class MyModel(models.Model):
    ...
MyModel.objects.create(...) # objects是管理对象
```

**方案一:MyModel.objects.create(属性1=值1,属性2=值2...)**

成功则返回创建成功的对象; 失败则抛出异常。

**方案二:创建Model实例对象,并调用save()进行保存**

```python
obj = MyModel(属性1=值1,属性2=值2)
obj.属性3=值3
obj.save()
```

**Django Shell:交互是应用项目工程代码执行相关操作:`python manage.py shell`**

用于命令调试，数据查询等等。
```python
>>> from music.models import Music
>>> m01=Music.objects.create(title='周杰伦',price=200,info='2022最新专辑')

MySQL [mysite3]> select * from zhuanji;
+----+-----------+--------+------------------+
| id | title     | price  | info             |
+----+-----------+--------+------------------+
|  1 | 周杰伦    | 200.00 | 2022最新专辑     |
+----+-----------+--------+------------------+
1 row in set (0.00 sec)

>>> m02=Music(title='SHE',price=180)
>>> m02.info='2021最新专辑'
>>> m02.save()

MySQL [mysite3]> select * from zhuanji;
+----+-----------+--------+------------------+
| id | title     | price  | info             |
+----+-----------+--------+------------------+
|  1 | 周杰伦    | 200.00 | 2022最新专辑     |
|  2 | SHE       | 180.00 | 2021最新专辑     |
+----+-----------+--------+------------------+
2 rows in set (0.00 sec)
```

#### 查询数据

通过`MyModel.objects`管理器调用查询方法。

| 方法      | 说明                               |
| --------- | ---------------------------------- |
| all()     | 查询全部记录，返回QuerySet查询对象 |
| get()     | 查询复合条件的的单一记录           |
| filter()  | 查询复合条件的多条记录             |
| exclude() | 查询复合条件之外的全部记录         |
| ...       | ...                                |

```python
>>> all=Music.objects.all()
>>> for m in all:
...     print(m.title)
... 
周杰伦
SHE
```

可以再MyModel中定义`__str__`方法，即可达到自定义`QuerySet`输出格式的目的。
```python
>>> from music.models import Music
>>> al=Music.objects.all()
>>> al
<QuerySet [<Music: title:周杰伦 price:200 info:2022最新专辑>, <Music: title:SHE price:180 info:2021最新专辑>]>
```

**查询部分列:`values('列1','列2')`**
用法:`MyModel.objects.values()`
作用: 查询部分列的数据并返回

```python
>>> partialCols=Music.objects.values('title','price')
>>> partialCols
<QuerySet [{'title': '周杰伦', 'price': Decimal('200.00')}, {'title': 'SHE', 'price': Decimal('180.00')}]> //返回值QuerySet中放的是字典
>>> for ele in partialCols:
...     print(ele['price'])
... 
200.00
180.00
```

**查询部分列:values_list('列1','列2')**
用法: `MyModel.objects.values_list('title','price')`

```python
>>> partialCols=Music.objects.values_list('title','price')
>>> partialCols
<QuerySet [('周杰伦', Decimal('200.00')), ('SHE', Decimal('180.00'))]> //返回值QuerySet中存放的是元组,用索引访问
>>> for ele in partialCols:
...     print(ele[0])
... 
周杰伦
SHE
```

**排序: order_by()**
用法: `MyModel.objects.order_by('-列','列')`, `-`表示降序;

```python
>>> a1=Music.objects.order_by('-price')
>>> a1
<QuerySet [<Music: title:周杰伦 price:200 info:2022最新专辑>, <Music: title:SHE price:180 info:2021最新专辑>]>
>>> a1=Music.objects.order_by('price')
>>> a1
<QuerySet [<Music: title:SHE price:180 info:2021最新专辑>, <Music: title:周杰伦 price:200 info:2022最新专辑>]>
```

组合方式:
```python
>>> a5=Music.objects.values('title').order_by('price')
>>> a5
<QuerySet [{'title': 'SHE'}, {'title': '周杰伦'}]>
```

**打印QuerySet对应的SQL语句:**

```python
>>> print(a5.query)
SELECT `zhuanji`.`title` FROM `zhuanji` ORDER BY `zhuanji`.`price` ASC
```

**结合views/urls/templates的案例**

```python
// 文件: music/urls.py
urlpatterns =[
    path('index', views.index_view),
    path('zhuanji', views.zhuanji_view),
]

//文件: music/views.py

def zhuanji_view(request):
    m1=Music.objects.all()
    return render(request,'music/zhuanji.html',locals())
    
//music/templates/music/zhuanji.html
        <thead>
          <tr>
            <th scope="col">#</th>
            <th scope="col">专辑名</th>
            <th scope="col">价格</th>
            <th scope="col">描述</th>
          </tr>
        </thead>
        <tbody>
            {% for item in m1 %}
            <tr>
                <th scope="row">{{forloop.counter}}</th>
                <td>{{item.title}}</td>
                <td>{{item.price}}</td>
                <td>{{item.info}}</td>
            </tr>
            {% endfor %}
        </tbody>
```

**条件查询:filter**
语法:`MyModel.objects.filter(属性1=值1,属性2=值2)`,所有条件默认做and操作,返回满足所有条件的数据。
最终返回`QuerySet`。

```python
>>> m2=Music.objects.filter(title='SHE')
>>> m2
<QuerySet [<Music: title:SHE price:180 info:2021最新专辑>]>
```

**反向条件查询:exclude**
语法:`MyModel.objects.exclude(属性1=值1,属性2=值2)`
最终返回`QuerySet`。

```python
>>> m2=Music.objects.exclude(title='SHE')
>>> m2
<QuerySet [<Music: title:周杰伦 price:200 info:2022最新专辑>]>
```

**get:条件查询,只返回一条结果**
语法: `MyModel.objects.get(属性1=值1)`
作用: 返回满足条件的唯一一条语句。
查询结果返回超过1条语句，则抛出`Model.MultiObjectsReturned`异常;
查询结果为空,则抛出`Model.DoesNotExist`异常。

```python
>>> m3=Music.objects.get(id=1)
>>> m3
<Music: title:周杰伦 price:200 info:2022最新专辑>
```

**非等值查询/范围查询**

- 查询谓词

  - `__exact`:等值查询

    ```python
    >>> m3=Music.objects.filter(id__exact=1)
    >>> 
    >>> m3
    <QuerySet [<Music: title:周杰伦 price:200 info:2022最新专辑>]>
    >>> print(m3.query)
    SELECT `zhuanji`.`id`, `zhuanji`.`title`, `zhuanji`.`price`, `zhuanji`.`info` FROM `zhuanji` WHERE `zhuanji`.`id` = 1
    ```

  - `__contains`:包含指定的值,等同于`like '%xxxx%'`

  - `__startswith`:功能等同于`like 'xxx%'`

  - `__endswith`:功能等同于`like '%xxxx'`

    ```python
    >>> m3=Music.objects.filter(title__startswith='周')
    >>> m3
    <QuerySet [<Music: title:周杰伦 price:200 info:2022最新专辑>]>
    >>> print(m3.query)
    SELECT `zhuanji`.`id`, `zhuanji`.`title`, `zhuanji`.`price`, `zhuanji`.`info` FROM `zhuanji` WHERE `zhuanji`.`title` LIKE BINARY 周%
    ```

  - `__gt`:大于指定值,等同于`>`

  - `__gte`:大于等于指定值,等同于`>=`

  - `__lt`:小于指定值,等同于`<`

  - `__lte`:小于等于指定值,等同于`<=`

    ```python
    >>> m3=Music.objects.filter(price__gt=100)
    >>> m3
    <QuerySet [<Music: title:周杰伦 price:200 info:2022最新专辑>, <Music: title:SHE price:180 info:2021最新专辑>]>
    >>> print(m3.query)
    SELECT `zhuanji`.`id`, `zhuanji`.`title`, `zhuanji`.`price`, `zhuanji`.`info` FROM `zhuanji` WHERE `zhuanji`.`price` > 100
    ```

  - `__in`: 枚举几个值,功能等同于`in ('xx01','xx02')`

  - `__range`:查询数据是否在某个区间,等同于`between xxx and xxx`

    ```python
    >>> m3=Music.objects.filter(price__in=(100,180))
    >>> m3
    <QuerySet [<Music: title:SHE price:180 info:2021最新专辑>]>
    >>> print(m3.query)
    SELECT `zhuanji`.`id`, `zhuanji`.`title`, `zhuanji`.`price`, `zhuanji`.`info` FROM `zhuanji` WHERE `zhuanji`.`price` IN (100, 180)
    
    >>> m3=Music.objects.filter(price__range=(100,190))
    >>> m3
    <QuerySet [<Music: title:SHE price:180 info:2021最新专辑>]>
    >>> print(m3.query)
    SELECT `zhuanji`.`id`, `zhuanji`.`title`, `zhuanji`.`price`, `zhuanji`.`info` FROM `zhuanji` WHERE `zhuanji`.`price` BETWEEN 100 AND 190
    ```

#### 更新数据

**单行数据更新**

过程大概是: 查 -> 改 -> 保存

```python
>>> m3=Music.objects.get(id=1)
>>> m3
<Music: title:周杰伦 price:200 info:2022最新专辑>
>>> m3.price=220
>>> m3.save()
```

**批量更新: 先拿到QuerySet,然后update**

```python
>>> m3=Music.objects.all()
>>> m3
<QuerySet [<Music: title:周杰伦 price:220 info:2022最新专辑>, <Music: title:SHE price:180 info:2021最新专辑>]>
>>> m3.update(info='歌手最新专辑')
2
```

#### 删除数据

**单个数据删除:先查 -> 再删除**

```python
>>> a3=Author.objects.get(id=3)
>>> a3.delete()
(1, {'music.Author': 1})
```

#### F对象和Q对象

**F对象**

- 通常是对数据中的字段值在不获取的情况下进行操作
- 用于类属性(字段)之间的比较
- 语法: `from django.db.models import F(列名)`

示例1: 更新所有Music实例,让所有实例价格涨10元

```python
// 旧方法,啰嗦,浪费数据库流量
>>> a4=Music.objects.all()
>>> for item in a4:
...     item.price=item.price+10
...     item.save()

// F对象方法
from django.db.models import F
Music.objects.all().update(price=F('price')+10) // update music set price=price+10
```

示例2: 对数据库中两个字段的值进行比较，列出哪些书的零售价高于定价
```python
from django.db.models import F
from bookstore.models import Book
books = Book.objects.filter(market_price__gt=F('price'))
// select * from bookstore_book where bookstore_book.market_price > bookstore_book.price;
```

**Q对象**

运算符: `& 与操作`、`| 或操作`、`~非操作`
语法:

- `Q(条件1)|Q(条件2)` 条件1成立 或 条件2成立
- `Q(条件1)&Q(条件2)` 条件1和条件2同时成立
- `Q(条件)&~Q(条件2)` 条件1成立且条件2不成立

当在获取查询结果时，通过Q对象事项 逻辑或`|`、逻辑非`~/!`

```python
// 价格小于210 同时 title 中包含 SHE 的专辑
>>> a5=Music.objects.filter(Q(price__lt=210)|Q(title__contains='SHE'))
>>> a5
<QuerySet [<Music: title:SHE price:200 info:歌手最新专辑>]>
>>> print(a5.query)
SELECT `zhuanji`.`id`, `zhuanji`.`title`, `zhuanji`.`price`, `zhuanji`.`info` FROM `zhuanji` WHERE (`zhuanji`.`price` < 210 OR `zhuanji`.`title` LIKE BINARY %SHE%)
```

### 聚合查询和原生数据库操作

**整表聚合**

- 导入方法: `from django.db.models import *`
- 聚合函数:`Sum`、`Avg`、`Count`、`Max`、`Min`
- 语法:`MyModel.objects.aggregate(结果变量名=聚合函数('列'))`；返回结果: 结果变量名和值组成的字典

```python
>>> from django.db.models import Count
>>> Music.objects.aggregate(res=Count('id'))
{'res': 2}
```

**分组聚合**

- 语法:`QuerySet.annotate(结果变量名=聚合函数('列'))`
- 返回值:`QuerySet`
- 步骤
  - 先用查询结果`MyModel.objects.values`查找要分组聚合的列
  - 通过返回结果的`QuerySet.annotate`方法分组聚合得到分组结果

```python
>>> from django.db.models import Count
>>> music_set=Music.objects.values('title')
>>> music_set
<QuerySet [{'title': '周杰伦'}, {'title': 'SHE'}, {'title': '周杰伦'}]>
>>> music_count=music_set.annotate(cnt=Count('title'))
>>> music_count
<QuerySet [{'title': '周杰伦', 'cnt': 2}, {'title': 'SHE', 'cnt': 1}]>

>>> music_set.annotate(cnt=Count('title')).filter(cnt__gt=1) //加上过滤条件
<QuerySet [{'title': '周杰伦', 'cnt': 2}]>
>>> print(music_set.annotate(cnt=Count('title')).filter(cnt__gt=1).query)
SELECT `zhuanji`.`title`, COUNT(`zhuanji`.`title`) AS `cnt` FROM `zhuanji` GROUP BY `zhuanji`.`title` HAVING COUNT(`zhuanji`.`title`) > 1 ORDER BY NULL
```

**原生数据库操作(难处理SQL注入问题)**
查询: `MyModel.objects.raw(sql语句,拼接参数)` 
返回值:`RawQuerySet`集合对象，**只支持基础操作，比如循环**

```python
>>> from music.models import Music
>>> m1=Music.objects.raw('select * from zhuanji where id=1')
>>> m1
<RawQuerySet: select * from zhuanji where id=1>
>>> for ele in m1:
...     print(ele)
... 
title:周杰伦 price:240 info:歌手最新专辑
>>> 
>>> for ele in m1:
...     print(ele.title)
... 
周杰伦
```

SQL注入防范:
错误:

```python
m1=Music.objects.raw('select * from zhuanji where id=%s'%('1 or 1=1'))
```

正确:

```python
m1=Music.objects.raw('select * from zhuanji where id=%s',['1 or 1=1'])
```

**原生数据库操作-cursor**

完全跨过模型类操作数据库 —— 查询/更新/删除

1. 导入cursor所在的包: `from django.db import connection`
2. 用创建cursor类的构造函数创建cursor对象，再使用cursor对象。为保证在出现异常时能释放cursor资源，通常使用with语句进行创建操作

```python
from django.db import connection
with connection.cursor() as cur:
    cur.execute('执行SQL语句','拼接参数')

# 用SQL语句将id=10的书出版社改为 "XXX出版社"
from django.db import connection
with connection.cursor() as cur:
    cur.execute('update bookstore_book set pub_house="xxxx出版社" where id=10;')

with connection.cursor() as cur:
    # 删除id为1的一条记录
    cur.execute('delete from bookstore_book where id=10;')
```

### 关系映射

**一对一: 表示事务间存在的一对一对应关系。如一个班中一个人唯一，一个人一个身份证号**
语法: `OneToOneField(类名,on_delete=xxx)`

```python
class A(model.Model):
   ...
class B(model.Model):
    属性 = models.OneToOneField(A,on_delete=xxx)
```

- `on_delete`的选项
  - `models.CASCADE` 级联删除, Django模拟`SQL`约束`ON DELETE CASCADE`行为，并删除包含`ForeignKey`的对象;
  - `models.PROTECT`抛出`ProtectedError`以阻止被引用对象的删除。等同于MySQL默认的`RESTRICT`;
  - `SET_NULL`设置`ForeignKey null`；需要指定`null=True`;
  - `SET_DEFAULT`将`ForeignKey`设置为默认值，必须设置`ForeignKey`的默认值;

```python
class Author(models.Model):
    name = models.CharField('作者名',max_length=11,null=False,default='')
    age = models.IntegerField('年龄',default=0)
    email = models.EmailField('邮箱',default='')

class Wife(models.Model):
    name = models.CharField('妻子名',max_length=11,null=False,default='')
    author = models.OneToOneField(Author,on_delete=models.CASCADE)

MySQL [mysite3]> show create table music_author\G
*************************** 1. row ***************************
       Table: music_author
Create Table: CREATE TABLE `music_author` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(11) NOT NULL,
  `age` int(11) NOT NULL,
  `email` varchar(254) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

MySQL [mysite3]> show create table music_wife\G
*************************** 1. row ***************************
       Table: music_wife
Create Table: CREATE TABLE `music_wife` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(11) NOT NULL,
  `author_id` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `author_id` (`author_id`),
  CONSTRAINT `music_wife_author_id_a0582439_fk_music_author_id` FOREIGN KEY (`author_id`) REFERENCES `music_author` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)
```

**创建数据**

```python
>>> from music.models import Author,Wife
>>> a1=Author.objects.get(id=1)
>>> w1=Wife.objects.create(name='kunlin',author=a1) //指定author对象
>>> w1.name
'kunlin'
>>> w1.author
<Author: Author object (1)>
>>> w1.author.id
1
>>> w1.author.name
'JayChou'

>>> w2=Wife.objects.create(name='zhangsan',author_id=4) //指定author_id
>>> w2.id
2
>>> w2.author
<Author: Author object (4)>
>>> w2.author.name
'lisi'
```

**查询数据**

- 正向查询: 谁有外键字段，则通过外键属性查询

  ```python
  from .models import wife
  wife = wife.objects.get(name='王夫人')
  print (wife.name,'的老公是',wife.author.name)
  ```

- 反向查询: 没有外键的一方，通过反向属性查询到关联的另一方。当反向引用不存在时,会触发异常

  反向关联属性为`实例对象.引用类名(小写)`,如作家的反向应用为`作家对象.wife`

  ```python
  >>> a1=Author.objects.get(id=1)
  >>> a1.name
  'JayChou'
  >>> a1.wife.name
  'kunlin'
  ```


**一对多场景: 如一个班级多个学生, 一个明星多不电影等**
一对多需明确出具体角色,在多表上设置外键。

```python
class Publisher(models.Model):
    '''出版社[一]'''
    name= models.CharField('名称',max_length=50,unique=True)

class Book(models.Model):
    '''书[多]'''
    title = models.CharField('书名',max_length=50)
    publisher = models.ForeignKey(Publisher,on_delete=models.CASCADE)
```

**数据创建: 先创建一,再创建多**

```python
>>> from music.models import Publisher,Book
>>> pub1 = Publisher.objects.create(name='清华大学出版社')
>>> Book.objects.create(title='C++',publisher=pub1)
<Book: Book object (1)>
>>> Book.objects.create(title='Java',publisher_id=pub1.id)
<Book: Book object (2)>
```

**正向查询[通过Book查询Publisher]**

```python
>>> abook=Book.objects.get(id=1)
>>> print(abook.title,'的出版社是',abook.publisher.name)
C++ 的出版社是 清华大学出版社
```

**反向查询[通过Publisher查询对应的所有Book]**

```python
>>> pub1=Publisher.objects.get(name='清华大学出版社')
>>> books=pub1.book_set.all() //注意这里用的是 book_set
>>> for book in books:
...     print(book.title)
>>> print(pub1.book_set.all().query)
SELECT `music_book`.`id`, `music_book`.`title`, `music_book`.`publisher_id` FROM `music_book` WHERE `music_book`.`publisher_id` = 1
```

**多对多场景:每个人都待过不同学校,每个学校都有不同的学生**

- MySQL 中多对多关系需创建第三张表来实现

- Django 无需手动创建第三张表,Django自动完成;

- 语法: 在关联的两个类中的任意一个类增加:

  `属性=models.ManyToManyField(MyModel)`

```python
class Student(models.Model):
    name = models.CharField('姓名',max_length=11)

class School(models.Model):
    name = models.CharField('学校名',max_length=11)
    students = models.ManyToManyField(Student)
```

生成了三个表:

```sql
MySQL [mysite3]> show create table music_school\G
*************************** 1. row ***************************
       Table: music_school
Create Table: CREATE TABLE `music_school` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

MySQL [mysite3]> 
MySQL [mysite3]> show create table music_student\G
*************************** 1. row ***************************
       Table: music_student
Create Table: CREATE TABLE `music_student` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

MySQL [mysite3]> show create table music_school_students\G
*************************** 1. row ***************************
       Table: music_school_students
Create Table: CREATE TABLE `music_school_students` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `school_id` int(11) NOT NULL,
  `student_id` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `music_school_students_school_id_student_id_4465b564_uniq` (`school_id`,`student_id`),
  KEY `music_school_students_student_id_02254bff_fk_music_student_id` (`student_id`),
  CONSTRAINT `music_school_students_school_id_1cb4f6f6_fk_music_school_id` FOREIGN KEY (`school_id`) REFERENCES `music_school` (`id`),
  CONSTRAINT `music_school_students_student_id_02254bff_fk_music_student_id` FOREIGN KEY (`student_id`) REFERENCES `music_student` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)
```

**数据创建**

- 先创建school，再关联 student

  ```python
  >>> from music.models import Student,School
  >>> s01=School.objects.create(name='县一中')
  >>> s02=School.objects.create(name='市一中')
  >>> student01=s01.students.create(name='小王')
  >>> s02.students.add(student01)
  ```

- 先创建学生,在创建school

  ```python
  student02=Student.objects.create(name='张三')
  student02.school_set.create(name='xxx大学') //等同创建大学
  
  student03=Student.objects.create(name='李四')
  s03=School.objects.get(id=3)
  student03.school_set.add(s03) //关联学校
  ```

**数据查询**

```python
// 正向查询
>>> for ele in s01.students.all():
...     print(ele.name)
... 
小王

>>> for ele in s01.students.filter(name__contains='小'):
...     print(ele.name)
... 
小王

// 反向查询
>>> for ele in student03.school_set.all():
...     print(ele.name)
... 
县一中
xxx大学
```

### admin管理后台

- django提供了比较完善的后台管理数据库接口，可供开发过程中调用和测试；

- Django 会搜集所有已注册的模型类，为这些模型类提供数据管理界面，共开发者使用

- 创建管理后台账号——该账号为管理后台最高权限账号
  `python3 manage.py createsuperuser`

- 进入管理后台:`http://127.0.0.1:8000/admin/`

  <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220717222243542.png" alt="image-20220717222243542" style="zoom:50%;" />
