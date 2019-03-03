---
layout: post
title: "somethings about ansible"
date: 2019-02-28 14:18:47 +0800
comments: true
categories: ansible
---

- ####task解析json并注册变量给其他task使用
多个task涉及变量传递时，有时taskA取值是json格式，taskB若想使用json中的数据，可以通过如下方式：
{% highlight python %}
# get config task stdout is json format
- name: "get_config"
  shell: /usr/local/sbin/kubectl get cm global-var -n alauda-system -o json
  register: cms

# use from_json plugin
- set_fact: data="{{ cms.stdout | from_json }}"

- name: "read_config"
  shell: /usr/local/bin/redis-cli -a
  when: data['kind'] == 'ConfigMap'
{% endhighlight %}

- ####ansible自定义模块执行命令支持管道
自定义ansible module时，可以默认使用`AnsibleModule`的实例`run_command`来执行shell命令,可以指定use_unsafe_shell=True来让ansible支持管道。
use_shell会和shell模块类似，若不指定，默认为False，则以command模块运行，command是比较安全的隧道连接，但是不支持管道
{% highlight python %}
def main():
    """The main function."""
    module = AnsibleModule(argument_spec=dict(),supports_check_mode=False)
    module.run_command("/usr/local/sbin/kubectl get po -n alauda-system |grep alauda-redis|awk \'END{print $1}", use_unsafe_shell=True)
    redis = Redis(module)
    rst = redis.get_redis_status()
    module.exit_json(**rst)
{% endhighlight %}

- ####多进程执行ansible任务
实现故障定位工具时为提高执行效率，打算采用多线程方式同时执行多个任务，