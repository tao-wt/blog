---
title: python多进程编程中导致程序在退出时卡住的两种情况
date: 2024-2-15 22:20:52
index_img: /img/index-7.jpg
tags:
  - python
categories:
  - [python, multiprocessing]
author: tao-wt@qq.com
excerpt: 本文描述了在不正确使用multiprocessing模块的Queue和Pipe时，导致程序在退出时卡住的两种情况
---
## 使用管道时
原理：创建管道`conn1, conn2 = multiprocessing.Pipe()`，调用`conn1`和`conn2`的`recv()`或`recv_bytes()`方法时，如果对端已关闭**且**不存在数据项则会引发`EOFError`异常。所以，多进程环境中如果生产者/消费者没有/不再使用管道的某个端点就应该将其关闭，以防止程序在管道的`recv()`/`recv_bytes()`方法上挂起。
> 管道是由操作系统进行引用计数的, 必须在所有进程中关闭管道才能生成`EOFError`异常！

### 问题复现
以下简单的代码展示了在**不使用哨兵**的情况下，程序在退出时卡住的现象：
```python
import multiprocessing


def consumer(conn):
    print("consumer start")
    while True:
        obj = conn.recv()
        if obj is None:
            break
        print(f"obj: {obj!r} recived.")
    print("consumer done")


c1, c2 = multiprocessing.Pipe()
proc = multiprocessing.Process(target=consumer, args=(c2,))
proc.deamon = False
proc.start()
c1.send(123)

```
运行结果如下，程序在退出时卡住(不论进程`proc`的`deamon`属性是否为`False`):
![pipe error](/img/pipe_error.png)

### 规避方法
1. 使用哨兵，有几个消费者，就传入几个哨兵：`c1.send(None)`, 推荐。
2. 在进程中关闭不使用的管道，使进程产生`EOFError`异常，但需要额外的异常处理代码：
    修改后的脚本如下：
    ```python
    import multiprocessing


    def consumer(conn):
        print(globals())
        global c1
        c1.close()
        print("consumer start")
        while True:
            try:
                obj = conn.recv()
            except EOFError as e:
                print(f"{e!r}")
                break
            if obj is None:
                break
            print(f"obj: {obj!r} recived.")
        print("consumer done")


    c1, c2 = multiprocessing.Pipe()
    proc = multiprocessing.Process(target=consumer, args=(c2,))
    proc.deamon = False
    proc.start()
    c1.send(123)
    c1.close()

    ```
    运行结果如下：
    ![pipe ok](/img/pipe_ok.png)

## 使用队列时
原理：创建队列`queue = multiprocessing.Queue()`, 当队列使用完毕执行`queue.close()`关闭时，队列关闭不会在消费者中的`get()`方法上生成任何类型的信号或异常！

### 问题复现
以下简单的代码展示了在**不使用哨兵**的情况下，程序在退出时卡住的现象：
```python
import multiprocessing


def consumer(queue):
    print("consumer start")
    while True:
        obj = queue.get()
        if obj is None:
            break
        print(f"obj: {obj!r} recived.")
    print("consumer done")


queue = multiprocessing.Queue()
proc = multiprocessing.Process(target=consumer, args=(queue,))
proc.deamon = False
proc.start()
queue.put("123")
# queue.put(None)

```
运行结果和使用管道时一样，程序在退出时卡住(不论进程`proc`的`deamon`属性是否为`False`):
![queue error](/img/queue_error.png)

### 规避方法
使用哨兵，有几个消费者，就传入几个哨兵：`queue.put(None)`
