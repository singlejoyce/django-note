# django 学习笔记 part6

@(django)


[TOC]

## 自定义你app的外观
首先，在你的polls文件夹下面创建一个文件夹，叫做static。

Django的STATICFILES_FINDER设置包含了一队列的指示器，它指出了怎样从不同的资源处发现静态文件。有一个默认的finder叫做AppDirectoriesFinder，它从每一个INSTALLED_APPS的‘static’子目录下寻找文件。

在static文件夹下再创建一个polls文件夹，在里面创建一个style.css文件。换句话说，你的样式表的位置应该在polls/static/polls/style.css。

在css中加入代码：

polls/static/polls/style.css:

```
li a{
    color: green;
}
```

接下来打开html文件，在文件头部引入css：

polls/templates/polls/index.html:

```
{% load static %}

<link rel="stylesheet" type="text/css" href="{% static 'polls/style.css' %}" />
```
模版标签{% static %}生成指向静态文件的绝对路径。

现在载入http://localhost:8000/polls/，你会看到现在链接会变成绿色。

## 添加背景图片

接下来，我们创建一个文件夹叫做images，放在polls/static/polls/目录下，在里面放入一张图片，取名为background.gif，换句话说，现在你的图片在polls/static/polls/images/background.gif.

polls/static/polls/style.css:

```
body {
    background: white url("images/background.gif") no-repeat right bottom;
}
```
警告

{% static %}模版标签不适用于不是用Django生成的样式表，以后你需要一直使用相对路径来链接你的静态文件。因为这样可以方便你修改STATIC_URL而不用去修改一大堆你的静态文件路径。

https://docs.djangoproject.com/en/2.0/intro/tutorial06/