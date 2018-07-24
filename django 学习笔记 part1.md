#django 学习笔记 part1

@(django)

[TOC]

###1、建立项目

```
django-admin startproject mysite
```

让我们看看文件夹的框架:

```
mysite/
	    manage.py
	    mysite/
        __init__.py
        settings.py
        urls.py
        wsgi.py`
```

manage.py: 一个命令行工具，让你能与django项目交互
mysite/wsgi.py: 一个WSGI兼容服务器的入口I
mysite/settings.py: 项目配置文件
mysite/urls.py: 项目url路径地址

接下来我们看项目是否能正常运行，进入文件夹然后输入以下命令：

```
python manage.py runserver
```

####修改端口
可以在runserver后面跟上以下代码来修改端口。

```
python manage.py runserver 8080
```

如果你想修改服务器的ip，可以把它和端口放在一起，这样就能让其他机器访问了。（特别是在你想向其他机器炫耀你的杰作！）

```
python manage.py runserver 0.0.0.0:8000
```


###2、创建app

```
python manage.py startapp polls
```
让我们看看文件夹的框架:
```
polls/
    __init__.py
    admin.py
    apps.py
    migrations/
        __init__.py
    models.py
    tests.py
    views.py
```

我们需要映射view到一个URL，在polls文件夹建立一个urls.py文件，在urls.py里面写入以下代码：
####polls/urls.py

```python
from django.urls import path

from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

再对比之前看下目录结构的变化：

```
polls/
    __init__.py
    admin.py
    apps.py
    migrations/
        __init__.py
    models.py
    tests.py
    urls.py
    views.py
```

下一步是在根目录的URL设置里引用polls.urls模块，在mysite/urls.py增加引用django.conf.urls.include，把include()插入到urlpatterns列表里面：
####mysite/urls.py

```python
from django.urls import include, path
from django.contrib import admin

urlpatterns = [
    path('polls/', include('polls.urls')),
    path('admin/', admin.site.urls),
]
```
include()函数允许引用其它的URL设置。记住这里的正则表达式不包含$(代表匹配结束)，’/polls/’后面的URL应该在polls/urls.py里面。你应该总是使用include()，除了唯一的例外：admin.site.urls

接着打开polls/views.py文件，在里面写入下面的代码：
####polls/views.py

```python
from django.http import HttpResponse


def index(request):
    return HttpResponse("Hello, world. You're at the polls index.")
```

运行服务器然后进入http://localhost:8000/polls/,你将见到你定义的index view。

https://docs.djangoproject.com/en/2.0/intro/tutorial01/