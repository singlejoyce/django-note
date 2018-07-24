# django 学习笔记 part7

@(django)


[TOC]

## 自定义你的admin表单
当使用admin.site.register(Question)注册你的Question model后，Django可以构造一个默认的表单形式.往往，你希望去自定义admin表单外观.

让我们看怎么让fieds在admin表单里可编辑：

polls/admin.py:

```python
from django.contrib import admin

from .models import Question


class QuestionAdmin(admin.ModelAdmin):
    fields = ['pub_date', 'question_text']

admin.site.register(Question, QuestionAdmin)
```

这个改变可以使”Publication date’成为admin里面’Question’的field

你也许想把表单按照field的集合进行分离:
polls/admin.py:

```
from django.contrib import admin
from .models import Question

class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None,          {'fields': ['question_text']}),
        ('Date information', {'fields': ['pub_date']}),
    ]
admin.site.register(Question, QuestionAdmin)
```

## 增加相关的对象

现在，我们在admin有了Question的页面，但是一个Question有多个Choice，而admin没有显示choices页面。

polls/admin.py:

```
from django.contrib import admin
from .models import Choice, Question

#...
admin.site.register(Choice)
```
但是，最好在创建Question对象的时候就把Choice加进去
polls/admin.py:

```
from django.contrib import admin
from .models import Choice, Question

class ChoiceInline(admin.StackedInline):
    model = Choice
    extra = 3

class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None,          {'fields':['question_text']}),
        ('Date information', {'fields':['pub_date'],'classes:['collapse']}),
    ]
    inlines = [ChoiceInline]

admin.site.register(Question, QuestionAdmin)
```

这些代码告诉Django， ’Choice‘对象可以在’Question‘里面编辑。这里有个小问题，花费了太多页面空间去显示这些choice对象。出于这个原因，Django提供了一个管状显示，你只需要把StackedInline改为TabularInline。


## 自定义admin的‘change list’
默认情况下，Django为每个对象显示它的str()函数。但有时你想显示更多一些信息，可以使用list_display，这是一个包含field名字的元组：

polls/admin.py:

```
class QuestionAdmin(admin.ModelAdmin):
    # ...
    list_display = ('question_text', 'pub_date')
```
为了更多的信息，我们把was_published_recently()方法加进去：
```
class QuestionAdmin(admin.ModelAdmin):
    #...
    list_display = ('question_text', 'pub_date', 'was_published_recently')
```
你可以点击每列的标题来排序 – 除了was_published_recently标题，因为Django不支持根据自定义类方法来进行排序。且让我们注意到，was_published_recently的标题名，默认情况下就是它的方法名(下划线被替换成空格)，把返回结果作为string为每一行显示出来。

你可以改进一下类方法，给它几个额外属性：
polls/models.py:

```
class Question(models.Model):
    # ...
    def was_published_recently(self):
        now = timezone.now()
        return now - datetime.timedelta(days=1) <= self.pub_date <= now
    was_published_recently.admin_order_field = 'pub_date'
    was_published_recently.boolean = True
    was_published_recently.short_description = 'Published recently?'
```

更多的方法属性，请看list_display。

再一次修改你的polls/admin.py为他增加你改进过的Question’change list’:过滤器使用list_filter，把它增加到QuestionAdmin:`list_filter = ['pub_date']`

现在我们在admin页面的右边可以看到‘过滤器’边框，过滤器一句你给定的field类型来过滤，因为pub_date是一个DateTimeField,Django明白应该可以给它哪些适当的选项：”Any date”,”Today”,”Past 7 days”,”This month”,”This year”.

让我们增加一些搜索功能：`search_fields = ['question_text']`
这会在页面的顶部增加一个搜索框。当某人输入一下条目，Django会在question_text里面搜索。你可以使用很多你想用的field，原理是在SQL语句中使用LIKE查询，搜索field数量合适的话可以让你的数据库更容易搜索。

现在是个好时候让我们理解一下分页，admin页面附带了分页功能，默认是每个页面显示100个条目。Change list pagination -|||- search boxes -|||- filters -|||- date-hierarchies -|||- column-header-ordering 以上链接都应该去看看。

## 自定义admin的外观和感觉
明显的,’Django管理’在每个admin页面的顶部有点可笑，它仅仅是个占位符文本。

这很容易更改，只要使用Django的模版系统。Django的admin也是由Django框架创建的，admin的界面使用Django自己的模版系统。

### 自定义你的项目模版

在你的项目根文件夹创建一个templates文件夹（和manage.py在同一目录）。templates可以放在任何地方，但是保持你的templates放在根目录诗歌好习惯。

打开设置文件(mysite/settings.py)，在TEMPLATES的可选参数DIRS增加值：

mysite/settings.py:

```
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
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
DIRS是一个包含文件系统管理器的列表，当Django要使用templates时从这个参数读取。
现在在templates里面创建一个admin文件夹，把Django库里的admin/base_site.html（django/contrib/admin/templates）复制进去。

现在我们可以打开文件把 {{ site_header|default:_(‘Django administration’) }}修改成合适的网站名，比如这样

mysite/templates/admin/base_site.html:

```
{% block branding %}
<h1 id="site-name"><a href="{% url 'admin:index' %}">Polls Administration</a></h1>
{% endblock %}
```
我们用这个方法教你怎么覆盖模版。在真正的项目里，你将更可能使用django.contrib.admin.AdminSite.site_header属性来更简单的自定义。

这个模版里包括一些{% block branding %}，{{ title }}。这些是Django模版语言的一部分。

如果你想覆盖模版，就像你覆盖base_site.html一样。从源模版把文件复制到你自定义的文件夹内，并做出修改。

### 自定义你的应用模版
可能有人会问：可是DIRS是空的时候，Django是怎么找到默认的admin模版的？答案在这，一旦APP_DIRS设置为True，Django将自动在每个应用包的templates/子目录下寻找，找到后回调使用（别忘了django.contrib.admin是一个应用）

我们的投票应用不太复杂也不需要定制的admin模版。但是当它变的更复杂、为了一些功能需要修改更多Django标准admin模版，就需要一些自定义的修改了。

想知道更多关于Django怎么寻找模版的？请看template loding documentation


https://docs.djangoproject.com/en/2.0/intro/tutorial07/