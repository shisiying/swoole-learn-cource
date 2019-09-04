# task进程异步任务实现

在之前的swoole的进程结构中，我们就已经讲过task进程，对于一些耗时任务，就交给相应的taskworker进程处理

## 什么是task进程

task进程是独立与worker进程的一个进程，主要处理耗时较长的业务逻辑，并且不影响worker进程处理客户端的请求，这大大提高了swoole的并发能力，当有耗时较长的任务时，worker进程通过task()函数把数据投递到task进程去处理。

![WeChat820cb89db7fe8d13d4094f451fe04976.png](https://i.loli.net/2019/07/10/5d255d416a67f15358.png)

在worker进程当中我们可以执行客户端提交过来的数据执行相应的逻辑，也就是在触发的事件当中，比如接收客户端提交的数据，处理同步的业务，这是worker进程当中执行，taks进程适用场景如下所示：

## task进程适用场景

- 情景一：管理员需要给100W用户发送邮件，当点击发送，浏览器会一直转圈，直到邮件全部发送完毕。
- 情景二：千万微博大V发送一条微博，其关注的粉丝相应的会接收到这个消息，是不是大V需要一直等待消息发送完成，才能执行其它操作
- 情景三：处理几TB数据
- 情景四: 数据库读写操作
- 情景五：广播消息

其实也就是提高用户体验，不用一直等待任务的完成

## task进程与worker进程的交互

![WeChat411beee0fdba321735283ebe8a3723a4.png](https://i.loli.net/2019/07/23/5d366a3e23e6e84242.png)

- 1、worker进程当中，我们调用对应的task()方法发送数据通知到task worker进程
- 2、task worker进程会在onTask()回调中接收到这些数据，并进行处理。
- 3、处理完成之后通过调用finsh()函数或者直接return返回消息给worker进程
- 4、worker进程在onFinsh()进程收到这些消息并进行处理


代码示例如下:

```php
    <?php
    
    //tcp协议
    $server=new Swoole\Server("0.0.0.0",9801);   //创建server对象
    
    
    $server->set([
        'worker_num'=>2, //设置进程
        'task_worker_num'=>3,  //task进程数
        /**
        设置Task进程与Worker进程之间通信的方式。
        进程间通信知识请看最后面的补充知识
        1, 使用Unix Socket通信，默认模式
        2, 使用消息队列通信
        3, 使用消息队列通信，并设置为争抢模式
        **/
        'task_ipc_mode'=>2, 
    ]);
    
    $server->on('start',function (){
    });
    
    
    $server->on('Shutdown',function (){
        echo "正常关闭";
    });
    
    $server->on('workerStart',function ($server,$fd){
    
    });
    
    
    //监听事件,连接事件
    $server->on('connect',function ($server,$fd){
    
        //echo "新的连接进入xxx:{$fd}".PHP_EOL;
    });
    
    
    //消息发送过来
    $server->on('receive',function (swoole_server $server, int $fd, int $reactor_id, string $data){
        //不需要立刻马上得到结果的适合task
        $data=['tid'=>time()];
        //sleep(10);
        $server->task($data); //投递到taskWorker进程组
        //服务端
    });
    
    //ontask事件回调
    $server->on('task',function ($server,$task_id,$form_id,$data){
        echo "任务来自于:$form_id".",任务id为{$task_id}".PHP_EOL;
        sleep(2);
        $server->finish("执行完毕");
    });
    
    $server->on('finish',function ($server,$task_id,$data){
        echo "任务{$task_id}执行完毕:{$data}".PHP_EOL;
    });
    
    //消息关闭
    $server->on('close',function (){
        //echo "消息关闭".PHP_EOL;
    });
    
    //服务器开启
    $server->start();

```

参数注意：

- $task_id不等于workerStart中的worker_id,而是swoole维护的任务自增长id
- $from_id表示来自于哪个woker投递而来的任务，id等于worker_id值
- $serv->worker_id等于回调方法workerStart中的worker_id
- $serv->worker_pid 此处是task_woker的进程pid
- worker_id跟task_id都不是进程id

使用task进程需要注意的事项

一、开启task功能

task功能默认是关闭的，开启task功能需要满足两个条件

1.	配置task进程的数量

2.	注册task的回调函数onTask和onFinish

配置task进程的数量，即配置task_worker_num这个配置项。比如我们开启8个task进程，同样task进程数量的配置也不是随意的配置

计算方法

单个task的处理耗时，如100ms，那一个进程1秒就可以处理1/0.1=10个task

task投递的速度，如每秒产生2000个task

2000/10=200，需要设置task_worker_num => 200，启用200个task进程

对于单个服务器的承受数量我们要提前做预知，处理能力必须是大于投放能力的

demo场景以及代码示例子

>场景：假设有一台服务器专门处理前台投递的数据,利用简单的任务拆分,分配到相应的进程去处理
   1.将一个大的任务拆分成相应份数(是由$task_worker_num数量来确定)
   2.通过foreach循环将数据投递到指定的task进程,范围是(0-(task_worker_num-1))区间之内
   3.执行失败的任务，需要保留，重新投递执行（进程间通讯管道方式）


client端：

```php
<?php

$client=new swoole\Client(SWOOLE_SOCK_TCP);

//发数据
$client->connect('0.0.0.0',9801);
$client->send('456');

//var_dump(strlen($client->recv())); //接收消息没有接收
$client->close();


```

server端：

```php

    <?php
    
    //tcp协议
    $server=new Swoole\Server("0.0.0.0",9801);   //创建server对象
    
    
    $server->set([
        'worker_num'=>2, //设置进程
        //'heartbeat_idle_time'=>10,//连接最大的空闲时间
        //'heartbeat_check_interval'=>3 //服务器定时检查
        'task_worker_num'=>3,  //task进程数
        'task_ipc_mode'=>2,
    ]);
    
    $server->on('start',function (){
        // include 'index.php'; 不能
    });
    
    
    $server->on('Shutdown',function (){
    
        // include 'index.php'; 不能
        echo "正常关闭";
    });
    
    $server->on('workerStart',function ($server,$fd){
        echo $fd;
        if($server->taskworker){
            echo 'task_worker:'.$server->worker_id.PHP_EOL;
        }else{
            echo 'worker:'.$server->worker_id.PHP_EOL;
        }
    
    });
    
    $server->on('PipeMessage',function (swoole_server $server, int $src_worker_id,$message){
        echo "来自于{$src_worker_id}的错误信息".PHP_EOL;
        //接收到投递的错误信息，记录错误次数，错误次数到达一定次数之后，就保留日志
    });
    
    
    //监听事件,连接事件
    $server->on('connect',function ($server,$fd){
    
        //echo "新的连接进入xxx:{$fd}".PHP_EOL;
    });
    
    
    //消息发送过来
    $server->on('receive',function (swoole_server $server, int $fd, int $reactor_id, string $data){
        for ($i=0;$i<100;$i++){
            $tasks[] =['id'=>$i,'msg'=>time()];
        }
        $count=count($tasks);
        $data=array_chunk($tasks,ceil($count/3));
        foreach ($data as $k=>$v){
            $server->task($v,$k);  //(0-task_woker_num-1)
        }
    });
    
    //ontask事件回调
    $server->on('task',function ($server,$task_id,$form_id,$data){
        echo "任务来自于:$form_id".",任务id为{$task_id}".PHP_EOL;
        try{
            foreach ($data as $k=>$v){
                if(mt_rand(1,5)==3){ //故意的去出现错误
                    $server->sendMessage($v,1); //主动去通知worker进程，0 ~ (worker_num + task_worker_num - 1）
                }
            }
        }catch (\Exception $e){
            //$server->sendMessage();
        }
        $server->finish("执行完毕");
    
    });
    
    $server->on('finish',function ($server,$task_id,$data){
        echo "任务{$task_id}执行完毕:{$data}".PHP_EOL;
    });
    
    
    
    //消息关闭
    $server->on('close',function (){
        //echo "消息关闭".PHP_EOL;
    });
    
    
    
    //服务器开启
    $server->start();
    


```


具体业务逻辑我们可以继续扩展。

常见问题:

- Task传递数据的大小

数据小于8k直接通过管道传递,数据大于8k写入临时文件传递onTask会读取这个文件,把他读出来运行Task,必须要在swoole服务中配置参数task_worker_num,此外，必须给swoole_server绑定两个回调函数：onTask和onFinish。 onTask要return 数据 

- Task传递对象

默认task只能传递数据，可以通过序列化传递一个对象的拷贝,Task中对象的改变并不会反映到worker进程中数据库连接,网络连接对象不可传递,会引起php报错

- Task的onFinish回调

Task的onFinish回调会回调调用task方法的worker进程

- task_max_request 设置task进程的最大任务数。一个task进程在处理完超过此数值的任务后将自动退出。这个参数是为了防止PHP进程内存溢出。如果不希望进程自动退出可以设置为0

- 每个woker都有可能投递任务给不同的task_worker处理, 不同的woker进程内存隔离,记录着worker_id, 标识woker进程任务处理数量

-注意:
   如果其他语言,或者是传统的php-fpm也希望投递消息到task当中处理,也是可行的
方法请参考下面链接: https://wiki.swoole.com/wiki/page/212.html

加入task的网络服务器的演进

补充知识：

进程间的通信

>进程间通信（IPC，Inter-Process Communication），指至少两个进程或线程间传送数据或信号的一些技术或方法。每个进程都有自己的一部分独立的系统资源，彼此是隔离的。为了能使不同的进程互相访问资源并进行协调工作，才有了进程间通信。

进程通信有如下的目的：

- 数据传输，一个进程需要将它的数据发送给另一个进程，发送的数据量在一个字节到几M之间；

- 共享数据，多个进程想要操作共享数据，一个进程对数据的修改，其他进程应该立刻看到；

- 进程控制，有些进程希望完全控制另一个进程的执行（如Debug进程），此时控制进程希望能够拦截另一个进程的所有异常，并能够及时知道它的状态改变。

系统进行进程间通信（IPC）的时候，可用的方式包括管道、命名管道、消息队列、信号、信号量、共享内存、套接字(socket)等形式。

我们接下来来看常用的两种方式：

- 消息队列

消息队列实际上就是一个链表，而消息就是链表中具有特定格式和优先级的记录，对消息队列有写权限的进程可以根据一定规则在消息链表中添加消息，对消息队列有读权限的进程则可以从消息队列中获得所需的信息。

在某个进程往一个消息队列写入消息之前，并不需要另外某个进程在该队列上等待消息的到达。对于消息队列来说，除非显式删除，否则其一直存在
php实现消息队列操作

在php中通过这两句话就可以创建一个消息队列。 ftok 函数，是可以将一个路径转换成 消息队列 可用的key值。 msg_get_queue函数的第一个参数 是消息队列的key，第二个参数是消息队列的读写权限，这个权限跟文件类似

注意：需要开启sysvmsg

涉及到的函数有以下几个：

   - msg_send函数，向指定消息队列写入信息。
   - msg_receive 读取函数 
   - msg_remove_queue($msg_queue)，销毁消息队列的方法 

php实现的代码如下：

```php

<?php

//父进程跟子进程实现消息发送

$msg_key=ftok(__DIR__,'u'); //注意在php创建消息队列，第二个参数会直接转成字符串，可能会导致通讯失败
$msg_queue=msg_get_queue($msg_key);


$pid=pcntl_fork();

if($pid==0){
    //子进程发送消息
    msg_send($msg_queue,10,"我是子进程发送的消息");
    exit();
}elseif ($pid){

    msg_receive($msg_queue,10,$message_type,1024,$message);
    var_dump($message);
    //父进程接收消息
    pcntl_wait($status);
    msg_remove_queue($msg_queue);

}


```


- 共享内存

在系统内存中开辟一块内存区，分别映射到各个进程的虚拟地址空间中，任何一个进程操作了内存区都会反映到其他进程中，各个进程之间的通信并没有像copy数据一样从内核到用户，再从用户到内核的拷贝。这种方式可以像访问自己的私有空间一样访问共享内存区，
是速度最快的一种通信方式。但是这事这种特性加大了共享内存的编程难度，比如多个进程同时读取到一个数据做操作，容易造成数据的混乱。


swoole中的实现：swoole_table

swoole_table一个基于共享内存和锁实现的超高性能，并发数据结构。用于解决多进程/多线程数据共享和同步加锁问题,应用代码无需加锁，swoole_table内置行锁自旋锁，所有操作均是多线程/多进程安全。用户层完全不需要考虑数据同步问题


使用方法：

创建一个共享内存的table	 使用swoole共享table非常简单,简单几步就可以创建

-   1、实例化table并且设置最大行数	

    new swoole_table(10)

-   2、指定表格字段，指定表格类型以及长度

    $this->table->column('task_worker_id',swoole_table::TYPE_INT,2)

-   3、创建表格
 
    $this->table->create()
 
-   4、设置一行数据

    $this->table->set('0',['task_worker_id'=>1,'status'=>0])

 注意事项：
在swoole_server->start()之前创建swoole_table对象。并存入全局变量或者类静态变量/对象属性中，在worker/task进程中获取table对象，并使用。
只有在swoole_server->start()之前创建的table对象才能在子进程中使用
swoole_table构造方法中指定了最大容量，一旦超过此数据容量将无法分配内存导致set操作失败。所以使用swoole_table之前一定要规划好数据容量。
