PK
    `�K?��Gs   s    % django 学习笔记 part3.mdup! ���django 学习笔记 part3.md# django 学习笔记 part3

@(django)


[TOC]

## 概貌

在我们的poll应用里，我们需要以下四个views：

1.Question的’index’索引页面 - 显示最近的若干questions
2.Question的’detail’细节页面 - 显示一个question的文本，不显示结果但有一个投票的表单
3.Question的’results’结果页面 - 显示特定question的结果
4.投票操作 - 处理特定question特定choice的投票

在Django，web页面和其他内容都是通过views发布。每个view都表现为一个简单的python函数（在一些以类为基础的views里，表现为类方法）。Django收到URL的请求后，选择一个view（注意是域名之后的URL，也就是相对URL）。

这个教程只涉及到极处的URLconfs，更多信息在django.urls

## 写更多的views
现在我们再写几个views，这几个和之前有些轻微的差别，它们都接受一个参数：

polls/views.py:

```python
def detail(request, question_id):
    return HttpResponse("You're looking at question {}.".format(question_id))

def results(request,question_id):
    response = "You're loking at the results of question {}."
    return HttpResponse(rensponse.format(question_id))

def vote(request,question_id):
    return HttpResponse("You're voting on question {}. ".format(question_id))
```

然后把这些新的views加入polls.urls：

```python
from django.urls import path

from . import views

urlpatterns = [
    # ex: /polls/
    path('', views.index, name='index'),
    # ex: /polls/5/
    path('<int:question_id>/', views.detail, name='detail'),
    # ex: /polls/5/results/
    path('<int:question_id>/results/', views.results, name='results'),
    # ex: /polls/5/vote/
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```

打开浏览器，看看’/polls/34/’,已经运行了detail()方法并且显示了你提供的ID，同样的也可以试试’/polls/34/results/’,’polls/34/vote/’。

当某人对你的网站的莫以页面发出请求-说：‘/polls/34/’,Django将会加载mysite.urls**python模块因为设置中这是ROOT_URLCONF。它发现了变量名叫**urlpatterns并且通过了正则表达式匹配，然后找到了匹配至‘^polls/’的’polls.urls’的URLconf，之后找到了匹配r’^(?P(0-9)+)/$’的view函数detail()，返回一个请求就像这样：

`detail(request=<HttpRequest object>, question_id='34')`

question_id=’34’部分来自(?P[0-9]+)。在pattern里使用括号捕获文本匹配，并将它当作一个参数传递给view函数；?P定义里名字并将之用来识别匹配的pattern。

## 写一些真正的views
每个view都负责做一两件事：返回包含请求页面内容的HttpResponse对象，或者引发一个exception，就像Http404。所有的Django最终都是一个HttpResponse,或者一个exception。

因为方便的原因，我们使用Django的数据库API。我们插入一个新的index() view，它可以根据pub_date显示最新的5个poll question，通过逗号分隔。

polls/view.py:
```python
from django.http import HttpResponse
from .models import Question

def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    output = ', '.join([q.question_text for q in latest_question_list])
    return HttpResponse(output)

# 剩下的暂不改变
```

这里有个问题，页面的设计是通过硬编码。如果你想更改页面的外观，不得不去修改python代码。所以让我们使用Django的模版系统来使设计分离。
第一步，在你的polls文件夹里，创建一个名为templates的文件夹，
polls/templates/polls/index.html

```htmlbars
{% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
        <li><a href="/polls/{{ question.id }}/">
            {{ question.question_text }}
            </a>
        </li>
    {% endfor %}
    </ul>
{% else %}
    <p> No polls are available.</p>
{% endif %}
```

然后更新index view，让它使用模版
polls/views.py:

```python
from django.http import HttpResponse
from django.template import loader
from .model import Question

def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    template = loader.get_template('polls/index.html')
    context = {
        'latest_question_list': latest_question_list,
    }
    return HttpResponse(template.render(context, request))
```

一个捷径:render()

常用的语法就是读取一个模版，传入一个上下文，并且返回一个包含渲染过的模版的HttpResponse对象。Django提供了一个更简洁的方法：

```
from django.shortcuts import render
from .models import Question

def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    context = {'latest_question_list': latest_question_list}
    return render(request, 'polls/index.html', context)
```

## 引发 404error
现在，我们来处理detail view - 这个页面显示question文本：
polls/views.py:

```python
from django.http import Http404
from django.shortcuts import render
from .models import Question

#...
def detail(request,question_id):
    try:
        question = Question.objects.get(pk=question_id)
    except Question.DoesNotExist:
        raise Http404('Question does not exist')
    return render(request, 'polls/detail.html', {'question':question })
```

这里有个新概念：当请求的question_id不存在，则会引发Http404错误。

## 一个捷径：get_object_or_404()

使用get()，raise Http404()是一个很常用的用法，Django仍然提供了更简洁的方法来代替：

polls/views.py:

```python
from django.shortcuts import get_object_or_404, render
from .models import Question

#...
def detail(request,question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(reqiest, 'polls/detail.html', {'question':question}
```

这个get_object_or_404()函数接受一个Django model作为它的第一个参数，然后接受任意数量的关键词参数，把这些关键词参数通过get()方法传递给model，如果object不存在就引发Http404。

还有一个get_list_or_404()函数，它和get_object_or_404()类似 – 除了用filter()代替get()，如果列表为空就会引发Http404错误。

## 使用模版系统
回到我们应用的detail() view。传递了一个context变量question，
polls/templates/polls/detail.html:

```html
<h1>{{ question.question_text }}</h1>
<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }}</li>
{% endfor %}
</ul>
```
版系统使用点查询语法来访问变量属性，在这个例子里 {{ question.question_text }}，试图进行一次属性查询，如果失败，则会试着进行一次列表索引式查询。更多关于templates请看[template guide.](https://docs.djangoproject.com/en/2.0/topics/templates/)


## 移去模版里的硬编码
我们在polls/index.html模版里写入链接时，链接部分是硬编码，如下：

```html
<li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
```

问题在于，使用硬编码，之后如果要更改URLs会很麻烦。然而，当你在polls.url模块的url定义了name参数后，你可以使用{% url %}模版标签去代替依赖于编写URL路径的方法。

```html
<li><a href=“{% url 'detail' question.id %}">{{ question.question_text }}</a></li>
```

## URL的命名空间
这个教学的项目仅仅只有一个app，在真实的Django项目里，可有有5，10，20或则更多。Django怎么区分它们的URL名字呢？举个例子，polls应用有一个detail视图，但是也许同一个项目的其它app也有detail视图，那么Django在使用{% url %}标签时要怎么区分它们呢？

答案是为你的URLconf增加命名空间，在polls/urls.py文件里，增加一个变量，叫做app_name,它的值设为命名空间，惯例是设置成应用名称。
polls/urls.py:

```python
from django.conf.urls import url
from . import views

app_name = 'polls'
urlpatterns = [
    #...
]
```

然后更改你的polls/index.html模版：

polls/templates/polls/index.html:

```html
<li><a href="{% url 'polls:detail' question.id %}">{{ question.question_text }}</a></li>
```

https://docs.djangoproject.com/en/2.0/intro/tutorial03/PK 
    `�K?��Gs   s    %               django 学习笔记 part3.mdup! ���django 学习笔记 part3.mdPK      o   �     