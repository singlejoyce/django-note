# django 学习笔记 part4

@(django)


[TOC]

## 写一个简单的form
polls/templates/polls/detail.html:

```htmlbars
<h1>{{ question.question_text }}</h1>

{% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}

<form action="{% url 'polls:vote' question.id %}" method="post">
{% csrf_token %}
{% for choice in question.choice_set.all %}
    <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}" />
    <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br />
{% endfor %}
<input type="submit" value="Vote" />
</form>
```

快速总结：

以上模版为每个question的choice提供了一个单选的按钮，每个单选按钮的值和choice.id相连接。name属性都是“choice”，这意味着，当媒人选了一个按钮并提交了表单，它就发送了一个POST数据 choice = # ，#就是所选按钮的CHOICE.ID值（VALUE属性）。这是HTML表单的基本概念。
设置表单的action为 {% url ‘polls:vote’ question.id %},跟着我们设置method=”post”。使用method=”post”提交表单非常重要，这不是特定使用Django的方法，而是一个好的Web开发习惯。
forloop.counter 只是for标签循环的次数
当我们创立表单时，需要担忧安全问题。但是Django自带一个非常容易使用的系统来防御这些，总之，当POST表单指向一个内部URL的时候使用{% csrf_token %}标签。


接下来我们创建vote功能，打开view文件：
polls/views.py:

```python
from django.shortcuts import get_object_or_404, render
from django.http import HttpResponseRedirect, HttpResponse
from django.urls import reverse

from .models import Choice, Question
# ...
def vote(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    try:
        selected_choice = question.choice_set.get(pk=request.POST['choice'])
    except (KeyError, Choice.DoesNotExist):
        # Redisplay the question voting form.
        return render(request, 'polls/detail.html', {
            'question': question,
            'error_message': "You didn't select a choice.",
        })
    else:
        selected_choice.votes += 1
        selected_choice.save()
        # Always return an HttpResponseRedirect after successfully dealing
        # with POST data. This prevents data from being posted twice if a
        # user hits the Back button.
        return HttpResponseRedirect(reverse('polls:results', args=(question.id,)))
```

这些代码包括一些马上要提到的概念：

request.POST 是一个类字典的对象，让你能通过键名取得数据。
request.POST[‘choice’]如果没找到健值对会引发KeyError。如果没有选择一个选项就提交，上面的代码检测Keyerror并且重新显示question表单和一个错误信息。
最后返回一个HttpResponseRedirect，并附带一个参数：将要重定向的URL。提交表单数据后应该总是使用HttpresponseRedirect，这是个好的Web开发习惯。
我们在HttpResponseRedirect里面使用了reverse()函数。这个函数避免我们使用硬编码来编写一个URl，并且可以附带参数给URL
最后vote()重定向到results页面，让我们继续编写views：
polls/views.py:

```python
def results(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, 'polls/results.html', {'question': question})
```

随后创建polls/results.html模版：
polls/results.html:
```htmlbars
<h1>{{ question.question_text }}</h1>

<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
{% endfor %}
</ul>

<a href="{% url 'polls:detail' question.id %}">Vote again?</a>
```

现在去/polls/1/页面可以看到投票选项。投票后可以看到更新的票数情况，如果你没选择一项就提交，就会看见错误信息。

注意
现在vote()代码还有些小问题，selected_choice对象来自数据库，计算新的votes值，并把它们返回给数据库。但如果有两个用户刚好在同时投票，这就会发生错误：比如现在是42票，两人一起投票后应该是44票，但其实结果是43票。

这叫做竞争条件(race conditon)。如果你感兴趣，可以读读[Avoiding race conditions using F()](https://docs.djangoproject.com/en/2.0/ref/models/expressions/#avoiding-race-conditions-using-f)学习一下怎么避免这个情况。

## 使用一般的views:代码少点更好

detail()和results() views都很简单。但是，也有一些冗余。包括index().

这些views代表了一种基本的Web开发例子：根据传入URL的参数从数据库取得数据,读取模版并返回渲染后的模版.因为这种用法太常见，Django提供了一个更简便的方法，叫做”generic views”系统。

“Generic views”抽象化平常的patterns，让你在某些地方可以少写很多Python代码。

让我们对polls 应用使用’generic views‘，所以要删一大把之前写的代码。然后用下面几个步骤来做出转变：
1.覆盖URLconf
2.删除一些老的，不需要的views
3.引入Django的’generic views‘

### 为什么要重洗代码？

一般情况下，我们写一个Django应用，你会先评估‘generic views’是否适合你的问题，然后在一开始就使用它们而不是写到一半再重构。但是这篇指导的目的是教会你Django的核心概念

在使用计算器之前你也得会一点数学吧。

## 修改URLconf

首先，打开polls/urls.py文件并修改：
polls/urls.py:

```python
from django.urls import path

from . import views

app_name = 'polls'
urlpatterns = [
    path('', views.IndexView.as_view(), name='index'),
    path('<int:pk>/', views.DetailView.as_view(), name='detail'),
    path('<int:pk>/results/', views.ResultsView.as_view(), name='results'),
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```
注意，在第二,三个pattern里，从`<question_id>` 修改为 `<pk>`.

## 修改views

下一步，让我们删去旧的index,detail和results views并使用Django的’generic views’代替。

polls/views.py:
```python
from django.views import generic


class IndexView(generic.ListView):
    template_name = 'polls/index.html'
    context_object_name = 'latest_question_list'

    def get_queryset(self):
        """Return the last five published questions."""
        return Question.objects.order_by('-pub_date')[:5]


class DetailView(generic.DetailView):
    model = Question
    template_name = 'polls/detail.html'


class ResultsView(generic.DetailView):
    model = Question
    template_name = 'polls/results.html'
```

我们在这里使用了两个views类：ListView和DetailView。这两个views分别抽象了‘显示一个列表的对象’和‘显示特定类别对象的详细信息’的概念。

每个’generic view’需要知道对哪个model生效，所以提供了model属性
DetailView期望从URL获取一个主键值，也叫‘pk’。所以我们在URLconf把question_id修改为pk
默认情况下，DetailView使用的模版从/_detail.html获取。在我们的例子里，使用对template_name属性赋值的方法更改了取得模版的位置。

同样的，ListView默认从/_list.html获取模版，我们使用template属性告诉ListView现在在”polls/index.html“已经存在模版，使用它就行了。

在之前的教学中，我们为模版提供了上下文环境包括question和latest_question_list环境变量。但是在DetailView中，只要我们使用Django model(Question),question变量会自动提供，Django会为环境变量判定一个适当的名字。然而，ListView自动生成的环境变量是question_list,覆盖的方法是我们赋值给context_object_name属性，声明我们希望用latest_question_list代替。反过来，你也可以修改你的模版来匹配默认的环境变量。

更多的关于”generic views“的详细信息，在[generic views documentation.](https://docs.djangoproject.com/en/2.0/topics/class-based-views/)

https://docs.djangoproject.com/en/2.0/intro/tutorial04/

