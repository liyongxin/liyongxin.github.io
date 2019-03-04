---
layout: post
title: "python多线程执行类方法"
date: 2019-03-03 16:56:27 +0800
comments: true
categories: python, threads, case
---

## case概述
python可以使用多线程处理多个任务，最基本的处理思路如下：

- 定义一个基类，然后基于该类实现多个子类完成业务功能；
- 为每个子类的调用分配一个子线程，并调用子线程的业务入口；
- 运行子线程，主线程实现任务结果处理；

基于以上思路，做了基本的case，代码参考[github地址](https://github.com/liyongxin/case-python/tree/master/thread_test)，项目结构如下：
{% highlight python %}
.
└── thread_test
    ├── __init__.py
    ├── collector
    │   ├── __init__.py
    │   ├── c1.py
    │   ├── c2.py
    │   └── collector.py
    ├── threads.py
    └── threads_queue.py
{% endhighlight %}

## 代码简介
#### 定义一个基类，然后基于该类实现多个子类完成业务功能
`collector`包下定义了`collector`这个基类，然后`c1.py与c2.py`分别是实现了该基类的子类，同时使用`get_data`模拟业务逻辑的入口，
{% highlight ruby %}
from .collector import Collector

class C1(Collector):
    @classmethod
    def get_data(cls, msg):
        print("i am from ", msg)
        return msg
{% endhighlight %}
<!--more-->
####为每个子类的调用分配一个子线程，并调用子线程的业务入口
- 获取所有的业务子类

为了给每个子类分配一个子线程，需要先获取所有的子类，使用递归的方式，根据基类名称来读取所有基类的实现类
{% highlight ruby %}
def get_all_subclasses(mod):
    all_subclasses = {}
    ## collections
    for subclass in .__subclasses__():
        class_name = subclass.__name__.lower()
        if not class_name in all_subclasses.keys():
            all_subclasses[class_name] = subclass
        get_all_subclasses(subclass)
    return all_subclasses
{% endhighlight %}

- 实现多线程类，并在默认的线程入口run方法中调用注册方法
{% highlight python %}
import threading
from thread_test.collector.collector import Collector

class GrabThread(threading.Thread):

    def __init__(self,func,args=()):
        super(GrabThread,self).__init__()
        self.func = func
        self.args = args
        self.result = None

    def run(self):
    """run func registry by sub-class"""
        self.result = self.func(*self.args)

    def get_result(self):
        try:
            return self.result
        except Exception as e:
            return str(e)
{% endhighlight %}

- 主线程注册子类到子线程中，并进行调用
{% highlight python %}
def main():
    all_cls = get_all_subclasses(Collector)
    threads = []
    for item, value in all_cls.items():
        fn_collect = getattr(value, 'get_data', None)
        if callable(fn_collect):
            p = GrabThread(func=fn_collect, args=(item,))
            threads.append(p)
            p.start()
            print("执行{0}的collect 方法".format(item))
    for p in threads:
        p.join()  #

    print("main done")
{% endhighlight %}

##执行结果
每个线程负责一个子任务，并行运行
{% highlight python %}
#/usr/local/Cellar/python/3.7.1/Frameworks/Python.framework/Versions/3.7/bin/python3.7 /Users/liyongxin/PycharmProjects/case-python/thread_test/threads.py
i am from  c1
执行c1的collect 方法
i am from  c2
执行c2的collect 方法
main done

Process finished with exit code 0
{% endhighlight %}