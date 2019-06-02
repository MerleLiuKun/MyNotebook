在flask项目中使用celery作异步任务处理时，由于异步的任务比较多，就想着把这些任务独立出一个模块，在视图函数中调用。之后测试时遇到了一个问题：

原项目结构如下：

> —webapp
>
> ——task_module
>
> ——views
>
> —— configs
>
> —manager.py

manage中初始化了celery和flask的app.

在views的request Handler中调用task任务，在task_module中写了对应的任务。

运行时，提示无法导入views中的handler模块。



看了异常，是明显的循环导入错误。 仔细分析一下，原来在任务代码中引入了与falskapp同文件的celeryapp. 造成导入错误。

那么怎么处理比较好呢？

尝试了几种方法，感觉自己陷入了误区， 又找了资料， 原来celery可以拉出来独立做一个模块。(从celery的运行方式就应该想到的)，这样的话，把celery的初始化独立出一个模块。如下：

>— webapp
>
>—— task_module
>
>—— views
>
>—— configs
>
>— manage.py
>
>— celery_app.py

现在就可以正常的运行啦。



参考资料：

https://stackoverflow.com/questions/29670961/flask-blueprints-uses-celery-task-and-got-cycle-import

