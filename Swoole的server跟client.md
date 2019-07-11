# server跟client

swoole是面向生产环境的PHP异步网络通信引擎，对于初学者来说，我们先看其怎么用的，然后再探究其原理，接下来我们先看最简单的server跟client。

本文的主要内容有：

- <a href="##server">server</a>（其中粘包的处理比较重要）
- <a href="##client">client</a>
- <a href="##心跳检测">心跳检测</a>

## server

server，顾名思义，就是服务端。这也涉及到了网络编程部分，网络编程就是编写程序使两台连网的计算机相互交换数据，网络编程领域需要一定的操作系统和系统编程知识，同时还需要理解好TCP/IP网络数据传输协议。进程以及操作系统相关知识我们后续会讲到，先简单介绍下TCP&IP协议结构。

我们平时接触比较多的无非就是nginx和apache。作为webServer，二者都是通过监听某端口对外提供服务,swoole的server也不例外同样需要绑定端口,同时能够提供给客户端相关的服务。

### TCP/IP协议

- 什么是网络协议

网络协议为计算机网络中进行数据交换而建立的规则，标准或约定的集合

- 协议分层

TCP/IP协议族是一个四层协议系统，自底而上分别是数据链路层，网络蹭，传输层，和应用层，每一层完成不同的功能，且通过若干协议来实现，上层协议使用下层协议提供的服务。

   ![TCP/IP协议族体系结构及主要协议](https://i.loli.net/2019/07/09/5d243783dbb4025293.png)

   - 数据链路层
     
      数据链路层实现了网卡接口的网络驱动程序，以处理数据在物理媒介上的传输。
     数据链路展两个常用的协议是ARP协议（Address Resolve Protocol,地址解析协议）和RARP协议（Reverse Address Resolve Protocol,逆地址解析协议）.它们实现了 IP地址和机器物理地址（MAC地址）之间的相互转换，那处理的流程是怎么样的呢？
     
     当数据交换时上层网络层使用IP地址寻找一台机器，而数据链路层使用物理地址寻找一台机器，因此网络层必须先将目标标机器的IP地址转化成其物理地址,才能使用数据链路提供的服务，这就 ARP协议的用途
   
   - 网络层
      
      网络层实现数据包的选路和转发，网络层最核心的协议是IP协议（Internet Protocol,因特网协议）。IP协议根据数据包的目的IP地址来决定如何投递它。

      网络层另外一个重要的协议足ICMP协议（Internet Control Message Protocol»因特网控制报文协议）。它是IP协议的重要要补充，主要用于检测网络连接，比如可以通过发送报文来检测目标是否可达


   - 传输层
   
     网络层当中的IP协议为了解决传输当中的路径选择的问题，只需要按照此路径传输数据即可，传输层利用udp或者tcp协议以及网络层提供的路径信息为基础完成实际的数据传输，所以称为传输层

     TCP和UDP对比介绍

     - 1、TCP面向连接（如打电话要先拨号建立连接）;UDP是无连接的，即发送数据之前不需要建立连接

     - 2、TCP提供可靠的服务。也就是说，通过TCP连接传送的数据，无差错，不丢失，不重复，且按序到达;UDP尽最大努力交付，即不保证可靠交付

     - 3、tcp通过校验和，重传控制，序号标识，滑动窗口、确认应答实现可靠传输。如丢包时的重发控制，还可以对次序乱掉的分包进行顺序控制。

     - 3、UDP具有较好的实时性，工作效率比TCP高，适用于对高速传输和实时性有较高的通信或广播通信。

     - 4、TCP对系统资源要求较多，UDP对系统资源要求较少

   - 应用层
   
     应用层负责处理应用程序的逻辑。数据链路层、网络层和传输层负责处理网络通讯细节，这部分必须既稳定又高效,因此它们都在内核空间中实现，应用层则负责在用户户空间实现，因为为它负责处理众多逻辑。

     应用层协议举例：
     
     FTP文件传输协议是TCP/IP网络上两台计算机传送文件的协议
     
     SMTP（SimpleMailTransferProtocol）即简单邮件传输协议

- TCP粘包处理

TCP通信特点

1. TCP 是流式协议没有消息边界，客户端向服务器端发送一次数据，可能会被服务器端分成多次收到。客户端向服务器端发送多条数据。服务器端可能一次全部收到。

2.保证传输的可靠性，顺序。

3.TCP拥有拥塞控制，所以数据包可能会延后发送。

没有消息边界:
可以理解为水在一个水管里的流动，我们不知道哪段数据是一个我们需要的完整数据

收发有缓冲区:
比如：当水从一端流到了另一端，我们在收数据的时候，不可能每来一滴水就处理一次，这个缓冲区就相当于有了一个水桶，再接了一定的水之后内核再给数据交到用户空间，这样可以大大提升性能。


a.什么是TCP粘包

TCP 粘包是指发送方发送的若干包数据 到 接收方接收时粘成一包，从接收缓冲区看，后一包数据的头紧接着前一包数据的尾。

b.出现粘包的原因

发送方：发送方需要等缓冲区满才发送出去，造成粘包

接收方：接收方不及时接收缓冲区的包，造成多个包接收

![WeChat303acd1c446c5770a15ec705b9f2bb1f.png](https://i.loli.net/2019/07/10/5d255abb8b0cc83780.png)

每个 socket 被创建后，都会分配两个缓冲区，输入缓冲区和输出缓冲区。
write()/send() 并不立即向网络中传输数据，而是先将数据写入缓冲区中，再由TCP协议将数据从缓冲区发送到目标机器。一旦将数据写入到缓冲区，函数就可以成功返回，不管它们有没有到达目标机器，也不管它们何时被发送到网络，这些都是TCP协议负责的事情。

TCP协议独立于 write()/send() 函数，数据有可能刚被写入缓冲区就发送到网络，也可能在缓冲区中不断积压，多次写入的数据被一次性发送到网络，这取决于当时的网络情况、当前线程是否空闲等诸多因素，不由程序员控制。


如果还看不懂swoole的例子，请先看跳过下面粘包的例子，先看下面的swoole是如何创建server以及client对象

模拟粘包的例子：

server端：

```php
    <?php
    //tcp协议
    $server=new Swoole\Server("0.0.0.0",9800);   //创建server对象
    
    $server->set([
        'worker_num'=>1, //设置进程
        'heartbeat_idle_time'=>10,//连接最大的空闲时间
        'heartbeat_check_interval'=>3 //服务器定时检查
    ]);
    
    //监听事件,连接事件
    $server->on('connect',function ($server,$fd){
        echo "新的连接进入:{$fd}".PHP_EOL;
    });
    
    
    //消息发送过来
    $server->on('receive',function (swoole_server $server, int $fd, int $reactor_id, string $data){
        var_dump("消息发送过来:".$data);
    });
    
    //消息关闭
    $server->on('close',function (){
        echo "消息关闭".PHP_EOL;
    });
    //服务器开启
    $server->start(); 
```


客户端例子：

```php
    <?php
     $client=new swoole\Client(SWOOLE_SOCK_TCP);
     //发数据
     $client->connect('127.0.0.1',9800);
     //一次性发送多条数据
     for ($i=0;$i<10;$i++){
         $client->send("123456");
     }
```

这时候，理想状态下是，会出现一行一行的123456，但是多尝试几次会发现，有几个会连在一起，这就是粘包现象，至于产生的原因，上述已经写过

c.swoole怎么处理粘包

- EOF结束协议

通过约定结束符，来确定包数据是否发送完毕
开启open_eof_check=true,并用package_eof来设置一个完整数据结尾字符，同时设置自动拆分open_eof_split

注意：
1、要保证业务数据里不能出现package_eof设置的字符,否则将导致数据错误了。
2、可以手动拆包，去掉open_eof_split,自行 explode("\r\n", $data),然后循环发送

代码示例：
服务端代码:

```php
   <?php
   /**
    * Created by PhpStorm.
    * User: Sixstar-Peter
    * Date: 2019/2/19
    * Time: 21:37
    */
   
   //tcp协议
   $server=new Swoole\Server("0.0.0.0",9800);   //创建server对象
   
   $server->set([
       'worker_num'=>1, //设置进程
       'heartbeat_idle_time'=>10,//连接最大的空闲时间
       'heartbeat_check_interval'=>3, //服务器定时检查
       'package_eof' => "\r\n", //设置EOF
       'open_eof_split' => true //开启自动拆分
   ]);
   
   //监听事件,连接事件
   $server->on('connect',function ($server,$fd){
       echo "新的连接进入:{$fd}".PHP_EOL;
   });
   
   
   //消息发送过来
   $server->on('receive',function (swoole_server $server, int $fd, int $reactor_id, string $data){
       var_dump("消息发送过来:".$data);
   });
   
   //消息关闭
   $server->on('close',function (){
       echo "消息关闭".PHP_EOL;
   });
   //服务器开启
   $server->start();
   
```

客户端代码

```php
    <?php
    
    $client=new swoole\Client(SWOOLE_SOCK_TCP);
     //发数据
     $client->connect('127.0.0.1',9800);
    
     //一次性发送多条数据
     for ($i=0;$i<10;$i++){
         $client->send("123456\r\n");
     }

```

- 固定包头+包体协议

这种方式也非常常见，原理是通过约定数据流的前几个字节来表示一个完整的数据有多长，从第一个数据到达之后，先通过读取固定的几个字节，解出数据包的长度，然后按这个长度继续取出后面的数据，依次循环。

相关配置：

open_length_check：打开包长检测特性

package_length_type：长度字段的类型，固定包头中用一个4字节或2字节表示包体长度。

package_length_offset：从第几个字节开始是长度，比如包头长度为120字节，第10个字节为长度值，这里填入9（从0开始计数）

package_body_offset：从第几个字节开始计算长度，比如包头为长度为120字节，第10个字节为长度值，包体长度为1000。如果长度包含包头，这里填入0，如果不包含包头，这里填入120

package_max_length：最大允许的包长度。因为在一个请求包完整接收前，需要将所有数据保存在内存中，所以需要做保护。避免内存占用过大。


package_length_type 长度值的类型

长度值的类型，接受一个字符参数，与php的pack函数一致。目前swoole支持10种类型：

c：有符号、1字节

C：无符号、1字节

s：有符号、主机字节序、2字节

S：无符号、主机字节序、2字节

n：无符号、网络字节序、2字节 (常用)

N：无符号、网络字节序、4字节 (常用)

l：有符号、主机字节序、4字节（小写L）

L：无符号、主机字节序、4字节（大写L）

v：无符号、小端字节序、2字节

V：无符号、小端字节序、4字节


示例代码如下所示:


服务端代码：
```php
    <?php

    //tcp协议
    $server=new Swoole\Server("0.0.0.0",9800);   //创建server对象
    
    
    $server->set([
        'worker_num'=>1, //设置进程
        'open_length_check'=>1,
        'package_length_type'=>'N',//设置包头的长度
        'package_length_offset'=>0, //包长度从哪里开始计算
        'package_body_offset'=>4,  //包体从第几个字节开始计算
        'package_max_length'=>1024 * 1024 * 2,
    
    ]);
    
    
    //监听事件,连接事件
    $server->on('connect',function ($server,$fd){
        echo "新的连接进入:{$fd}".PHP_EOL;
    });
    
    
    //消息发送过来
    $server->on('receive',function (swoole_server $server, int $fd, int $reactor_id, string $data){
    
        //解包，并且截取数据包，截取的长度就是包头的长度
        $info=unpack('N',$data);
        var_dump('长度',$info);//这里是8
        var_dump(substr($data,4));
    });
    
    //消息关闭
    $server->on('close',function (){
        echo "消息关闭".PHP_EOL;
    });
    //服务器开启
    $server->start();
```

客户端代码：

```php
    <?php
    
    $client=new swoole\Client(SWOOLE_SOCK_TCP);
    //发数据
    $client->connect('127.0.0.1',9800);
    
    $body="sevenshi";
    //包头+包体
    $data=pack("N",strlen($body)).$body;
    
    var_dump($data);//这里应该是12
    $client->send($data);
```

- 字节序的理解  

计算机硬件有两种储存数据的方式：大端字节序（big endian）和小端字节序（little endian）。

大端字节序：高位字节在前，低位字节在后，这是人类读写数值的方法。

小端字节序：低位字节在前，高位字节在后，即以0x1122形式储存。

为什么要有字节序呢？

计算机电路先处理低位字节，效率比较高，因为计算都是从低位开始的。所以，计算机的内部处理都是小端字节序。

不过，人类还是习惯读写大端字节序。所以，除了计算机的内部处理，其他的场合几乎都是大端字节序，比如网络传输和文件储存。

所以我们要确定通信双方交流的信息单元应该以什么样的顺序进行传送。如果达不成一致的规则，计算机的通信与存储将会无法进行


### 创建一个server对象

创建server的步骤:

- 1、实例化Server对象
- 2、设置运行时参数
- 3、注册事件回调函数
- 4、启动服务器

示例如下所示：

```cmd
    <?php
    //创建server对象，监听9051端口
    $serv = new swoole_server('0.0.0.0',9051);
    
    //监听连接进入事件，有客户端连接进来的时候会触发
    $serv->on('connect',function ($serv,$fd){
        echo "有新的客户端连接,连接标示为$fd".PHP_EOL;
    });
    
    //监听数据接收事件，server接收到客户端的数据后，worker进程内触发该回调
    $serv->on('receive',function ($serv,$fd,$from_id,$data){
       $serv->send($fd,"服务器给你发送消息了：".$data);
    });
    
    //监听连接关闭事件，客户端关闭或者服务端主动关闭
    $serv->on("close",function ($serv,$fd){
        echo "编号为{$fd}的客户端已经关闭".PHP_EOL;
    });
    
    //启动服务器
    $serv->start();

```


#### 可配置的参数

- worker进程数的配置

因为swoole是多进程的异步服务器所以需要设置工作进程数，提升服务器性能，官方建议我们设置为CPU核数的1-4倍。因为我们开的进程越多，内存的占用也就更多，进程间切换也就需要耗费更多的资源。我们这里设置开启两个worker进程。默认该参数的值等于你机器的CPU核数



#### 常见的事件回调

```cmd
  //监听连接进入事件，有客户端连接进来的时候会触发
    $serv->on('connect',function ($serv,$fd){
        echo "有新的客户端连接,连接标示为$fd".PHP_EOL;
    });
```

参数$serv是我们一开始创建的swoole_server对象，

参数$fd是唯一标识，文件描述符，用于区分不同的客户端，如果还不懂啥意思的话，可以看下《TCP&IP网络编程》这本书，书中介绍，在Linux世界里，socket也被认为是文件的一种，因此在网络数据传输过程中自然可以使用文件I/O的相关函数。实际上，文件描述符只不过是为了方便称呼操作系统创建的文件或套接字而赋予的数而已。至于套接字可以简单理解为网络数据传输用的软件设备,即使对网络数据传输原理不太熟悉，我们也能通过套接字完成数据传输
由于swoole_server只能运行在CLI模式下，所以不要试图通过浏览器进行访问，在命令行执行```php swoole-server.php```,这时候你会发现，当前程序一直处于执行中的状态，并没有退出终端，平时，我们执行完一个指令，执行完就结束了，因为swoole的server是常驻内存运行的，所以如果修改了服务端的代码，我们需要ctrl+c终端，重新运行才行。


## client

### 创建一个同步客户端

相信如果是写php的同学，对于同步异步的概念应该会很陌生，swoole是即支持全异步，也支持同步，因此我们来简单了解下同步跟异步的区别：

同步跟异步的重点在于消息通知的方式上，也就是调用结果通知的方式

- 同步：当一个同步调用发出去后，调用者要一直等待调用结果的通知后，才能进行后续下一步的执行
- 异步：当一个异步调用发出去后，调用者不用一直等待调用结果的返回，适合没有顺序依赖的程序执行

同步客户端示例子如下：

```php

    $client = new swoole_client(SWOOLE_SOCK_TCP);
    if (!$client->connect('localhost',9051, -1))
    {
        exit("connect failed. Error: {$client->errCode}\n");
    }
    $client->send("hello world\n");
    echo $client->recv();
    $client->close();

```

同步client是同步阻塞的。一整套connect->send()->rev()->close()是同步进行的。如果需要大量的数据处理，后台不能在规定的时间内返回数据会导致接收超时，并且因为是同步执行所以需要等待后台数据的返回


### 创建一个异步客户端

```php
    $client = new Swoole\Client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_ASYNC);
    $client->on("connect", function(swoole_client $cli) {
        $cli->send("GET / HTTP/1.1\r\n\r\n");
    });
    $client->on("receive", function(swoole_client $cli, $data){
        echo "Receive: $data";
        $cli->send(str_repeat('A', 100)."\n");
        sleep(1);
    });
    $client->on("error", function(swoole_client $cli){
        echo "error\n";
    });
    $client->on("close", function(swoole_client $cli){
        echo "Connection close\n";
    });
    $client->connect('127.0.0.1', 9501);
```

当设定swoole_client为异步模式后，swoole_client就不能使用recv方法了，而需要通过on方法提供指定的回调函数，然后在回调函数当中处理,消息发送跟接收并不是同步运行的。


## 心跳检测

因为有server跟client相互连接，所以我们也要了解到心跳检测这个概念

### 心跳是什么

用来判断一个连接是正常还是断开的

### 为什么有心跳检测

心跳的目其实通过判断客户端是否存活，从而回收fd，因为fd资源是有限的，所以 必须重复利用

心跳检测作用主要有两个：

- 客户端定时给服务端发送点数据，防止连接由于长时间没有通讯而被某些节点的防火墙关闭导致连接断开的情况

- 服务端可以通过心跳来判断客户端是否在线，如果客户端在规定时间内没有发来任何数据，就认为客户端下线。这样就可以检测到客户端下线的事件


心跳检测例子：

服务端代码:

```php
//tcp协议
$server=new Swoole\Server("0.0.0.0",9800);   //创建server对象

$server->set([
    'worker_num'=>1, //设置进程
    'heartbeat_idle_time'=>10,//连接最大的空闲时间
    'heartbeat_check_interval'=>3 //服务器定时检查
]);

//监听事件,连接事件
$server->on('connect',function ($server,$fd){
    echo "新的连接进入:{$fd}".PHP_EOL;
});


//消息发送过来
$server->on('receive',function (swoole_server $server, int $fd, int $reactor_id, string $data){
    echo "消息发送过来:".$fd.PHP_EOL;

    //服务端
    //$server->send($fd,'我是服务端');
});

//消息关闭
$server->on('close',function (){
    echo "消息关闭".PHP_EOL;
});
//服务器开启
$server->start();

```

heartbeat_check_interval： 服务器定时检测在线列表的时间
heartbeat_idle_time：       连接最大的空闲时间 （如果最后一个心跳包的时间与当前时间之差超过这个值，则认为该连接失效）

建议 heartbeat_idle_time 为 heartbeat_check_interval 的两倍多一点。
这个两倍是为了进行容错，允许丢一个包而多一点是考虑到网络的延时。


客户端代码：

```cmd
    $client = new Swoole\Client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_ASYNC);
    
    //连接事件回调（必须注册所有事件）
    $client->on("connect", function(swoole_client $cli){
        $cli->send("GET / HTTP/1.1\r\n\r\n");
        //sleep(5);
    });
    
    //异步回调客户端
    $client->on("receive", function(swoole_client $cli, $data){
          echo "Receive: $data";
         //$cli->send(str_repeat('A', 100)."\n");
    
    });
    
    $client->on("error", function(swoole_client $cli){
        echo "error\n";
    });
    
    
    $client->on("close", function(swoole_client $cli){
          echo "Connection close\n";
    });
    
    $client->connect('127.0.0.1', 9800) || exit("");
    
    //定时器,保持长连接
    swoole_timer_tick(9000,function () use($client){
         $client->send('1');
    });

```

这里使用了定时器在9秒的时候发送一个1，保持心跳，所以客户端连接不会关闭，如果没有这个定时器发送，过10秒，客户端连接就会直接关闭了。









