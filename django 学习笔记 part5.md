# django 学习笔记 part5

@(django)


[TOC]

## 自动化测试入门
什么是自动化测试？ 
测试时检查你代码运行状况的常规手段。

测试操作层次有很大区别，有些可能需要处理一些极小的细节（比如特定model方法是否返回期待值？），或是检查软件的所有操作途径（一连串的用户输入是否会返还正确的值？），你可以手动在shell里面检测，输出数据看它是否按相应情况做出更改。

自动化测试有哪些不同呢？你创造一组测试，当你改变你的应用时，你可以检查一下代码是不是像你最初想的一样运行，不需要浪费时间去手动测试。

## 创建一个测试来暴露这个bug

惯例是把应用的测试放在各自应用目录下的tests.ph文件里；测试系统会自动寻找以test开头的文件。

polls/tests.py:

```python
import datetime
from django.utils import timezone
from django.test import TestCase
from .models import Question

class QuestionMethodTests(TestCase):
    def test_was_published_recently_with_future_question(self):
        """ 
        was_published_recently() 当使用未来日期对象的时候,
        应该会返回False 
        """
        time = timezone.now() + datetime.timedelta(days=30)
        future_question = Question(pub_date=time)
        self.assertIs(future_question.was_published_recently(), False)

```

## 运行测试

在终端，我们输入如下命令： `python manage.py test polls`

```
F:\study\python\mysite> python manage.py test polls
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
E
======================================================================
ERROR: test_was_published_recently_with_future_question (polls.tests.QuestionMod
elTests)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "F:\study\python\mysite\polls\tests.py", line 18, in test_was_published_r
ecently_with_future_question
    self.assertIs(future_question.was_published_recently(), False)
AttributeError: 'Question' object has no attribute 'was_published_recently'

----------------------------------------------------------------------
Ran 1 test in 0.003s
```

`python manage.py test polls`找到polls 应用里的测试程序
1.它找到一个django.test.TestCase 的子类
2.它为这次测试创建了一个临时的特殊数据库
3.它搜索测试方法 – 名字以‘test’开头的方法
4.在test_wat_published_recently_with_future_question它创建了一个Question实例，为pub_date域传入了30天以后的日期
5.最后使用了assertIs()方法。它发现was_published_recently()返回True，但是它预期是False

## 修复这个bug

我们已经知道问题出在哪里，现在去修改代码：
polls/models.py：

```python
def was_published_recently(self):
    now = timezone.now()
    return now - datetime.timedelta(days=1) <= self.pub_date <= now
```

再一次运行test：Ran 1 test in 0.001s，一切ok

我们的应用可能之后还会发生很多错误，但我们不会再疏忽让这个bug出现，因为只要运行测试就马上会发出警告，我们可以考虑应用的这小部分是安全的了。


## 更多综合性测试
增加2个新方法：
polls/tests.py

```
def test_was_published_recently_with_old_question(self):
    """
    was_published_recently() returns False for questions whose pub_date
    is older than 1 day.
    """
    time = timezone.now() - datetime.timedelta(days=1, seconds=1)
    old_question = Question(pub_date=time)
    self.assertIs(old_question.was_published_recently(), False)

def test_was_published_recently_with_recent_question(self):
    """
    was_published_recently() returns True for questions whose pub_date
    is within the last day.
    """
    time = timezone.now() - datetime.timedelta(hours=23, minutes=59, seconds=59)
    recent_question = Question(pub_date=time)
    self.assertIs(recent_question.was_published_recently(), True)
```

## 改进我们的view

打开polls/views.py，我们需要修改get_queryset()方法，让它检查日期并且和当前时间做比较。
polls/views.py

```python
from django.utils import timezone

def get_queryset(self):
    """
    Return the last five published questions (not including those set to be
    published in the future).
    """
    return Question.objects.filter(
        pub_date__lte=timezone.now()
    ).order_by('-pub_date')[:5]
```

Question.objects.filter(pub_date__lte=timezone.now())返回一个包括pub_date少于或等于（也就是时间上早于或等于）现在的查询集合。lte == less than equal

## 测试我们的新view

创建一个test，以刚才的shell内容为基础

```python
def create_question(question_text, days):
    """
    Create a question with the given `question_text` and published the
    given number of `days` offset to now (negative for questions published
    in the past, positive for questions that have yet to be published).
    """
    time = timezone.now() + datetime.timedelta(days=days)
    return Question.objects.create(question_text=question_text, pub_date=time)


class QuestionIndexViewTests(TestCase):
    def test_no_questions(self):
        """
        If no questions exist, an appropriate message is displayed.
        """
        response = self.client.get(reverse('polls:index'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "No polls are available.")
        self.assertQuerysetEqual(response.context['latest_question_list'], [])

    def test_past_question(self):
        """
        Questions with a pub_date in the past are displayed on the
        index page.
        """
        create_question(question_text="Past question.", days=-30)
        response = self.client.get(reverse('polls:index'))
        self.assertQuerysetEqual(
            response.context['latest_question_list'],
            ['<Question: Past question.>']
        )

    def test_future_question(self):
        """
        Questions with a pub_date in the future aren't displayed on
        the index page.
        """
        create_question(question_text="Future question.", days=30)
        response = self.client.get(reverse('polls:index'))
        self.assertContains(response, "No polls are available.")
        self.assertQuerysetEqual(response.context['latest_question_list'], [])

    def test_future_question_and_past_question(self):
        """
        Even if both past and future questions exist, only past questions
        are displayed.
        """
        create_question(question_text="Past question.", days=-30)
        create_question(question_text="Future question.", days=30)
        response = self.client.get(reverse('polls:index'))
        self.assertQuerysetEqual(
            response.context['latest_question_list'],
            ['<Question: Past question.>']
        )

    def test_two_past_questions(self):
        """
        The questions index page may display multiple questions.
        """
        create_question(question_text="Past question 1.", days=-30)
        create_question(question_text="Past question 2.", days=-5)
        response = self.client.get(reverse('polls:index'))
        self.assertQuerysetEqual(
            response.context['latest_question_list'],
            ['<Question: Past question 2.>', '<Question: Past question 1.>']
        )
```

首先创立一个快速建立question实例的函数，减少代码输入量。

test_index_view_with_no_questio没有创建question,所有检查response会发现’No polls are available.’，经过确认latest_question_list也确实是空的。记住django.test.TestCase类提供了很多抛出异常的assert方法。在这里我们用到了assertContains()和assertQuerysetEqual()。

在test_index_view_with_a_past_question中，我们创建了一个question，并且确认它在list中。

在test_index_view_with_a_future_question中，我们创建了一个pub_date在未来的question。测试数据库在每次测试都会重置，所以第一个question已经不在里面了，经确认list为空。

## 测试DatailView

即使日期为未来的question不出现在index，用户也可以自己靠猜的构造一个URL访问它，所以我们需要在DetailView添加相似的限制。

polls/views.py:

```
class DetailView(generic.DetailView):
    ...
    def get_queryset(self):
        """
        Excludes any questions that aren't published yet.
        """
        return Question.objects.filter(pub_date__lte=timezone.now())
```
然后我们添加一些tests，检查detail页面，如果pub_date是以前的话就可以显示，如果是未来则不现实

polls/tests.py:

```
class QuestionDetailViewTests(TestCase):
    def test_future_question(self):
        """
        The detail view of a question with a pub_date in the future
        returns a 404 not found.
        """
        future_question = create_question(question_text='Future question.', days=5)
        url = reverse('polls:detail', args=(future_question.id,))
        response = self.client.get(url)
        self.assertEqual(response.status_code, 404)

    def test_past_question(self):
        """
        The detail view of a question with a pub_date in the past
        displays the question's text.
        """
        past_question = create_question(question_text='Past Question.', days=-5)
        url = reverse('polls:detail', args=(past_question.id,))
        response = self.client.get(url)
        self.assertContains(response, past_question.question_text)
```

## 更多的测试想法

我们应该给ResultsView创建一个get_queryset方法，并且创建一个新的测试类，和上面的代码会有很大的重复。

我们也可以在其它方面改进我们的应用。比如检测没有发布Choice的Question，如果有则不显示。我们的测试会创造一个没有Choice的Question实例看看它有没有显示在网页上，或者我们可以创建与之相反的测试，创建一个有Choice的实例，测试它是否显示在页面上。

也许登录的admin用户可以看到未发布的Question。再次声明，为软件增加功能也需要增加响应的测试。

## 测试的时候，代码多就是好

看起来我们的测试代码可能臃肿的难以掌控，甚至测试代码比程序本身还多，代码重复也是不美观的。

但是这些都不重要，让他们臃肿。大多数情况，你都是写一个测试然后就忘了它了。但是只要你继续开发你的程序，它就会继续发挥自己的作用。

有时候，test也需要更新。假如你修改了View，使得只有Questions于Choices可以发布，万一发生这种情况，很多现存的tests就会失效。

为了让你的tests看起来更易于管理一些，可以养成下面这些好习惯

每个model或者view都有一个分开的TestClass子类
一个分离的test设置所有你想测试的情况
test方法名里面描述它的功能

## 进一步了解测试

这篇教学仅仅介绍了一些基础的测试。还有很多你可以做的，以及一些很有用的工具。

例如，你可以用内置浏览器框架Selenium来测试你的HTML是否成功渲染。Django内置的LiveServerTestCase整合了一些类似Selenium的工具。

如果你有个复杂的应用，你可能想每次提交时都自动测试，请看continuous integration。

检查你的测试覆盖率，给帮你识别脆弱的死代码。如果你没法测试一片代码，通常意味着这些代码需要重构或删除。更多细节看integration with coverage.py。

关于测试的综合信息在Testing in Django.

https://docs.djangoproject.com/en/2.0/intro/tutorial05/