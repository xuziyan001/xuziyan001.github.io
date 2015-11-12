---
layout: post
title:  "About Multiplexing IO"
date:   2015-11-07 15:00:00
categories: Lejent update
---
最近实在是忙得不行，都没有时间学一些新的东西，也没空做demo，只能把之前总结的东西捞出来再复习一下，这次我们介绍的是Linux下IO复用的接口以及对应的python模块。
###有关Linux的IO复用
Linux提供的实现IO复用的接口不外乎select，poll和epoll三个，从大体上来说，select和poll本质上区别不大，都是同步轮询所有的连接；而epoll每次调用都只会遍历活跃的描述符，在多连接少活跃的情况下性能相当的棒，著名的nginx底层就应该是通过epoll对IO进行的复用，有时间要深入了解一下nginx这玩意。我们先逐个介绍这些接口吧：  
####select
{% highlight c %}  
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);  
{% endhighlight %}
select调用定义了5个参数：  
* nfds: 文件描述符集中要被检测的数目，至少要比待检测的最大文件描述符大1.  
* readfds: 需要被读监测的文件描述符.  
* writefds: 需要被写监测的文件描述符.  
* exceptfds: 可能出现异常情况的文件描述符.  
* timeout: 微秒级定时器，这个参数至关重要，它可以使select处于三种状态，第一，若将NULL以形参传入，即不传入时间结构，就是将select置于阻塞状态，一定等到监视文件描述符集合中某个文件描述符发生变化为止；第二，若将时间值设为0秒0毫秒，就变成一个纯粹的非阻塞函数，不管文件描述符是否有变化，都立刻返回继续执行，文件无变化返回0，有变化返回一个正值；第三，timeout的值大于0，这就是等待的超时时间，即select在timeout时间内阻塞，超时时间之内有事件到来就返回了，否则在超时后不管怎样一定返回。 
一般情况下，函数的返回值定义如下：  
负值：select错误。  
正值：某些文件可读写或出错。  
0：等待超时，没有可读写或错误的文件。  
####poll
{% highlight c %}  
int poll(struct pollfd *fds, nfds_t nfds, int timeout);  
struct pollfd {int fd; /* 文件描述符 */short events; /* 等待的事件 */short revents; /* 实际发生了的事件 */};  {% endhighlight %}  
poll定义了自己的文件描述符结构体，3个参数的定义如下：  
* fds: 一个需要监测的pollfd结构体数组。  
* nfds： 指定第一个参数数组的元素个数。  
* timeout： 同select的timeout。  
其中events是针对这个文件描述符感兴趣事件的掩码，revents是这个文件描述符上实际发生的事件的掩码。poll采用统一的pollfd替代了select中三个不同事件的fd_set数组，并且取消了select最多监控1024个文件描述符的限制，但它的调用依旧是同步的，与select函数调用了同样的底层函数fop->poll，对所有的描述符进行轮询。  
####epoll
{% highlight c %}  
int epoll_create(int size);int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);typedef union epoll_data {  
    void *ptr;  
    int fd;  
    __uint32_t u32;  
    __uint64_t u64;  
} epoll_data_t;  
 //感兴趣的事件和被触发的事件  
struct epoll_event {  
    __uint32_t events; /* Epoll events */  
    epoll_data_t data; /* User data variable */  
};  
{% endhighlight %}  
epoll本身也是一个文件，调用epoll_create将返回一个fd，使用完之后需要close对文件描述符进行释放；  
epoll_ctl对fd进行op操作（EPOLL_CTL_ADD，EPOLL_CTL_MOD，EPOLL_CTL_DEL），event代表具体监听的事件；epoll_wait收集在epoll监控的事件中已经发送的事件，参数events是分配好的epoll_event结构体数组，epoll将会把发生的事件赋值到events数组中。  
epoll_wait每次调用只会遍历活跃的描述符，因为一旦epoll监控的文件状态发生改变，就会产生一个回调，将发生改变的fd以及相应的事件会写到注册时的ev中，所以在活跃连接很少的情况下，epoll的性能会相当高。  
### greenlet
[greenlet](https://greenlet.readthedocs.org/en/latest/)是stackless Python的附加产物，它作为一个c扩展模块提供给未修改的解释器，它不存在调度机制，所以使用它的时候需要自己构造调度器，其内部使用switch原语手动在greenlet之间进行切换。  
{% highlight python %}
def test1(x,y):    z=gr2.switch(x+y)    print zdef test2(u):    print u    gr1.switch(42)
gr1=greenlet(test1)gr2=greenlet(test2)gr1.switch("hello"," world")
{% endhighlight %}
如上，main协程是gr1和gr2的父协程，程序首先由main切换到gr1，传递参数'hello'以及'world'，gr1将'hello world'组装并将其作为参数切换到gr2，gr2将'hello world'打印出来，并切换到gr1，将42作为返回值发送到gr1的z中，此时gr2因为执行完毕而退出，gr1打印42之后也完成了使命而退出，程序返回main协程，以下是switch的正确调用方式：  
{% highlight python %}
g.switch(obj=None or *args)
{% endhighlight %}  
greenlet实例可以在实例之间传递参数或者是消息，所以在使用的时候需要对消息传递顺序以及协程栈运行暂停的地方要规划合理。  
greenlet类支持以下几个操作：
{% highlight python %}
greenlet(run=None,parent=None) #创建一个greenlet对象，而不执行。run是执行回调，而parent是父greenlet，缺省是当前greenlet。greenlet.getcurrent() #返回当前greenlet，也就是谁在调用这个函数。greenlet.GreenletExit #这个特定的异常不会波及到父greenlet，它用于干掉一个greenlet。
{% endhighlight %}
Greenlet实例支持如下操作：
{% highlight python %}
g.switch(obj=None or *args) #切换执行点到greenlet g，同上。g.run #调用可执行的g，并启动。在g启动后，这个属性就不再存在了。g.parent #greenlet的父。这是可写的，但是不允许创建循环的父关系。g.gr_frame #当前顶级帧，或者None。g.dead #判断是否已经死掉了bool(g) #如果g是活跃的则返回True，在尚未启动或者结束后返回False。g.throw([typ,[val,[tb]]]) #切换执行点到greenlet g，但是立即抛出指定的异常到g。如果没有提供参数，异常缺省就是 greenlet.GreenletExit 。根据异常波及规则，有如上面描述的。注意调用这个方法等同于如下:def raiser():    raise typ,val,tbg_raiser=greenlet(raiser,parent=g)g_raiser.switch()
{% endhighlight %}  
任何尝试切换到死掉的greenlet的行为都会切换到死掉greenlet的父greenlet或父的父，一直到main。且任何一个未捕捉的非GreenletExit异常都会杀死当前Greenlet并回到主函数。  
Greenlet可以和python线程一同使用，每个线程包含一个独立的main greenlet，并拥有自己的greenlet树，不同线程间的greenlet不能切换。当一个greenlet实例所有的引用都指向不到的时候，系统会在该greenlet中生成一个GreenletExit，所以在协程中我们需要使用try…finally来清理协程中的资源。  
###gevent
[gevent](http://www.gevent.org/)和greenlet类似，通过gevent.sleep函数对协程进行切换，也可以使用通过gevent包装的Greenlet类来实现并发，这里有点诡异，因为使用gevent.spawn和gevent.Greenlet.spawn并没有太大的差别，为什么不保留一套机制。  
Gevent.Timeout类可以对协程处理时间进行约束，将其加入with控制块内：  
{% highlight python %}
class TooLong(Exception):    passwith Timeout(time_to_wait, TooLong):    gevent.sleep(10)  
{% endhighlight %}  
当协程处理超时的时候，就会抛出相应的异常。Timeout大概有一万种用法，这个就只能依赖编程的习惯了。Monkey补丁可以将python内置的多路复用函数进行替换，继而支持协程的行为：  
{% highlight python %}
from gevent import monkeymonkey.patch_socket() #对socket.socket进行替换monkey.patch_select() #对select.select进行替换monket.patch_all() #对包括socket，ssl，threading，select模块进行替换  
{% endhighlight %} 
event是greenlet之间的异步通信方式：  gevent.event.Event于多线程编程中的事件概念是一致的。  Gevent.event.AsyncResult可以在事件中设置一个参数，等待者可以通过get方法同步得到这个值。  Gevent.queue.Queue类似于共享队列，没什么好说的。  Gevent.pool.Group提供了对greenlet的共同管理和调度，Group实例提供map调用简化程序的编写：  
{% highlight python %}
igroup = Group()for i in igroup.imap_unordered(intensive, xrange(3)):    print(i)  
{% endhighlight %}  
gevent.pool.Pool为限制并发数量而设计，也可以通过map同时起多个协程。  
信号量和锁也没啥好说的，在gevent.coros包内。  gevent.local.local提供了协程内的局部变量，这个估计和线程变量类似，出了这个协程的控制范围，这个变量就访问不到了。在线程变量中，线程变量可能没有释放会造成线程复用是出错，所以协程变量也要注意在协程结束之后释放协程局部变量。  
{% highlight python %}  
＃一个flask风格的http会话from gevent.local import localfrom werkzeug.local import LocalProxyfrom werkzeug.wrappers import Requestfrom contextlib import contextmanagerfrom gevent.wsgi import WSGIServer_requests = local()request = LocalProxy(lambda: _requests.request)@contextmanagerdef sessionmanager(environ):    _requests.request = Request(environ)    yield    _requests.request = None #注意清空变量def logic():    return "Hello " + request.remote_addrdef application(environ, start_response):    status = '200 OK'    with sessionmanager(environ):        body = logic()    headers = [        ('Content-Type', 'text/html')    ]    start_response(status, headers)    return [body]WSGIServer(('', 8000), application).serve_forever()
{% endhighlight %}  
Gevent.subprocess支持协作式等待的子进程，这也涉及到了multiprocessing协同gevent一起使用的问题，官方是这么解释的：  
> 很多人也想将gevent和multiprocessing一起使用。最明显的挑战之一，就是multiprocessing提供的进程间通信默认不是协作式的。由于基于 multiprocessing.Connection的对象(例如Pipe)暴露了它们下面的 文件描述符(file descriptor)，gevent.socket.wait_read和wait_write 可以用来在直接读写之前协作式的等待ready-to-read/ready-to-write事件。 

不是很理解，暴露了文件描述符又能怎么样呢，是怕多协程访问同一个文件而导致竞态么？  
一个可以用于生产的实例，用子类化的Greenlet类来模拟Erlang的Actor模型：  
{% highlight python %}  
import geventfrom gevent.queue import Queueclass Actor(gevent.Greenlet):    def __init__(self):        self.inbox = Queue()        Greenlet.__init__(self)    def receive(self, message):        """        Define in your subclass.        """        raise NotImplemented()    def _run(self):        self.running = True    	while self.running:            message = self.inbox.get()            self.receive(message)
{% endhighlight %}  
###Stackless Python
[Stackless Python](http://www.stackless.com/)是python语言的一个增强版本，它可以为python带来一种微进程（tasklet）的扩展，之前所介绍的greenlet就是tasklet的一个附加产物，而gevent提供的是对greenlet调度的包装以及epoll类型的IO复用。在不使用stackless的情况下，stackless python表现得和python一样。  
tasklet是stackless的基本构成单元，每当建立一个tasklet，它就会在调度器中排队，直到调用stackless.run方法来触发：  
{% highlight python %}  
>>> import stackless>>> def print_x(x):...     print 1,x...     print 2,x>>> stackless.tasklet(print_x)('one')<stackless.tasklet object at 0x00A45870>>>> stackless.tasklet(print_x)('two')<stackless.tasklet object at 0x00A45A30>>>> stackless.run()1 one2 one1 two2 two
{% endhighlight %}  

schedule方法可以显式的在各tasklet间进行切换：  
{% highlight python %}  
>>> import stackless>>>>>> def print_x(x):...     print 1,x...     stackless.schedule()...     print 2,x...>>> stackless.tasklet(print_x)('one')<stackless.tasklet object at 0x00A45870>>>> stackless.tasklet(print_x)('two')<stackless.tasklet object at 0x00A45A30>>>> stackless.run()1 one1 two2 one2 two
{% endhighlight %}  

channel提供tasklet间的通信，接收消息的tasklet调用channel.receive的时候会被阻塞，一旦有其他tasklet向该channel发送消息后，发送消息的tasklet则被转移到调度列表的尾部，如同调用schedule一样，同时监听消息的tasklet立即恢复执行。  
{% highlight python %} 
>>> import stackless>>> channel = stackless.channel()>>> def receiving_tasklet():...     print "Receiving tasklet started"...     print channel.receive()...     print "Receiving tasklet finished">>> def sending_tasklet():...     print "Sending tasklet started"...     channel.send("send from sending_tasklet")...     print "sending tasklet finished">>> def another_tasklet():...     print "Just another tasklet in the scheduler">>> stackless.tasklet(receiving_tasklet)()<stackless.tasklet object at 0x00A45B30>>>> stackless.tasklet(sending_tasklet)()<stackless.tasklet object at 0x00A45B70>>>> stackless.tasklet(another_tasklet)()<stackless.tasklet object at 0x00A45BF0>>>> stackless.run()Receiving tasklet started Sending tasklet startedsend from sending_taskletReceiving tasklet finishedJust another tasklet in the schedulersending tasklet finished 
{% endhighlight %}  

tasklet的调度顺序为：  1.	receiving打印receiving tasklet started然后阻塞在receive；  2.	sending打印sending tasklet started然后发送消息到channel，然后被塞入调度尾部；  3.	receiving接收到消息，则插队到another前面，打印channel中的send from sending tasklet，然后打印receiving tasklet finished结束该tasklet；  4.	another打印just another tasklet in the scheduler，然后结束；  5.	sending最后从send调用处恢复并打印sending tasklet finished，结束。  以上就是stackless的全部内容，简直随意到没有朋友。下面来介绍一些协程的基本概念，先来个小栗子：  
{% highlight python %} 
def ping():    print "PING"    ping()ping()
{% endhighlight %}  
大家都知道这个函数会造成栈溢出，因为递归函数没有结束条件，栈帧一直增长而没有释放，这就是我们传统stack模型程序调用，而如果用stackless方式，通过channel来协调，如下：  
{% highlight python %}  
import stacklessping_channel = stackless.channel()def ping():    while ping_channel.receive(): #在此阻塞        print "PING"        ping_channel.send("from ping")stackless.tasklet(ping)()ping_channel.send('start')stackless.run(){% endhighlight %}  
这里没有对自己的递归调用，所以永远也不会有栈溢出的问题，尽管这个程序也是不停的打印pingpingpingping，但是它可以打印一年也不会挂。  
最后帖一段代码吧，代码模拟了一个小世界，一群机器人在里面瞎飞，碰到墙壁就转弯。。。  
{% highlight python %}  
"""
@author: vic
@describe:
    simple demo using stackless to communicate with tasklets
    regard every thing as an actor
    the World instance should be singleton but make no difference
@date: Oct 23, 2015
"""
import pygame
import pygame.locals
import os
import sys
import stackless
import math
import copy
import time
class Actor(object):
    def __init__(self):
        self.channel = stackless.channel()
        self.process_msg = self.default_message_action
        stackless.tasklet(self.process)()
    def process(self):
        while True:
            self.process_msg(self.channel.receive())
    def default_message_action(self, args):
        print args
class World(Actor):
    def __init__(self):
        super(World, self).__init__()
        self.registered = {}
        stackless.tasklet(self.send_state_to_actors)()
        print "world init"
    def test_for_collision(self, x, y):
        if x < 0 or x > 496 or y < 0 or y > 496:
            return True
        return False
    def send_state_to_actors(self):
        while True:
            actors = copy.copy(self.registered)
            for actor in actors:
                info = self.registered[actor]
                if info[1] != (-1, -1):
                    vx, vy = (math.sin(math.radians(info[2])) * info[3], \
                              math.cos(math.radians(info[2])) * info[3])
                    x, y = info[1]
                    x += vx
                    y -= vy
                    if self.test_for_collision(x, y):
                        actor.send((self.channel, "COLLISION"))
                    else:
                        self.registered[actor] = tuple([info[0], (x, y), info[2], info[3]])
            world_state = [self.channel, "WORLD_STATE"]
            for actor in actors:
                info = self.registered[actor]
                if info[1] != (-1, -1):
                    world_state.append((actor, info))
            message = tuple(world_state)
            print 'actors: %s' % actors
            for actor in actors:
                actor.send(message)
            stackless.schedule()
    def default_message_action(self, args):
        sent_from, msg, msg_args = args[0], args[1], args[2:]
        print "world recv msg %s" % str(args)
        if msg == "JOIN":
            print "ADDING", msg_args
            self.registered[sent_from] = msg_args
        elif msg == "UPDATE_VECTOR":
            self.registered[sent_from] = tuple([self.registered[sent_from][0], \
                    self.registered[sent_from][1], msg_args[0], msg_args[1]])
        else:
            print "UNKNOWN MSG", args
class Display(Actor):
    def __init__(self, world = None):
        super(Display, self).__init__()
        self.world = world
        self.icons = {}
        pygame.init()
        window = pygame.display.set_mode((660, 648))
        pygame.display.set_caption("Demo")
        join_msg = (self.channel, "JOIN", self.__class__.__name__, (-1, -1))
        print "Display send msg to world"
        self.world.send(join_msg)
    def default_message_action(self, args):
        sent_from, msg, msg_args = args[0], args[1], args[2:]
        if msg == "WORLD_STATE":
            print "recv WORLD_STATE, start to update display"
            self.update_display(msg_args)
        else:
            print "UNKNOWN MSG", args
    def get_icon(self, icon_name):
        if self.icons.has_key(icon_name):
            return self.icons[icon_name]
        else:
            icon_file = os.path.join("data", "%s.bmp" % icon_name)
            surface = pygame.image.load(icon_file)
            surface.set_colorkey((0xf3, 0x0a, 0x0a))
            self.icons[icon_name] = surface
            return surface
    def update_display(self, actors):
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                sys.exit()
        screen = pygame.display.get_surface()
        bg = pygame.Surface(screen.get_size())
        bg = bg.convert()
        bg.fill((200, 200, 200))
        screen.blit(bg, (0, 0))
        for item in actors:
            screen.blit(pygame.transform.rotate(self.get_icon(item[1][0]), -item[1][2]), item[1][1])
        pygame.display.flip()
class Robot(Actor):
    def __init__(self, location = (0, 0), angle = 135, velocity = 1, world = None):
        super(Robot, self).__init__()
        self.location = location
        self.angle = angle
        self.velocity = velocity
        self.world = world
        join_msg = (self.channel, "JOIN", self.__class__.__name__, \
                self.location, self.angle, self.velocity)
        print "Robot %s send msg to world" % self.__class__.__name__
        self.world.send(join_msg)
    def default_message_action(self, args):
        sent_from, msg, msg_args = args[0], args[1], args[2:]
        if msg == "WORLD_STATE":
            self.location = (self.location[0]+self.velocity, self.location[1]+self.velocity)
            self.angle += 0.5
            if self.angle >= 360:
                self.angle -= 360
            update_msg = (self.channel, "UPDATE_VECTOR", self.angle, self.velocity)
            self.world.send(update_msg)
        elif msg == "COLLISION":
            self.angle += 73
            if self.angle >= 360:
                self.angle -= 360
        else:
            print "UNKNOWN MSG", args
if __name__ == "__main__":
    world_channel = World().channel
    Display(world=world_channel)
    print "display init"
    r1 = Robot(angle=135, velocity=5, world=world_channel)
    r2 = Robot((464, 0), angle=225, velocity=10, world=world_channel)
    r3 = Robot((464, 0), angle=0, velocity=100, world=world_channel)
    r4 = Robot((464, 4), angle=0, velocity=10, world=world_channel)
    r5 = Robot((464, 6), angle=0, velocity=20, world=world_channel)
    r6 = Robot((464, 6), angle=0, velocity=30, world=world_channel)
    r7 = Robot((464, 6), angle=0, velocity=40, world=world_channel)
    r8 = Robot((464, 6), angle=0, velocity=50, world=world_channel)
    r9 = Robot((464, 6), angle=0, velocity=60, world=world_channel)
    r10 = Robot((464, 6), angle=0, velocity=70, world=world_channel)
    r11 = Robot((464, 6), angle=0, velocity=80, world=world_channel)
    r12 = Robot((464, 6), angle=0, velocity=90, world=world_channel)
    stackless.run()                                                                                                               
{% endhighlight %}  


