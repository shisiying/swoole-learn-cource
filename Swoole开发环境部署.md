# Swoole

> swoole是PHP的异步、并行、高性能网络通信引擎，使用纯C语言编写，提供了PHP语言的异步多线程服务器，异步TCP/UDP网络客户端，异步MySQL，异步Redis，数据库连接池，AsyncTask，消息队列，毫秒定时器，异步文件读写，异步DNS查询。 Swoole内置了Http/WebSocket服务器端/客户端、Http2.0服务器端。

## swoole提供的功能库


- 	http服务 ，编写一个简单的web server。
- 	TCP/UDP服务 ，编写一个消息接受处理系统。
- 	异步，可以异步的处理请求。
-	并发 ，可以并发的处理同一个业务逻辑。
-	socket，socket通讯处理技术。
-	毫秒级别定时器，可以在php中使用定时器了。
-	协程，相比线程更稳定和好用。

## 具体的使用场景：

- 互联网
- 移动通信
- 企业软件
- 云计算
- 网络游戏
- 物联网（IOT）
- 车联网
- 智能家居等领域


如果在我们日常的开发任务当中，有用到以上等特性，而且我们又在使用php，那么我们完全可以用swoole来完成了

## Swoole的框架

- [Swoft](https://www.swoft.org/)
- [EasySwoole](http://www.easyswoole.com/)


## Swoole开发环境部署

>安装git、docker和docker-compose。mac和windows用户安装docker便集成了docker-compose,类unix用户需要单独安装docker-compose。

- docker一键安装部署

    - 安装dnmp
    ```cmd
        git clone https://github.com/shisiying/dnmp-swoole.git
    ```
    
    以上我的仓库主要是对[dnmp](https://github.com/yeszao/dnmp)做了简化
    
    - 快速使用
    
  在开发的时候，我们可能经常使用docker exec -it切换到容器中，把常用的做成命令别名是个省事的方法。
  
  打开~/.bashrc,(mac上的是.zshrc)加上：
  
  alias dnginx='docker exec -it dnmp-swoole_nginx_1 /bin/sh'
  alias dphp72='docker exec -it dnmp-swoole_php72_1 /bin/sh'
  alias dphp56='docker exec -it dnmp-swoole_php56_1 /bin/sh'
  alias dmysql='docker exec -it dnmp-swoole_mysql_1 /bin/bash'
  alias dredis='docker exec -it dnmp-swoole_redis_1 /bin/sh'

  然后直接在dnmp-swoole目录执行docker-compose up -d,这里默认你已经安装了docker以及docker-compose

   - 测试
   
   如果你配置好了alias dphp72的命令，那么就可以使用下列命令测试swoole有没有安装好
   
   启动服务端
   ```cmd
        $ dhphp72
        $ cd swoole-project/
        $ php swoole-server.php
   ``` 
   
   新建终端
   启动客户端
   ```cmd
       $ dhphp72
       $ cd swoole-project/
       $ php swoole-client.php
   ```
   
   我们可以在客户端看到：服务器给你发送消息了：hello world，说明环境已经部署成功
   
   
- PhpStorm配置ide，swoole提示工具
    
  安装好之后呢。如果你还需要对你想对你的编辑器，比如：phpstrom 对swoole的代码提示功能，就可以下载帮助文件：https://github.com/swoole/ide-helper
  
  点击setting选择languages 点击+号添加我们下载的文件

    
    
 
    
    


