#django 学习笔记 part2

@(django)


[TOC]

###建立数据库
打开mysite/settings.py.
默认情况下，配置里使用SQLite

现在我们修改使用mysql数据库：
1.使用pip安装pymysql包：`pip install pymysql` 

2.在Application中修改Django默认使用的MySQLdb包，因为MySQLdb包仅仅支持python2.7，不支持python3，而Django内置使用于连接MySQL的只有MySQLdb，因此需要转换成pymysql这个几乎和MySQLdb一样包，但是支持python3.5的。

在settings.py 添加以下几行代码：

```python
import pymysql  
pymysql.install_as_MySQLdb() 
```

3.在项目下面修改默认数据框，使用MySQL连接。Django默认数据库是SQLite，因此要修改成MySQL。
找到setting.py文件，将其下的DATABASES修改成如下格式：


```python
DATABASES = {  
	     'default': {  
	     'ENGINE': 'django.db.backends.mysql',  
	     'NAME': '', #MySQL中Schema名字  
	     'USER':'root',  
	     'PASSWORD':'', #MySQL的密码  
	     'HOST':'127.0.0.1', #默认本地  
	     'PORT':'3306' #默认3306端口  
    }  
}  
```

4.执行manage.py操作：在项目文件下打开CMD执行：

```
python manage.py makemigrations  
python manage.py migrate  
```

执行中可能会输出错误日志：

```
  File "C:\Python36\lib\site-packages\django\db\utils.py", line 110, in load_backend
  return import_module('%s.base' % backend_name)
  File "C:\Python36\lib\importlib\__init__.py", line 126, in import_module
  return _bootstrap._gcd_import(name[level:], package, level)
  File "C:\Python36\lib\site-packages\django\db\backends\mysql\base.py", line 36, in <module>
  raise ImproperlyConfigured("mysqlclient 1.3.3 or newer is required; you have %s" % Database.__version__)
django.core.exceptions.ImproperlyConfigured: mysqlclient 1.3.3 or newer is required; you have 0.7.11.None
```
解决方案：在\Python3.6.1\lib\site-packages\django\db\backends\mysql\base.py把下面的内容注释掉

```python
if version < (1, 3, 3):
     raise ImproperlyConfigured("mysqlclient 1.3.3 or newer is required; you have %s" % Database.__version__)
```

5.需要创建数据库表等操作，都需要在Application下的models.py中操作。


当你编辑mysite/settings.py时，首先修改TIME_ZONE为你自己的时区:`TIME_ZONE = 'Asia/Shanghai'`


接下来，注意到文件顶部的INSTALLED_APPS.这里已经包括了Django实例里正在运行的应用。Apps可以用于多个项目，你可以打包并分配它们去其他的项目。
默认情况下，INSTALLED_APPS包括下面的app，都是Django自身附带的：

django.contrib.admin - admin站点，你马上就会用到它.
django.contrib.auth - 一个身份验证系统
django.contrib.contenttypes - 一个内容类别的框架
django.contrib.sessions - 一个回话框架
django.contrib.messages - 一个消息框架
django.contrib.staticfiles - 一个管理静态文件的框架

在我们用数据库之前先要创建它，使用下面的命令：


```
python manage.py migrate
```


###创造models
在我们简单的poll app里面，只需要创建两个models：Question和Choice。一个Question包含有一个问题和发布日期，一个Choice包含有两个域：choice的文本和投票的计分。每个Choice都与Question联结。

这个概念可以用简单的Python类来表达，编辑polls/models.py文件：
#####polls/models.py：

```python
from django.db import models


class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')


class Choice(models.Model):
    question = models.ForeignKey(Question,on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
```


###触发models
为了让project包含app，我们需要引用它的配置类，添加到INSTALLED_APPS设置。PollsConfig类在polls/apps.py，它的点式路径(dotted path)是’polls.apps.PollsConfig‘,编辑mysite/settings.py文件然后添加点路径到INSTALLED_APPS。像下面这样：
#####mysite/settings.py:

```python
INSTALLED_APP = [
    'polls',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

现在，Django知道包含了polls app，让我们运行另一条命令：

```
python manage.py makemigrations polls
```

你应该看到类似下面的输出：

```
Migrations for 'polls':
    polls/migrations/0001_initial.py
        - Create model Choice
        - Create model Question
        - Add field question to choice
```

当运行makemigrations时，你告诉Django你对你的models做了一些更改而且你希望你的变化作为migration保存下来。

Migrations是怎样在Django存储你的models（也就是你的数据库schema）-它们存储在硬盘上。如果你愿意你可以阅读这个migration；它存储在polls/migrations/0001_initial.py.不用担心，你不需要每次都去读它们，只是它们设计出来是可人为编辑的，在万一你需要手工扭转Django改变的东西。

这些这个命令可以为你运行migrations，可以自动管理你的数据库schema-这个命令叫做migrate，一会就要提到-但是首先，让我们看看migration会运行怎样的SQL。sqlmigrate命令拿到migrations的名字之后返回它的SQL。

```
python manage.py sqlmigrate polls 0001
```

如果你感兴趣，你可以运行python manage.py check;这个可以检查你项目的任何问题，在不制造migrations或者触碰数据库的情况下。
现在，再一次运行migrate，用以在你的数据库创造models表。

```
python manage.py migrate
```

migrate命令接收所有没有应用过的migrations（Django追踪你用过的每一个migration，方法是在你的数据库使用一个特别的表，叫做django_migrations)且对你的数据库应用它们 - 本质上，在你对models作出改变后同步执行这些命令。

Migration是一个强力的功能，让你任何时候都能改变自己的models，在你开发你的项目时，除了不能删除你的数据库或表，也不能创建新的 - 它特定地用来动态更新你的数据库，而不会遗失数据。在后面的章节会对这个功能涉及很深，但是现在，记住这个“三步法”应用你的model改变。

改变你的models（在models.py)
运行 python manage.py makemigrations 来为这些变化创建migrations
运行 python manage.py migrate 来对数据库应用这些变化


阅读 django-admin documentation，里面有完整的信息，可以在里面获取 manage.py 工具的用法。

让我们为Question和Choice添加一个__str__:
#####polls/models.py:

```python
from django.db import models


# Create your models here.
class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')

    def __str__(self):
        return self.question_text


class Choice(models.Model):
    question = models.ForeignKey(Question,on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)

    def __str__(self):
        return self.choice_text

```

加入`__str__()`很重要，不仅仅是方便你在命令行提示你，表现形式同样遍及Django的整个自动化里面。

让我们加入一个自定义方法，像下面一样

```python
import datetime
from django.db import models
from django.utils import timezone

class Question(models.Model):
    #...
    def was_publised_recently(self):
        return self.pub_date >= timezone.now() - datetime.timedelta(days=1)
```

这个函数的意义显而易见，如果你想知道更多的timezone方法，可以阅读官方文档time zone support docs.


###建立一个admin用户

运行以下的命令来创建一个可以登录admin网页的账号：

```
 python manage.py createsuperuser
```
接下来跟着提示输入相关信息即可,注册成功并确保服务器在运行时即可登录http://127.0.0.1:8000/admin/。之后你就能看到admin登录界面了。（界面的语言默认在settings.py）里面设定，简体中文建议LANGUAGE_CODE = ‘zh-hans’）。


让poll app 可以在admin里面修改
一开始你在admin界面是看不到自己的app的，只要做一件事就可以了：
####polls/admin.py:

```python
from django.contrib import admin
from .models import Question

admin.site.register(Question)
```

###探索admin的功能
http://127.0.0.1:8000/admin/


https://docs.djangoproject.com/en/2.0/intro/tutorial02/