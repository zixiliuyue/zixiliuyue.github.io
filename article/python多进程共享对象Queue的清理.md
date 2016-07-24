#### python多进程共享对象Queue的清理
@(进程间通信)[消息队列|资源回收]
在做进程间数据共享的时候，发现python2.7自带的multiprocessing模块的Queue对象没有清理的功能，put进去的数据如果想清除掉释放资源要一个一个删除，速度非常慢。阅读**multiprocessing**模块源码，发现可以通过重写实现一种非常快的方法：

- **override_Queue.py ** 重写Queue对象
``` python
from Queue import Queue

class OverRideQueue(Queue):
    #父类是老式继承关系
    def __init__(self):
        Queue.__init__(self)

    def clear(self):
        #模仿取大小 先进行唯一锁操作
        self.mutex.acquire()
        self.queue.clear()
        self.mutex.release()
```
- **override_SyncManager.py **  重写对象注册
``` python
from multiprocessing.managers import SyncManager
from multiprocessing.managers import BaseManager
from override_Queue import OverRideQueue


class override_SyncManagers(SyncManager):
    '''
        继承自SyncManager，增加重写过的OverRideQueue
    '''
override_SyncManagers.register("OverRideQueue",OverRideQueue)
```
- **multiprocess_manager.py ** 重写manager对象,实现程序中调用
``` python
def override_Manager():
    from override_SyncManager import override_SyncManagers
    m = override_SyncManagers()
    m.start()
    return m
```
- **test_process.py ** 测试程序
``` python
import os
import multiprocessing
from override_SyncManager import override_SyncManagers
from override_Queue import OverRideQueue
import time
from multiprocess_manager import override_Manager

def show(queue):
    print "一子线程线程的pid %s"%os.getpid()
    print queue.qsize()
    x=time.time()
    queue.clear()
    y = time.time()
    print "删除时间%s" % str(float(y)-float(x))
    print "接收数据长度%s"%str(queue.qsize())


def additem(queue):
    print "二子线程的pid %s"%os.getpid()
    v=time.time()
    for item in range(100000):
        queue.put(item)
    p=time.time()
    print "插入时间%s"%str(float(p)-float(v))
    print "插入数据长度%s"%str(queue.qsize())


print "父线程的pid %s"%os.getpid()

sharequeue=override_Manager()
a=sharequeue.OverRideQueue()

aprocess=multiprocessing.Process(target=additem,args=(a,))
aprocess.start()
aprocess.join()

print "*"*60
time.sleep(2)

bprocess=multiprocessing.Process(target=show,args=(a,))
bprocess.start()
bprocess.join()
```
> 最终在公司电脑上测试发现，如果在进程一插入十万数据，共享给进程二，通过进程二调用重写的clear方法，删除只需要0.4秒。相比以前的共享数据回收方式，速度快了很多。