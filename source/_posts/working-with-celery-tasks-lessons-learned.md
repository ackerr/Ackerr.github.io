---
title: 使用Celery的一些经验教训（翻）
date: 2019-09-14 18:02:09
tags:
    - Celery
    - Python
categories:
    - 翻译系列
---

> 本文为渣翻文章，原文地址: [这里](https://blog.daftcode.pl/working-with-asynchronous-celery-tasks-lessons-learned-32bb7495586b)

本文将使用一个简单的例子来展示使用Celery可以遇到的最常见的问题。

### 栗子

假设我们正在写一个网上商城APP，一个模块负责新用户注册成功后，发送欢迎邮箱。

本文使用Django的伪代码，因为简短易懂，且可以应用到其他web框架。

处理`create user`的请求如下:

```python
@transaction.commit_on_success
def create_user(request):
   user = User.objects.create(name=request.name, email=request.email)
   send_email(email=user.email, subject='Welcome', content='...')
   return HttpResponse(render_html('user_created.html'))
```

首先，我们用ORM创建一个包含`name, email`的User对象，然后调用`send_email`方法，通过STMP发送邮件给用户，最后返回HTML页面。这里还使用了装饰器`transaction.commit_on_success`， 在函数执行结束并且没有抛出异常时，自动提交User对象至数据库。

<!--more-->

### 仔细选择异步任务参数

如你所知，通过SMTP发送邮箱会导致一些问题，例如服务器响应时间过长，服务不可用。我们提取`send_email`相关逻辑给Celery task，让发送邮件是一个单独的过程。让`create_user`可以更快的返回结果，无论发送邮件有什么潜在的问题。修改之后，代码如下：

```python
@transaction.commit_on_success
def create_user(request):
   user = User.objects.create(name=request.name, email=request.email)
   send_welcome_email_task.delay(user)
   return HttpResponse(render_html('user_created.html'))

@app.task()
def send_welcome_email_task(user):
   send_email(email=user.email, subject='Welcome', content='...')
```

一切看起来变得简洁了，但有个小问题。这里传递给`send_welcome_email_task`的参数是一个User实例，一个绑定了当前数据库连接的ORM对象。这个task将由不同的进程执行，使用不同的数据库连接，所以传参使用User实例并不是一个好主意。此外，**User对象可能会被同步修改，导致Celery中使用一个过时的版本。**如果对象中还包括一些复杂的字段像datatime类型，不能被序列化为JSON（Celery最常用的序列化）。但不要担心，可以使用user.id代替user作为task的参数解决这个问题。

```python
@transaction.commit_on_success
def create_user(request):
   user = User.objects.create(name=request.name, email=request.email)
   send_welcome_email_task.delay(user.id)
   return HttpResponse(render_html('user_created.html'))

@app.task()
def send_welcome_email_task(user_id):
   user = User.objects.get(id=user_id)
   send_email(email=user.email, subject='Welcome', content='...')
```

现在，序列化`user.id`并不会有什么问题，因为传参现在只是一个简单的int。并且user实例将通过id在执行task开始时，从数据库中读取，所以可以确定现在的user对象是最新的一个版本。

> 使用简单类型作为task的传参

记住，实例对象在发送到broker之后，执行之前，可能会被更新。这是特别重要的一点，当两个task同时操作一个实例对象时，第二个应该知道第一个所做的更改。

> 执行Celery task时，总是获取实例对象的最新版本

### 发送任务与数据库事务

如果`render_html`需要执行很长时间，Celery worker可能在`create_user`  提交user实例完成之前，执行task，导致`send_welcome_email_task`中抛出异常`User object not found in the database for a given id`。这个场景也可能是在将task发生Celery broker之后，执行一些耗时长的操作导致。

那怎么解决这个问题呢？首先想到的是，延迟执行task的时间，例如2秒，确保Celery worker获得broker之前函数返回：

```python
send_welcome_email_task.apply_async(args=(user.id,), countdown=2)
```

这种简单的解决方法，在执行`render_html`需要超过2秒的时候，显然会失败。它将导致发生一份不必要的邮件。

那有没有更好的方法？这里不得不确保Celery task发送至broker不能早于数据库事务提交。为了实现这个条件，可以使用装饰器`transaction.commit_manually`去控制数据库事务。

```python
@transaction.commit_manually
def create_user(request):
   try:
       user = User.objects.create(name=request.name, email=request.email)
       response = HttpResponse(render_html('user_created.html'))
   except:
       transaction.rollback()
       raise
   else:
       transaction.commit()
       send_welcome_email.delay(user.id)
       return response
```

有了这个新的调整，在事务被提交后，task才被发送至Celery，从而解决了竞态。这种实现方法还有一个优点：

只有在事务成功提交后，并且`create_user`函数没有任何异常的情况下，才会发送task至Celery。先前`render_html`导致的异常，依然会发生邮箱的情况，并不是我们所期望的。

> tip: 大型项目中，解决这种问题的最好方法是扩展Celery task的基类，在事务提交后，使用事务事件发送task。这将自动化各式各样异步任务操作数据库的过程，使代码更加简洁（[一个Django例子](https://medium.com/hypertrack/dealing-with-database-transactions-in-django-celery-eac351d52f5f)）

> 在函数成功提交事务并且没有任何异常后，再发送异步任务至Celery broker

### 棘手的失败任务

让事情变得复杂一些。如果有一个短暂的SMTP服务问题，重试发送邮件是一个很好的结果。我们可以用Celery的重试机制来实现。它需要添加两个参数给Celery task装饰器和捕获异常。

现在如果有任何问题在发送邮件期间发生，task将会每60秒重新执行，直到它执行成功或失败达到上限次数（最大120次）。从理论上讲，这应该没有任何问题，但如果使用Redis作为broker，不一定。为什么呢？

Redis使用`visibility_timeout`参数来定义时间限制，worker需要知道这个task执行是否成功，如果没有，这个task将被定义为丢失，会被重新发送至一个worker。`visibility_timeout`的默认值是一个小时。在这个例子中，它意味着如果一个电子邮件一个小时没有被发送，task将被重新发送。然而如果task没有丢失，而是在等待重试中，将会出现两个相同的task。如果最后两个都执行成功，用户将会收到两份邮件，这是不希望出现的行为。

> tip: 如果你想避免与`visibility_timeout`相关问题，可以考虑使用RabbitMQ作为Celery broker

> 更多关于`visibility_timeout`信息可以看[Celery文档](http://docs.celeryproject.org/en/latest/getting-started/brokers/redis.html)和[GitHub issue](https://github.com/celery/kombu/issues/337)

第一个想到的解决方法是增加`visibility_timeout`，例如2小时，但如果其他任务重新超过2小时呢？如果不需要去重发等待超过一个小时的任务，当一个任务真的丢失了（例如worker网络不可用）

最好的办法是编写一个异步任务，不关心任务执行多少次，但核心逻辑（这里是发送邮件）执行不超过一次。

这就是所谓的幂等性任务，幂等使任务必须遵循两个步骤。

首先，任务执行开始时，我们必须检查邮件是否已经被发送。为了达到要求，需要给User表增加`is_welcome_email_sent`布尔字段，如果字段为True，异步任务应该不做任何事，否则，应该发送邮件并将`is_welcome_email_sent`设置为True。

第二，需要保证代码在竞态条件下，可能发生两个任务同时试图检查`is_welcome_email_sent`值。如果发生，一份邮件会被发生两次。这里将使用读锁`select_for_update()`来保证user对象是同一时间只有一个worker在处理。其他的任务不得不等待`select_for_update()`，直到第一个锁被释放（提交事务）。答案应该如下所示：

```python
@transaction.commit_on_success
@app.task(bind=True, default_retry_delay=60, max_retries=120)
def send_welcome_email_task(user_id):
   user = User.objects.select_for_update().get(id=user_id)
   if not user.is_welcome_email_sent:
       try:
           send_email(email=user.email, subject='Welcome', content='...')
       except SMTPException as e:
           self.retry(exc=e)
       else: # no exception
          user.is_welcome_email_sent = True
          user.save()
```

即使一个任务会在多个worker之间，但只有第一个真正执行。剩下的任务只会检查是否已经完成，不会造成其他副作用。

> 保证幂等性，确保task只完成一次

### late acknowledgment

当所有任务都是幂等时，可以使用`late acknowledgment`

```python
@app.task(bind=True, default_retry_delay=60, max_retries=120, acks_late=True)
```

默认情况下，worker确认task仅仅在task被broker接收时（执行task前），称为`early acknowledgment`。这意味着如果执行过程中，worker奔溃，task不会被再次执行，因为它被认为执行成功。结果就是这个task不会被完成。

在设置为`late acknowledgment`后，task会在worker执行后确认。如果worker执行过程中奔溃，task会再次执行。这是一个好的解决方式，因为可以确保整个任务至少被执行一次。不过它也有一次缺点：大部分task会被执行两次。幸运地，我们的task都是幂等的，所以我们防止业务员逻辑不会被多次执行。

> 使用`late acknowledgment`，确保幂等的task一定会被执行
