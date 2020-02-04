---
tags: python异步IO并发编程
---
子程序，或者称为函数，在所有语言中都是层级调用，比如A调用B，B在执行过程中又调用了C，C执行完毕返回，B执行完毕返回，最后是A执行完毕。
所以子程序调用是通过栈实现的，一个线程就是执行一个子程序。
子程序调用总是一个入口，一次返回，调用顺序是明确的。而协程的调用和子程序不同。
协程看上去也是子程序，但执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行。
所以不同点在于程序只有一个调用入口起始点，返回之后就结束了，而协程入口既可以是起始点，又可以从上一个返回点继续执行，也就是说协程之间可以通过yield方式转移执行权，对称（symmetric）、平级地调用对方，而不是像例程那样上下级调用关系。

### 相关语法

在python的函数中使用yield语句可以将函数变成一个生成器(generator)。生成器既可以返回数据，也可以接收数据。yield使得函数能够跳出和返回原始的执行流程

```python

def generator():
    print('step 1, line 2')
    a = yield 1
    print("a = %s" % a)
    print('step 2, line5')
    b = yield 2
    print(b)

g = generator()
m = g.send(None)             # 激活g, 运行至line3, 输出 'step 1, line2'，yeild返回1，m = 1
print("m = %d" % m)          # 输出 'm = 1'
n = g.send(1)              # 向g发送‘x’, g重新从line3开始运行，yeild接收‘x’，即a='x',
                             # 输出 'a = x'
                             # 输出 'step 2, line5', yeild返回2 ，n = 2
print("n = %d" % n)

```

generator.send 函数使得函数推进到下一个yield的执行位置。上文中的第一句g.send执行到了yield 1 处，返回值1。第二句g.send(1) 使得generator 从上一个yield的位置运行，并且准备推进到下一个yield函数中，同时send函数可以带一个参数进入到上一个yield的执行位置。因此g.send(1) 将 1 带入了generator 中，并且赋值给了1。

第一个next(core)或core.send(None)被称为prime，用来让协程向前运行到第一个yield表达式，准备好从后续的core.send(value)调用中接收值

### 应用场景

假设我们需要爬去100次百度的网页。我们常用方法是使用epoll事件模型探测活跃的socket,以线程的方式消费活跃的socket(即使python有GIL锁，只要是IO密集型任务，多线程依然能够发挥优势)。然后借助yield，我们可以实现自己的协程，通过异步驱动的方式在单线程下也能够充分利用gpu。同时减少线程创建与调度的开销。

假设我们现在有多个socket连接，我们希望socket准备好了之后进行读或写操作。在处理socket的时候，一旦阻塞，我们将退出对当前socket的处理。

一个爬取网页的代码应该是：
```python
def fetch():
    sock = socket.socket()
    sock.setblocking(False)
    try:
        sock.connect(('www.baidu.com', 80))  # 不会阻塞
    except BlockingIOError:
        pass

    sel.register(sock.fileno(), EVENT_WRITE)
    sel.unregister(sock.fileno())
    req = b'GET / HTTP/1.0\r\n Host:www.baidu.com\r\n\r\n'
    sock.send(req)

    sel.register(sock, EVENT_READ)
    sel.unregister(sock.fileno())pyt
    data = sock.recv(4096)  # Should be ready
```
通常来说，socket如果是阻塞式的，读和写操作都是阻塞操作。为了是cpu不浪费在读写操作上，使用epoll多路复用模型，当检测到socket 可写时写入数据，当socket 可读时读入数据。

借助python 的yield关键字保留上下文的能力，我们可以将读写操作合并在同一协程中处理。即当等待socket 可读或者可写的时候退出当前协程，当socket准备就绪后返回当前协程。

为了达到这个目的，我们需要在原来的代码中使用yield关键字增加进入和退出点。我们首先定一个Feature类，它表示将来的socket的可读和可写事件。当socket可读或者可写的时候，它应该能够恢复协程的运行。
```python
class Furture():
    def __init__(self):
        self.coro = None #需要恢复的协程

    def add_coro(self, coro):
        self.coro = coro

    def resume(self):
        global times
        try:
            self.coro.send(None) #恢复协程
        except StopIteration:
            times = times - 1 #times是启动的携程的个数，一旦出现stopIteration er ror 表示当前协程没有返回点，已经处理完毕
```
fetch函数(这里就是我们的协程)修改为
```python
    def fetch():
        sock = socket.socket()
        sock.setblocking(False)
        try:
            sock.connect(('www.baidu.com', 80))  # 不会阻塞
        except BlockingIOError:
            pass
    
        f = Furture()
    
        sel.register(sock.fileno(), EVENT_WRITE, f.resume)
    
        # 等待sock可写后返回这里
        yield f
    
        sel.unregister(sock.fileno())
        req = b'GET / HTTP/1.0\r\n Host:www.baidu.com\r\n\r\n'
        sock.send(req)
    
        sel.register(sock, EVENT_READ, f.resume)
    
        # 等待sock可读后返回这里
        yield f
        sel.unregister(sock.fileno())
        data = sock.recv(4096)  # Should be read
```
除了定义恢复协程的类，我们还需要一个Task类驱动程序的运行。
```python
def Task():
  coro = fetch() #对爬取网页任务创建一个协程
  future = coro.send(None) #启动协程到第一个推出点,后面由future类驱动
  future.add_coro(coro)

定义loop函数，使用多路复用，一旦返现socket可读或者可写则使用feature驱动携程运行

def loop():
    while times:
        events = sel.select()  # 阻塞，有活动连接就返回活动连接列表
        for key, mask in events:
            callback = key.data  # accept
            callback()
            if times <= 0:
                return
```
Main 程序
```python
import socket
import time
from selectors import DefaultSelector, EVENT_READ, EVENT_WRITE


sel = DefaultSelector()
times = 10
if __name__ == '__main__':
    t1 = time.time()
    for i in range(times):
        Task()
    loop()
    t2 = time.time()
    print('耗时:', t2 - t1)
    # 耗时: 0.5629799365997314
```
实际上是用epoll或者协程（本质上也是epoll），跟线程池比起来其实没有什么明显的性能优势，对于比较新的内核，当网卡IO量大的时候，甚至多线程还会略微快一些，这跟新的多队列特性有些关系，使用多线程的时候你会发现python进程的CPU使用率可以达到200%甚至更高，因为recv, send这样的系统调用在内核到用户空间复制数据的过程不需要加GIL。协程的优势还是在单socket处理的开销很小这一点上。

线程切换开销这个，理论上的确是存在，但是Python调度器也有CPU开销，而且因为Python比较慢，实际上反而是不如多线程系统调度的。只有用C/C++实现调度器的时候有可能有一定性能优势（gevent就是C实现的），而且实际上也并不大

协程主要并不是为了快，而是为了并发量大，因为一个线程同时只能以阻塞方式处理一个socket，如果连接数到了10k甚至100k的量级的时候，10k个线程基本上操作系统就要崩了。而在用户空间中进行调度的协程没有这个问题

参考:https://zhuanlan.zhihu.com/p/27258289 （灵剑回答）
