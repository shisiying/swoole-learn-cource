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


## 守护进程，信号和平滑重启


### 信号和平滑重启

经过上述代码的编写，我们可以知道，每次修改完代码之后，我们都需要Ctrl+C，然后重新执行代码，这样修改的才会直接生效，这也是Swoole之所以性能卓越对比于传统的php-fpm模式，是因为Swoole减少了每一次请求加载PHP文件以及初始化的开销。但是这种优势也导致开发者无法像过去一样，修改PHP文件，重新请求，就能获取到新代码的运行结果（具体看另外的课程文档）。如果需要新代码开始执行，往往需要先关闭服务器然后重启，这样才能使得新文件被加载进内存运行，这样很明显不能满足开发者的需求。幸运的是，Swoole 提供了这样的功能，但是我们有更友好的方式：我们先了解下信号处理。

在swoole中，我们可以向主进程发送各种不同的信号，主进程根据接收到的信号类型做出不同的处理。比如下面这几个

- 1、kill -SIGTERM|-15 master_pid  终止Swoole程序,一种优雅的终止信号，会待进程执行完当前程序之后中断，而不是直接干掉进程
- 2、kill -USR1|-10  master_pid  重启所有的Worker进程
- 3、kill -USR2|-12  master_pid   重启所有的Task Worker进程 

当USR1信号被发送给Master进程后，Master进程会将同样的信号通过Manager进程转发Worker进程，收到此信号的Worker进程会在处理完正在执行的逻辑之后，释放进程内存，关闭自己，然后由Manager进程重启一个新的Worker进程。新的Worker进程会占用新的内存空间。

那么什么是信号呢？

信号是由用户、系统或者进程发送给目标进程的倍息，以通知目标进程某个状态的改变 或系统异常。Linux信号可由如下条件产生：

- 对于前台进程，用户可以通过输人特殊的终端字符来给它发送信号。比如输入Ctrl+C 通常会给进程发送一个中断信号。
- 系统异常。比如浮点异常和非法内存段访问。
- 系统状态变化。比如alarm定时器到期将引起SIGALRM倍号。
- 运行kill命令或调用kill函数。


### 热重启

不影响用户的情况下重启服务，更新内存中已经加载的php程序代码，从而达到对业务逻辑的更新。swoole为我们提供了平滑重启机制，我们只需要向swoole_server的主进程发送特定的信号，即可完成对server的重启

注意事项：

1、更新仅仅只是针对worker进程，也就是写在master进程跟manger进程当中更新代码并不生效，也就是说只有在onWorkerStart回调之后加载的文件，重启才有意义。在Worker进程启动之前就已经加载到内存中的文件，如果想让它重新生效，只能关闭server再重启

2、直接写在worker代码当中的逻辑是不会生效的，就算发送了信号也不会，需要通过include方式引入相关的业务逻辑代码才会生效


### 利用inotify实现热重启

在传统的nginx+php-fpm模式中，每次请求结束后资源都会被释放，下次有新的请求会重新加载文件，所以只要更新了代码即可马上生效，但是在cli命令行模式开发中，开启的php进程服务一般都是守护进程，代码只在开启时进行加载，就算代码有更新也不会重新加载，直到进程结束都还是最开始加载的代码，导致每次更新代码都要重启php服务，这样的体验是非常不好的，我们可以借用swoole+Inotify来解决这个问题

Inotify介绍

inotify是Linux内核提供的一组系统调用，它可以监控文件系统操作，比如文件或者目录的创建、读取、写入、权限修改和删除等。

inotify使用也很简单，使用inotify_init创建一个句柄，然后通过inotify_add_watch/inotify_rm_watch增加/删除对文件和目录的监听。

PHP中提供了inotify扩展，支持了inotify系统调用。inotify本身也是一个文件描述符，可以加入到事件循环中，配合使用swoole扩展，就可以异步非阻塞地实时监听文件/目录变化

Inotif安装

可以使用 pecl install inotify

在swoole中使用需要注意的事项：

1、OnWorkerStart之后加载的代码都在各自进程中，OnWorkerStart之前加载的代码属于共享内存。

2、可以将公用的，不易变的php文件放置到onWorkerStart之前。这样虽然不能重载入代码，但所有worker是共享的，不需要额外的内存来保存这些数据。onWorkerStart之后的代码每个worker都需要在内存中保存一份

Inotif代码示例：

```php
    <?php
    
    class Worker
    {
        //监听事件连接
        public $onConnect;
        //监听事件关闭
        public $onClose;
        public $onMessage;
        protected $socket;
        //进程数
        protected $allSocket;
        protected $workerNum = 4;
        public $addr;
        protected $woker_pids; //子进程
        protected $master_pid;
    
        public function __construct($socket_address)
        {
            $this->addr = $socket_address;
            $this->master_pid = posix_getpid();
        }
    
        /**
         * 文件监视，自动重启
         */
        protected function watch()
        {
            $init = inotify_init();
            $files = get_included_files();
    
            foreach ($files as $file) {
                inotify_add_watch($init,$file,IN_MODIFY);//监视修改相关的文件
            }
    
            //监听
            swoole_event_add($init,function ($fd){
                $events = inotify_read($fd);
                if (!empty($events)) {
                    posix_kill($this->master_pid,SIGUSR1);
                }
            });
        }
    
        public function fork($workerNum)
        {
            //创建多进程
            for ($i=0;$i<$workerNum;$i++){
    
                $test=include 'index.php';
                var_dump($test);
                $pid=pcntl_fork(); //创建成功会返回子进程id
    
                if($pid<0){
                    exit('创建失败');
                }else if($pid>0){
                    $this->woker_pids[] = $pid;
                    //父进程空间，返回子进程id
                }else{ //返回为0子进程空间
                    $this->accept();
                    exit;
                }
            }
        }
    
        public function accept()
        {
            $opts = array(
                'socket' => array(
                    'backlog' =>10240, //成功建立socket连接的等待个数
                ),
            );
    
            $context = stream_context_create($opts);
            //开启多端口监听,并且实现负载均衡
            stream_context_set_option($context,'socket','so_reuseport',1);
            stream_context_set_option($context,'socket','so_reuseaddr',1);
            $this->socket=stream_socket_server($this->addr,$errno,$errstr,STREAM_SERVER_BIND|STREAM_SERVER_LISTEN,$context);
    
            //设置监听服务端事件
            swoole_event_add($this->socket, function ($fd) {
                $clientSocket = stream_socket_accept($fd);
    
                if (!empty($clientSocket) && is_callable($this->onConnect)) {
                    //触发连接事件的回调
                    call_user_func($this->onConnect, $clientSocket);
                }
    
                //设置监听客户端事件
                swoole_event_add($clientSocket, function ($fd) {
                    //读取客户端数据
                    $buffer = fread($fd, 655535);
    
                    //如果数据为空或者为false，不是资源类型
                    if (empty($buffer)) {
                        if (feof($fd) || !is_resource($fd)) {
                            fclose($fd);
                        }
                    }
    
                    //正常读取数据触发onmesseage回调，响应内容
                    if (!empty($buffer) && is_callable($this->onMessage)) {
                        call_user_func($this->onMessage, $fd, $buffer);
                    }
                });
            });
        }
    
        /**
         * 重启进程
         */
        public function reload()
        {
            foreach ($this->woker_pids as $pid_key => $pid) {
                posix_kill($pid,SIGKILL);//结束进程
                unset($this->woker_pids[$pid_key]);
                $this->fork(1);
            }
        }
    
        /**
         * 信号捕获
         * 监视worker进程，拉起进程
         */
        public function monitorWorkers()
        {
            //注册信号事件回调,是不会自动执行的
            pcntl_signal(SIGUSR1,[$this,'signalHandler'],false);//重启woker进程信号
    
            $status = 0;
    
            while(1) {
                // 发现信号队列,一旦发现有信号就会触发进程绑定事件回调
                pcntl_signal_dispatch();
                //主进程处理子进程
                $pid = pcntl_wait($status);
                //进程重启的过程当中会有新的信号过来,如果没有调用pcntl_signal_dispatch,信号不会被处理
                pcntl_signal_dispatch();
    
            }
        }
    
        public function signalHandler($sigo)
        {
            switch ($sigo) {
                case SIGUSR1:
                    $this->reload();
                    echo "收到信号";
                    break;
            }
    
        }
    
    
        public function start()
        {
            $this->watch();
            $this->fork($this->workerNum);
            $this->monitorWorkers();
        }
    }
    
    $worker = new Worker('tcp://0.0.0.0:9080');
    
    $worker->onConnect = function ($fd){
        echo '连接事件触发',(int)$fd,PHP_EOL;
    };
    
    $worker->onMessage = function ($conn,$message){
    
        $content="hello world!";
        $http_resonse = "HTTP/1.1 200 OK\r\n";
        $http_resonse .= "Content-Type: text/html;charset=UTF-8\r\n";
        $http_resonse .= "Connection: keep-alive\r\n"; //连接保持
        $http_resonse .= "Server: php socket server\r\n";
        $http_resonse .= "Content-length: ".strlen($content)."\r\n\r\n";
        $http_resonse .= $content;
        fwrite($conn, $http_resonse);
    };
    
    
    $worker->start();
```

在swoole中集成Inotif实现热重启

代码示例如下:

```php

<?php
class Worker{
    //监听socket
    protected $socket = NULL;
    //连接事件回调
    public $onConnect = NULL;
    public  $reusePort=1;
    //接收消息事件回调
    public $onMessage = NULL;
    public $workerNum=3; //子进程个数
    public  $allSocket; //存放所有socket
    public  $addr;
    protected $worker_pid; //子进程pid
    protected  $master_pid;//主进程id
    protected  $watch_fd;//文件监视的句柄
    public function __construct($socket_address) {
        //监听地址+端口
        $this->addr=$socket_address;
        $this->master_pid=posix_getpid();
    }

    public function start() {
        //获取配置文件
        $this->watch();
        $this->fork($this->workerNum);
        $this->monitorWorkers(); //监视程序,捕获信号,监视worker进程
    }

    /**
     * 文件监视,自动重启
     */
    protected  function watch(){

        $this->watch_fd=inotify_init(); //初始化
        $files=get_included_files();
        foreach ($files as $file){
            inotify_add_watch($this->watch_fd,$file,IN_MODIFY); //监视相关的文件
        }
        //监听
        swoole_event_add($this->watch_fd,function ($fd){
            $events=inotify_read($fd);
            if(!empty($events)){
                posix_kill($this->master_pid,SIGUSR1);
            }
        });
    }
    /**
     * 捕获信号
     * 监视worker进程.拉起进程
     */
    public  function monitorWorkers (){
         //注册信号事件回调,是不会自动执行的
        // reload
        pcntl_signal(SIGUSR1, array($this, 'signalHandler'),false); //重启woker进程信号
        //ctrl+c
        pcntl_signal(SIGINT, array($this, 'signalHandler'),false); //重启woker进程信号

        //
        $status=0;
        while (1){
            // 当发现信号队列,一旦发现有信号就会触发进程绑定事件回调
            pcntl_signal_dispatch();
            $pid = pcntl_wait($status); //当信号到达之后就会被中断

            //会去查询我们的子进程id
            $index=array_search($pid,$this->worker_pid);
            //如果进程不是正常情况下的退出,重启子进程,我想要维持子进程个数
            if($pid>1 && $pid != $this->master_pid && $index!=false && !pcntl_wifexited($status)){
                     $index=array_search($pid,$this->worker_pid);
                     $this->fork(1);
                     var_dump('拉起子进程');
                     unset($this->worker_pid[$index]);
            }
            pcntl_signal_dispatch();
            //进程重启的过程当中会有新的信号过来,如果没有调用pcntl_signal_dispatch,信号不会被处理

        }
    }

    public function signalHandler($sigo){
        switch ($sigo){
            case SIGUSR1:
                $this->reload();
                echo "收到重启信号";
                break;
            case SIGINT:
                $this->stopAll();
                echo "按下ctrl+c,关闭所有进程";
                swoole_event_del($this->watch_fd);
                exit();
                break;
        }

    }
    public function fork($worker_num){
        for ($i=0;$i<$worker_num;$i++){
            $test=include 'index.php';
            var_dump($test);
            $pid=pcntl_fork(); //创建成功会返回子进程id
            if($pid<0){
                exit('创建失败');
            }else if($pid>0){
                //父进程空间，返回子进程id
                $this->worker_pid[]=$pid;
            }else{ //返回为0子进程空间
                $this->accept();//子进程负责接收客户端请求
                exit;
            }
        }
        //放在父进程空间，结束的子进程信息，阻塞状态

    }
    public  function  accept(){
        $opts = array(
            'socket' => array(
                'backlog' =>10240, //成功建立socket连接的等待个数
            ),
        );

      $context = stream_context_create($opts);
      //开启多端口监听,并且实现负载均衡
      stream_context_set_option($context,'socket','so_reuseport',1);
      stream_context_set_option($context,'socket','so_reuseaddr',1);
      $this->socket=stream_socket_server($this->addr,$errno,$errstr,STREAM_SERVER_BIND|STREAM_SERVER_LISTEN,$context);

        //第一个需要监听的事件(服务端socket的事件),一旦监听到可读事件之后会触发
        swoole_event_add($this->socket,function ($fd){
                $clientSocket=stream_socket_accept($fd);
                //触发事件的连接的回调
                if(!empty($clientSocket) && is_callable($this->onConnect)){
                    call_user_func($this->onConnect,$clientSocket);
                }
            //监听客户端可读
             swoole_event_add($clientSocket,function ($fd){
                //从连接当中读取客户端的内容
                $buffer=fread($fd,1024);
                //如果数据为空，或者为false,不是资源类型
                if(empty($buffer)){
                    if(!is_resource($fd) || feof($fd) ){
                        //触发关闭事件
                        fclose($fd);
                    }
                }
                //正常读取到数据,触发消息接收事件,响应内容
                if(!empty($buffer) && is_callable($this->onMessage)){
                    call_user_func($this->onMessage,$fd,$buffer);
                }


            });
        });
    }


    /**
     * 重启worker进程
     */
    public  function reload(){
        foreach ($this->worker_pid as $index=>$pid){
            posix_kill($pid,SIGKILL); //结束进程
            var_dump("杀掉的子进程",$pid);
            unset($this->worker_pid[$index]); //删除之前的pid了所以worker_pid当中没有记录了
            $this->fork(1); //重新拉起worker
        }
    }

    //捕获信号之后重启worker进程
    public  function stopAll(){
        foreach ($this->worker_pid as $index=>$pid){
            posix_kill($pid,SIGKILL); //结束进程
            unset($this->worker_pid[$index]);
        }

    }

}


//ps -ef | grep php | grep -v grep | awk '{print $2}' | xargs kill -s 9
$worker = new Worker('tcp://0.0.0.0:9800');

//开启多进程的端口监听
$worker->reusePort = true;

//连接事件
$worker->onConnect = function ($fd) {
    //echo '连接事件触发',(int)$fd,PHP_EOL;
};

$worker->onTask = function ($fd) {
    //echo '连接事件触发',(int)$fd,PHP_EOL;
};

//消息接收
$worker->onMessage = function ($conn, $message) {
    //事件回调当中写业务逻辑
    // $a=include 'index.php';
    // var_dump($a);

     //$server->task(); //在worker进程当中能够向task进程发送消息

    //var_dump($conn,$message);
    $content="hello wrodl";
    $http_resonse = "HTTP/1.1 200 OK\r\n";
    $http_resonse .= "Content-Type: text/html;charset=UTF-8\r\n";
    $http_resonse .= "Connection: keep-alive\r\n"; //连接保持
    $http_resonse .= "Server: php socket server\r\n";
    $http_resonse .= "Content-length: ".strlen($content)."\r\n\r\n";
    $http_resonse .= $content;
    fwrite($conn, $http_resonse);
};

$worker->start(); //启动



```
