# Swoole进程结构

本篇文章会涉及到以下几个方面

- 进程的基本知识
- swoole进程结构
- 进程及其相对应的事件绑定
- 守护进程，信号和平滑重启


## 进程的基础知识

什么是进程，所谓进程其实就是操作系统中一个正在运行的程序，我们在一个终端当中，通过php，运行一个php文件，这个时候就相当于我们创建了一个进程，这个进程会在系统中驻存，申请属于它自己的内存空间系统资源并且运行相应的程序
   
对于一个进程来说，它的核心内容分为两个部分，一个是它的内存，这个内存是这进程创建之初从系统分配的，它所有创建的变量都会存储在这一片内存环境当中

一个是它的上下文环境我们知道进程是运行在操作系统的，那么对于程序来说，它的运行依赖操作系统分配给它的资源，操作系统的一些状态。
 
在操作系统中可以运行多个进程的，对于一个进程来说，它可以创建自己的子进程，那么当我们在一个进程中创建出若干个子进程的时候那么可以看到如图，子进程和父进程一样，拥有自己的内存空间和上下文环境

- fork子进程的时候，子进程回复制父进程的内存空间和上下文环境
- 修改某个子进程的内存空间，不回修改父进程或其他子进程中的内存空间
- swoole本身也是一个多进程的模型，它有多个worker进程和自己的master进程，多个worker进程中创建的变量是不能通用的

## Swoole进程结构

Swoole进程结构图

![WeChat820cb89db7fe8d13d4094f451fe04976.png](https://i.loli.net/2019/07/10/5d255d416a67f15358.png)

![WeChatb395e47629fba565fcbfd53bee7f3a31.png](https://i.loli.net/2019/07/10/5d2594b8164b995422.png)


- 1、Master进程：主进程
- 2、Manger进程：管理进程
- 3、Worker进程：工作进程
- 4、Task进程：异步任务工作进程


### Master进程

第一层，Master进程，这个是swoole的主进程,这个进程是用于处理swoole的核心事件驱动的，这个进程当中可以看到它拥有一个MainReactor[线程]以及若干个Reactor[线程]，swoole所有对于事件的监听都会在这些线程中实现，比如来自客户端的连接，信号处理等。

![WeChatbfe6d68aef2f9b8a7666ceaf0de09387.png](https://i.loli.net/2019/07/10/5d25943dd588536889.png)


- MainReactor(主线程)

主线程会负责监听server socket，如果有新的连接accept，主线程会评估每个Reactor线程的连接数量。将此连接分配给连接数最少的reactor线程，做一个负载均衡。

- Reactor线程组

Reactor线程负责维护客户端机器的TCP连接、处理网络IO、收发数据完全是异步非阻塞的模式。

swoole的主线程在Accept新的连接后，会将这个连接分配给一个固定的Reactor线程，在socket可读时读取数据，并进行协议解析，将请求投递到Worker进程。在socket可写时将数据发送给TCP客户端。

除了以上这些，如果我们配置了心跳或者是udp协议的话，还会启动以下这两个线程

   - 心跳包检测线程（HeartbeatCheck）

   Swoole配置了心跳检测之后，心跳包线程会在固定时间内对所有之前在线的连接发送检测数据包

   - UDP收包线程（UdpRecv）
   
   接收并且处理客户端udp数据包

- 管理进程Manager

Swoole想要实现最好的性能必须创建出多个工作进程帮助处理任务，但Worker进程就必须fork操作，但是fork操作是不安全的，如果没有管理会出现很多的僵尸进程，进而影响服务器性能，同时worker进程被误杀或者由于程序的原因会异常退出，为了保证服务的稳定性，需要重新创建worker进程

Swoole在运行中会创建一个单独的管理进程，所有的worker进程和task进程都是从管理进程Fork出来的。管理进程会监视所有子进程的退出事件，当worker进程发生致命错误或者运行生命周期结束时，管理进程会回收此进程，并创建新的进程。换句话也就是说，对于worker、task进程的创建、回收等操作全权有“保姆”Manager进程进行管理。

Manager进程和Worker/Task进程的关系：

![WeChat050db2e9ab970eb653c3811edd4cc846.png](https://i.loli.net/2019/07/10/5d25946886e9723127.png)


- Worker进程

worker 进程属于swoole的主逻辑进程，用户处理客户端的一系列请求，接受由Reactor线程投递的请求数据包，并执行PHP回调函数处理数据生成响应数据并发给Reactor线程，由Reactor线程发送给TCP客户端可以是异步非阻塞模式，也可以是同步阻塞模式

- Task进程

taskWorker进程这一进城是swoole提供的异步工作进程，这些进程主要用于处理一些耗时较长的同步任务，在worker进程当中投递过来。


Swoole的Reactor、Worker、TaskWorker之间可以紧密的结合起来，提供更高级的使用方式。

一个更通俗的比喻，假设Server就是一个工厂，那Reactor就是销售，接受客户订单。而Worker就是工人，当销售接到订单后，Worker去工作生产出客户要的东西。而TaskWorker可以理解为行政人员，可以帮助Worker干些杂事，让Worker专心工作。



学完了进程的基本结构之后，我们再来看client与server的交互

![WeChat8ff1a11430481283083dca2febdd765c.png](https://i.loli.net/2019/07/10/5d2586251151515619.png)

1、client请求到达 Main Reactor,Client实际上是与Master进程中的某个Reactor线程发生了连接。

2、Main Reactor根据Reactor的情况，将请求注册给对应的Reactor (每个Reactor都有epoll。用来监听客户端的变化) 

3、客户端有变化时Reactor将数据交给worker来处理

4、worker处理完毕，通过进程间通信(比如管道、共享内存、消息队列)发给对应的reactor。 

5、reactor将响应结果发给相应的连接请求处理完成

## 进程及其相对应的事件绑定

- Master进程内的回调函数

onStart   Server启动在主进程的主线程回调此函数

onShutdown  此事件在Server正常结束时发生

- Manager进程内的回调函数

onManagerStart 当管理进程启动时调用它

onManagerStop  当管理进程结束时调用它

onWorkerError  当worker/task_worker进程发生异常后会在Manager进程内回调此函数

- Worker进程内的回调函数

onWorkerStart  此事件在Worker进程/Task进程启动时发生

onWorkerStop    此事件在worker进程终止时发生。

onConnect   有新的连接进入时，在worker进程中回调

onClose   TCP客户端连接关闭后，在worker进程中回调此函数

onReceive 接收到数据时回调此函数，发生在worker进程中

onPacket 接收到UDP数据包时回调此函数，发生在worker进程中

onFinish  当worker进程投递的任务在task_worker中完成时，task进程会通过finish()方法将任务处理的结果发送给worker进程。

onWorkerExit  仅在开启reload_async特性后有效。异步重启特性

onPipeMessage  当工作进程收到由 sendMessage 发送的管道消息时会触发事件


- Task进程内的回调函数

onTask   在task_worker进程内被调用。worker进程可以使用swoole_server_task函数向task_worker进程投递新的任务

onWorkerStart  此事件在Worker进程/Task进程启动时发生

onPipeMessage  当工作进程收到由 sendMessage 发送的管道消息时会触发事件


运行流程图如下所示：

![WeChatdb8757139c4b07dd63ce57ac012da180.png](https://i.loli.net/2019/07/10/5d2593dd2f0ea65645.png)


