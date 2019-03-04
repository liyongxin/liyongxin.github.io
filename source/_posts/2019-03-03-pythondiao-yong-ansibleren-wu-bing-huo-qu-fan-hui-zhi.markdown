---
layout: post
title: "python调用ansible任务并获取返回值"
date: 2019-03-03 17:59:19 +0800
comments: true
categories: ansible, python
---

## 背景
项目中需要使用python执行ansible的任务，每项任务均需要获取执行结果并做解析，为了方便调用，对ansible的api调用做了封装并对返回结果做了友好处理，特做记录。
## 理想中的调用方式
先不考虑封装逻辑，个人作为使用者，希望的一种使用方式如下(伪代码)：
{% highlight python %}
if __name__ == "__main__":
    host_list = ['192.168.8.33', '192.168.8.35']
    api = AnsibleApi(host_list, user=root, password=123)
    # 传递playbook的path，执行playbook task
    api.run_playbook(["playbook_path"])
    # 得到执行结果
    return api.get_playbook_result()

{% endhighlight %}
即我告诉api，我想在host_list所在的机器上执行playbook_path这个playbook指定的任务，然后我得到一个友好的执行结果

##理想中的友好返回
因为ansible会在多个机器执行多个任务，每个机器上的每个任务的执行情况都需要收集，于是作为开发者，更期望得到如下结构的返回结果：
<!--more-->
{% highlight python %}
    """
    return data
    {
            "task_name1": {
                "success": {
                    "192.168.8.33": {
                        {"changed": "true", "end": "2019-01-17 02:31:58.519839", "stdout": "{}" ...}
                    },
                    "192.168.8.34": {
                    }
                },
                "failed": {
                    "192.168.8.35": {
                    }
                },
                "unreachable": {
                    "192.168.8.36": {
                    }
                },
                "skipped": {
                }
            },
            "task_name2": {
                "success": {
                    "192.168.8.33": {
                        {"changed": "true", "end": "2019-01-17 02:31:58.519839", "stdout": "{}" ...}
                    },
                    "192.168.8.35": {
                    }
                    "192.168.8.34": {
                    }
                },
                "failed": {
                },
                "unreachable": {
                    "192.168.8.36": {
                    }
                },
                "skipped": {
                }
            },
        }
    """
{% endhighlight %}
即以我的playbook中的每个task为核心(key值)，告诉我每个task的执行结果，结果包含了该task在哪些机器上执行成功了，哪些机器上执行失败了，并且无论
成功还是失败，输出结果告知我情况。
那么因为task_name是可以传递到playbook中的，所以开发者很容易就可以获取任何一个task的执行情况，进而做后续的逻辑处理。

## 如何针对上述调用方式和返回结果进行封装
主要说明如何对playbook的调用结果做收集，对于直接调用ansible的模块的结果收集比较简单，代码中有实现，不做额外的说明了。
[代码地址](https://github.com/liyongxin/case-python/blob/master/ansible/api.py)，下面只对ansible执行后对callback做说明，其他部分参考代码即可。

- 自定义callback类，注册给ansible
{% highlight python %}
    def run_playbook(self, playbook_path, ip=None, **kwargs):
        #print('self_ips_run_playbook:',self.ips)
        # self.variable_manager.extra_vars = {'ansible_ssh_pass': self.default_password, 'disabled': 'yes'}
        self.variable_manager.extra_vars = self.args
        playbook = PlaybookExecutor(playbooks=playbook_path,
                                    inventory=self.inventory,
                                    variable_manager=self.variable_manager,
                                    loader=self.loader,
                                    options=self.options,
                                    passwords=self.password)
        # 注册callback
        playbook._tqm._stdout_callback = PlaybookResultCallback()

        return playbook.run()
{% endhighlight %}

- 实现自定义callback类

`ansible`的`plugins`中定义了`CallbackBase`，只需要实现该基础类即可。
大致的逻辑是按照上述期望的理想的返回结果进行数据拼装，实际使用过程中，发现如果有台机器是unreachable的，那么无论有多少task，如果遇到了unreachable
的情况，那么后续的task就不会在unreachable的机器上执行，因为在实现中做了人工补救`__fix_unreachable_result`，即对每个task来说，只要有unreachable
的机器，那么都会注册到task的返回结果中。
{% highlight python %}
class PlaybookResultCallback(CallbackBase):

    def __init__(self, *args, **kwargs):
        super(PlaybookResultCallback,self).__init__(*args, **kwargs)
        self.task_unreachable = []
        self.result = {}

    def __init_result_dict(self, result):
        if not result._task.name in self.result:
            self.result[result._task.name] = {
                KEYS_SUCCESS: {},
                KEYS_FAILED: {},
                KEYS_UNREACHABLE: {},
                KEYS_SKIPPED: {}
            }

    def __set_ansi_result(self, result, type):
        self.result[result._task.name][type][result._host] = result._result

    def __fix_unreachable_result(self, result):
        if self.task_unreachable:
            for task_unreachable in self.task_unreachable:
                unreachable_host = task_unreachable["host"]
                if not unreachable_host in self.result[result._task.name][KEYS_UNREACHABLE]:
                    self.result[result._task.name][KEYS_UNREACHABLE][unreachable_host] = task_unreachable["msg"]

    def v2_runner_on_unreachable(self, result, ignore_errors=True):
        print("unreachable task {} on host {}".format(result._task, result._host))
        task_unreachable = {
            "host": result._host,
            "msg": result._result
        }
        self.task_unreachable.append(task_unreachable)
        self.__init_result_dict(result)
        self.__set_ansi_result(result, KEYS_UNREACHABLE)
        #self.result[result._task.name][result._host] = result._result

    def v2_runner_on_ok(self, result, *args, **kwargs):
        print("playbook task {} ok on host {}".format(result._task, result._host))
        self.__init_result_dict(result)
        self.__set_ansi_result(result, KEYS_SUCCESS)
        self.__fix_unreachable_result(result)
        # self.result[result._task.name]["success"][result._host] = result._result
        ## add unreachable
        """if self.task_unreachable:
            for task_unreachable in self.task_unreachable:
                unreachable_host = task_unreachable["host"]
                if not unreachable_host in self.result[result._task.name]:
                    self.result[result._task.name][unreachable_host] = task_unreachable["msg"]"""

    def v2_runner_on_failed(self, result, ignore_errors=True, *args, **kwargs):
        print("failed task {} on host {}".format(result._task, result._host))
        self.__init_result_dict(result)
        self.__set_ansi_result(result, KEYS_FAILED)
        self.__fix_unreachable_result(result)

    def v2_runner_on_skipped(self, result, ignore_errors=True, *args, **kwargs):
        print("unreachable task {}  on host {}".format(result._task, result._host))
        self.__init_result_dict(result)
        self.__set_ansi_result(result, KEYS_SKIPPED)
        self.__fix_unreachable_result(result)
{% endhighlight %}