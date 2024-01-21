---
title: 使用logging模块时遇到的奇怪问题
date: 2024-1-21 13:20:52
index_img: /img/index-12.jpg
tags:
  - python
categories:
  - [python, logging]
  - [python, multiprocessing]
author: tao-wt@qq.com
excerpt: 使用logging模块时遇到的奇怪问题
---
## 背景
最近在写测试用例的自动化调度脚本时，用logging做的日志记录，但是脚本在运行时日志输出和预想的相差很多...
代码逻辑：主进程首先创建一个子进程collect_result来收集和处理每个testcase进程的运行结果，如日志的上传。然后主程序循环串行的为每个要执行的用例脚本创建新的进程testcase运行其代码，并将用例脚本的结果通过队列传递给collect_result进程。另外，所有子进程的日志都通过`logging.handlers.QueueHandler`传递给父进程的`logging.handlers.QueueListener`线程由其统一处理。

## 代码
代码如下：
```python
import random
import time
import logging
import logging.handlers
import multiprocessing
from queue import Empty


def worker_configure(queue):
    h = logging.handlers.QueueHandler(queue)
    root = logging.getLogger()
    root.addHandler(h)
    root.setLevel(logging.DEBUG)
    return root


class TestCase(multiprocessing.Process):
    def __init__(self, queue, r_q):
        super().__init__()
        worker_configure(queue)
        self.r_q = r_q

    def run(self):
        name = multiprocessing.current_process().name
        logging.info('%s, started', name)
        ss = random.randint(3, 5)
        time.sleep(ss)
        logging.info('Doing some work from %s', name)
        self.r_q.put((name, ss))


def listener_configure(queue):
    h = logging.StreamHandler()
    f = logging.Formatter(
        '%(asctime)s %(processName)-10s %(name)s %(levelname)-8s %(message)s'
    )
    h.setFormatter(f)
    root = logging.getLogger()
    root.addHandler(h)
    root.setLevel(logging.DEBUG)

    listener = logging.handlers.QueueListener(queue, h)
    listener.start()
    return listener


def collect_results(queue, r_queue):
    worker_configure(queue)
    logging.info("collect_results start")
    while True:
        try:
            tuple_info = r_queue.get(block=False)
        except Empty:
            # log.debug("queue is empty")
            time.sleep(0.5)
            continue
        if not tuple_info:
            break
        logging.info(tuple_info)
    logging.info("collect_results done")


def main():
    r_queue = multiprocessing.Queue()
    queue = multiprocessing.Queue(-1)
    listener = listener_configure(queue)
    logging.info("main start")

    result_collect = multiprocessing.Process(
        target=collect_results,
        name="collect_results",
        args=(queue, r_queue)
    )
    result_collect.start()

    for i in range(2):
        wp = TestCase(queue, r_queue)
        wp.start()
        wp.join()

    r_queue.put(None)
    result_collect.join()

    queue.put_nowait(None)
    listener.stop()
 

if __name__ == '__main__':
    main()

```
在windows11(python3.12)上运行结果如下所示, 所有TestCase进程的输出都丢失了。
```
PS C:\Users\tao> python .\desktop\logging_test.py
2024-01-21 15:35:20,019 MainProcess root INFO     main start
2024-01-21 15:35:20,190 collect_results root INFO     collect_results start
2024-01-21 15:35:23,703 collect_results root INFO     ('TestCase-2', 3)
2024-01-21 15:35:27,710 collect_results root INFO     ('TestCase-3', 4)
2024-01-21 15:35:27,710 collect_results root INFO     collect_results done
```

而在debain12上(python3.11)运行结果更是奇怪，很多日志重复输出了很多遍，why？！
```
tao@Dell:~/python_test$ python test_logging.py
2024-01-21 15:33:05,181 MainProcess root INFO     main start
2024-01-21 15:33:05,184 collect_results root INFO     collect_results start
2024-01-21 15:33:05,186 TestCase-2 root INFO     TestCase-2, started
2024-01-21 15:33:05,184 collect_results root INFO     collect_results start
2024-01-21 15:33:05,186 TestCase-2 root INFO     TestCase-2, started
2024-01-21 15:33:10,187 TestCase-2 root INFO     Doing some work from TestCase-2
2024-01-21 15:33:10,187 TestCase-2 root INFO     Doing some work from TestCase-2
2024-01-21 15:33:10,189 collect_results root INFO     ('TestCase-2', 5)
2024-01-21 15:33:10,189 collect_results root INFO     ('TestCase-2', 5)
2024-01-21 15:33:10,194 TestCase-3 root INFO     TestCase-3, started
2024-01-21 15:33:10,194 TestCase-3 root INFO     TestCase-3, started
2024-01-21 15:33:10,194 TestCase-3 root INFO     TestCase-3, started
2024-01-21 15:33:13,196 TestCase-3 root INFO     Doing some work from TestCase-3
2024-01-21 15:33:13,196 TestCase-3 root INFO     Doing some work from TestCase-3
2024-01-21 15:33:13,196 TestCase-3 root INFO     Doing some work from TestCase-3
2024-01-21 15:33:13,692 collect_results root INFO     ('TestCase-3', 3)
2024-01-21 15:33:13,692 collect_results root INFO     ('TestCase-3', 3)
2024-01-21 15:33:13,694 collect_results root INFO     collect_results done
2024-01-21 15:33:13,694 collect_results root INFO     collect_results done
```

### 问题分析
猜测应该是主程序创建进程时，子进程携带有父进程的一些对象信息，所有修改上面脚本加一些debug输出
```python
import random
import time
import logging
import logging.handlers
import multiprocessing
from queue import Empty


def worker_configure(queue, name):
    h = logging.handlers.QueueHandler(queue)
    root = logging.getLogger()
    root.addHandler(h)
    root.setLevel(logging.DEBUG)
    logging.info(f"call from {name}: w c: {root.handlers!r}")
    return root


class TestCase(multiprocessing.Process):
    def __init__(self, queue, r_q):
        super().__init__()
        worker_configure(queue, "TestCase")
        self.r_q = r_q
        name = multiprocessing.current_process().name
        logging.info(f"{name} init done")

    def run(self):
        name = multiprocessing.current_process().name
        print(f"{name} logging handlers: {logging.getLogger().handlers!r}")
        logging.info('%s, started', name)
        print(f"{name} logging handlers after call: {logging.getLogger().handlers!r}")
        ss = random.randint(3, 5)
        time.sleep(ss)
        logging.warning('Doing some work from %s', name)
        self.r_q.put((name, ss))


def listener_configure(queue):
    h = logging.StreamHandler()
    f = logging.Formatter('%(processName)-18s%(process)-7d%(levelname)-8s %(message)s')
    h.setFormatter(f)
    root = logging.getLogger()
    root.addHandler(h)
    root.setLevel(logging.DEBUG)

    listener = logging.handlers.QueueListener(queue, h)
    listener.start()
    return listener


def collect_results(queue, r_queue):
    worker_configure(queue, "collect_r")
    logging.info("collect_results start")
    while True:
        try:
            tuple_info = r_queue.get(block=False)
        except Empty:
            # log.debug("queue is empty")
            time.sleep(0.5)
            continue
        if not tuple_info:
            break
        logging.info(tuple_info)
    logging.info("collect_results done")


def main():
    r_queue = multiprocessing.Queue()
    queue = multiprocessing.Queue(-1)
    print(f"before listen: {logging.getLogger().handlers!r}")
    listener = listener_configure(queue)
    logging.info(f"after listen: {logging.getLogger().handlers!r}")
    logging.info("main start")

    result_collect = multiprocessing.Process(
        target=collect_results,
        name="collect_results",
        args=(queue, r_queue)
    )
    result_collect.start()

    for i in range(2):
        wp = TestCase(queue, r_queue)
        wp.start()
        wp.join()

    r_queue.put(None)
    result_collect.join()
    logging.info("main nearly finifsh")

    queue.put_nowait(None)
    listener.stop()
    logging.info("main finish")
 

if __name__ == '__main__':
    main()

```

### windows系统
Windows系统上，debug的输出如下
```
PS C:\Users\tao> python .\desktop\logging_test.py
before listen: []
MainProcess       4908   INFO     after listen: [<StreamHandler <stderr> (NOTSET)>]
MainProcess       4908   INFO     main start
MainProcess       4908   INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       4908   INFO     MainProcess init done
MainProcess       4908   INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       4908   INFO     MainProcess init done
collect_results   16440  INFO     call from collect_r: w c: [<QueueHandler (NOTSET)>]
collect_results   16440  INFO     collect_results start
TestCase-2 logging handlers: []
TestCase-2 logging handlers after call: [<StreamHandler <stderr> (NOTSET)>]
WARNING:root:Doing some work from TestCase-2
MainProcess       4908   INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       4908   INFO     MainProcess init done
MainProcess       4908   INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       4908   INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       4908   INFO     MainProcess init done
MainProcess       4908   INFO     MainProcess init done
TestCase-3 logging handlers: []
TestCase-3 logging handlers after call: [<StreamHandler <stderr> (NOTSET)>]
collect_results   16440  INFO     ('TestCase-2', 4)
WARNING:root:Doing some work from TestCase-3
collect_results   16440  INFO     ('TestCase-3', 4)
collect_results   16440  INFO     collect_results done
MainProcess       4908   INFO     main nearly finifsh
MainProcess       4908   INFO     main nearly finifsh
MainProcess       4908   INFO     main nearly finifsh
MainProcess       4908   INFO     main finish
```
对上面的输出，进行一步步分析
1. before listen: []
    说明主程序在运行`listener_configure`函数进行logging配置前，root logger的handlers列表里没有任何handler, 这步ok
2. MainProcess       4908   INFO     after listen: [<StreamHandler <stderr> (NOTSET)>]
    运行`listener_configure`函数配置后，主程序root logger有一个`StreamHandler`, **它没有设置level，所以会使用与其关联的root logger的日志等级**，即`logging.DEBUG`。这一步也ok
3. MainProcess       4908   INFO     main start ；这步ok
4. MainProcess       4908   INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
    这是在TestCase里`__init__`方法里调用`worker_configure`产生的输出
    这个输出就差不多发现问题了，本应该出现在子进程TestCase中的输出，出现在了主进程里，导致主进程handler列表额外多了一个`QueueHandler`对象，这也是出现后面三行输出的原因。
    这一步说明：**继承自`multiprocessing.Process`的TestCase子类创建新的进程时，`__init__`初始化方法是在父进程中执行的**
5. collect_results   16440  INFO     call from collect_r: w c: [<QueueHandler (NOTSET)>]
    这是collect_results进程的输出，经过`worker_configure`配置后，collect_results进程有只有一个`QueueHandler`，这步ok，其下面的第10行输出也ok
6. 接下来第11-13行是TestCase进程的输出：
    TestCase-2 logging handlers: []
    TestCase-2 logging handlers after call: [<StreamHandler <stderr> (NOTSET)>]
    WARNING:root:Doing some work from TestCase-2
    其中前两行是`logging.info('%s, started', name)`语句(由于日志等级低于WARNNING所以没有被输出)前后两个`print`语句的输出:
    - 在TestCase-2进程第一次调用logging函数前，`handlers`列表为空
    - 在第一次调用`logging.info`函数后，root logger的`handlers`列表包含一个`StreamHandler`
    上面第三行log是`logging.warning('Doing some work from %s', name)`语句的输出
    > 摘自[官方文档](https://docs.python.org/3/howto/logging.html "logging HOWTO")：
    > The `INFO` message doesn’t appear because the default level is `WARNING`. 
    > The call to `basicConfig()` should come before any calls to `debug()`, `info()`, etc. Otherwise, those functions will call `basicConfig()` for you with the default options. As it’s intended as a one-off simple configuration facility, only the first call will actually do anything: subsequent calls are effectively no-ops.
    通过这三个输出证明，**windows平台multiprocessing创建的子进程并没有继承父进程的环境，而是启动新的解释器环境**，windows上创建新进程的默认方法为**spawn**：
    ```python
    >>> import multiprocessing
    >>> multiprocessing.get_start_method
    <bound method DefaultContext.get_start_method of <multiprocessing.context.DefaultContext object at 0x000001B5AEABA300>>
    >>> multiprocessing.get_start_method()
    'spawn'
    >>>
    ```
7. 后面每次创建新的TestCase子进程时，都会使父进程的root logger的handler(`QueueHandler`)数量加1

经过上面分析，在windows上，脚本日志对不上的原因基本上已经很明了了：
1. 继承自`multiprocessing.Process`的TestCase子类创建新的进程时，`__init__`初始化方法是在父进程中执行的
2. Windows平台`multiprocessing`创建的子进程并没有继承父进程的环境，而是启动新的解释器环境

### linux系统
debain系统上，debug的log多了很多，输出如下:
```
tao@Dell:~/python_test$ python logging_test.py
before listen: []
MainProcess       210618 INFO     after listen: [<StreamHandler <stderr> (NOTSET)>]
MainProcess       210618 INFO     main start
MainProcess       210618 INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
collect_results   210620 INFO     call from collect_r: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       210618 INFO     MainProcess init done
MainProcess       210618 INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
collect_results   210620 INFO     collect_results start
collect_results   210620 INFO     call from collect_r: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       210618 INFO     MainProcess init done
collect_results   210620 INFO     collect_results start
TestCase-2 logging handlers: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
TestCase-2        210623 INFO     TestCase-2, started
TestCase-2 logging handlers after call: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>]
TestCase-2        210623 INFO     TestCase-2, started
TestCase-2        210623 WARNING  Doing some work from TestCase-2
TestCase-2        210623 WARNING  Doing some work from TestCase-2
MainProcess       210618 INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       210618 INFO     MainProcess init done
MainProcess       210618 INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       210618 INFO     call from TestCase: w c: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>, <QueueHandler (NOTSET)>]
MainProcess       210618 INFO     MainProcess init done
MainProcess       210618 INFO     MainProcess init done
TestCase-3 logging handlers: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>, <QueueHandler (NOTSET)>]
TestCase-3        210626 INFO     TestCase-3, started
TestCase-3 logging handlers after call: [<StreamHandler <stderr> (NOTSET)>, <QueueHandler (NOTSET)>, <QueueHandler (NOTSET)>]
TestCase-3        210626 INFO     TestCase-3, started
TestCase-3        210626 INFO     TestCase-3, started
collect_results   210620 INFO     ('TestCase-2', 4)
collect_results   210620 INFO     ('TestCase-2', 4)
TestCase-3        210626 WARNING  Doing some work from TestCase-3
TestCase-3        210626 WARNING  Doing some work from TestCase-3
TestCase-3        210626 WARNING  Doing some work from TestCase-3
collect_results   210620 INFO     ('TestCase-3', 3)
collect_results   210620 INFO     collect_results done
collect_results   210620 INFO     ('TestCase-3', 3)
collect_results   210620 INFO     collect_results done
MainProcess       210618 INFO     main nearly finifsh
MainProcess       210618 INFO     main nearly finifsh
MainProcess       210618 INFO     main nearly finifsh
MainProcess       210618 INFO     main finish
```
分析过程和windows一样，这里只说明关键点：
1. 第5行的输出说明，linux平台上multiprocessing模块在创建进程时，`Process`子类的`__init__`方法也是在父进程执行，这点和windows的执行过程一样；即，**不论windows还是linux，`Process`子类的`__init__`方法在创建多进程时都是在父进程执行。**
2. 第6行的collect_result进程调用`worker_configure`函数产生的输出显示，在linux系统collect_result进程比windows系统多了一个`StreamHandler`对象(继承自父进程)。这说明：**linux系统上multiprocessing模块(通过默认的`fork`调用)创建的新进程会继承父进程的环境**
3. 后面每次创建新的TestCase子进程时，都会使父进程的root logger的handler(`QueueHandler`)数量加1

经过上面分析说明，在linux平台上当multiprocessing通过`fork`系统调用创建进程时，子进程会复制父进程的环境，所以才导致linux上的输出比windows上多了很多

## 优化后的脚本
通过上面的分析，优化点只要在以下两个方面：
- 对于以`multiprocessing.Process`子类的方式创建子进程，需要将`worker_configure`函数的调用从`__init__`方法移动到`run`方法或其它方法。
- 如果在linux系统上运行，则需要在子进程开始时移除logging模块从父进程继承过来的`handler`
优化后的脚本如下：
```python
import random
import time
import logging
import logging.handlers
import multiprocessing
from queue import Empty


def worker_configure(queue):
    logging.getLogger().handlers = list()
    h = logging.handlers.QueueHandler(queue)
    root = logging.getLogger()
    root.addHandler(h)
    root.setLevel(logging.DEBUG)
    return root


class TestCase(multiprocessing.Process):
    def __init__(self, queue, r_q):
        super().__init__()
        self.queue = queue
        self.r_q = r_q

    def run(self):
        worker_configure(self.queue)
        name = multiprocessing.current_process().name
        logging.info('%s, started', name)
        ss = random.randint(3, 5)
        time.sleep(ss)
        logging.warning('Doing some work from %s', name)
        self.r_q.put((name, ss))


def listener_configure(queue):
    h = logging.StreamHandler()
    f = logging.Formatter(
        '%(processName)-18s%(process)-7d%(levelname)-8s %(message)s'
    )
    h.setFormatter(f)
    root = logging.getLogger()
    root.addHandler(h)
    root.setLevel(logging.DEBUG)

    listener = logging.handlers.QueueListener(queue, h)
    listener.start()
    return listener


def collect_results(queue, r_queue):
    worker_configure(queue)
    logging.info("collect_results start")
    while True:
        try:
            tuple_info = r_queue.get(block=False)
        except Empty:
            # log.debug("queue is empty")
            time.sleep(0.5)
            continue
        if not tuple_info:
            break
        logging.info(tuple_info)
    logging.info("collect_results done")


def main():
    r_queue = multiprocessing.Queue()
    queue = multiprocessing.Queue(-1)
    listener = listener_configure(queue)
    logging.info("main start")

    result_collect = multiprocessing.Process(
        target=collect_results,
        name="collect_results",
        args=(queue, r_queue)
    )
    result_collect.start()

    for i in range(2):
        wp = TestCase(queue, r_queue)
        wp.start()
        wp.join()

    r_queue.put(None)
    result_collect.join()
    logging.info("main nearly finifsh")

    queue.put_nowait(None)
    listener.stop()
    logging.info("main finish")
 

if __name__ == '__main__':
    main()

```

运行上面优化后的脚本，不论是在linux系统还是windows系统，都会有一致的log输出！
Windows输出如下：
```
PS C:\Users\tao> python .\desktop\logging_test.py
MainProcess       6296   INFO     main start
collect_results   7180   INFO     collect_results start
TestCase-2        5584   INFO     TestCase-2, started
TestCase-2        5584   WARNING  Doing some work from TestCase-2
TestCase-3        17100  INFO     TestCase-3, started
collect_results   7180   INFO     ('TestCase-2', 5)
TestCase-3        17100  WARNING  Doing some work from TestCase-3
collect_results   7180   INFO     ('TestCase-3', 3)
collect_results   7180   INFO     collect_results done
MainProcess       6296   INFO     main nearly finifsh
MainProcess       6296   INFO     main finish
```

Linux输出如下：
```
tao@Dell:~/python_test$ python logging_test.py
MainProcess       213851 INFO     main start
collect_results   213853 INFO     collect_results start
TestCase-2        213854 INFO     TestCase-2, started
TestCase-2        213854 WARNING  Doing some work from TestCase-2
collect_results   213853 INFO     ('TestCase-2', 4)
TestCase-3        213858 INFO     TestCase-3, started
TestCase-3        213858 WARNING  Doing some work from TestCase-3
collect_results   213853 INFO     ('TestCase-3', 3)
collect_results   213853 INFO     collect_results done
MainProcess       213851 INFO     main nearly finifsh
MainProcess       213851 INFO     main finish
```
