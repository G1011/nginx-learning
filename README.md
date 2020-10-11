# Nginx整理学习

## 一、什么是Nginx

比如163  F12可以看到服务采用的是nginx，采用反向代理将他们的网页请求分发到公司的多台服务器上

*Nginx* (engine x) 是一个高性能的[HTTP](https://baike.baidu.com/item/HTTP)和[反向代理](https://baike.baidu.com/item/反向代理/7793488)web服务器，同时也提供了IMAP/POP3/SMTP服务。

Nginx: http://nginx.org/

原生nginx没有集成很多的插件，不方便，可以采用OpenResty，它其实就是nginx，只是采用了大量的第三方插件、库、模块等

OpenResty: http://openresty.org/en/



### 1.1 下载配置启动

#### 1.1.1启动

命令行：

```shell
nginx.exe

nginx的启动和关闭
nginx -h 查看帮助信息
nginx -v 查看Nginx的版本号
nginx -V 显示Nginx的版本号和编译信息
start nginx 启动Nginx
nginx -s stop 快速停止和关闭Nginx
nginx -s quit 正常停止或关闭Nginx
nginx -s reload 配置文件修改重新加载
nginx -t 测试Nginx配置文件的正确性及配置文件的详细信息
task /fi "imagename eq nginx.exe" windows命令框下查看nginx的进程命令
```

访问本地80端口（默认）

- localhost

- 服务起来了
- 修改nginx.conf文件

修改：

```nginx
http {
  default_type application/octet-stream; #数据流
  server {
    listen 80;
    server_name localhost;
    default_type text/html; #根据server修改
    # / 默认匹配所有的请求
    location / {
      echo "hello nginx"; 
    }
  }
}
```

重启指令 nginx.exe -s reload

其中-s 代表输入一些指令

访问会发现在下载 原因：default_type类型是一些数据流

修改 default_type text/html;

#### 1.1.1.2 location匹配级别

```nginx
http {
  default_type application/octet-stream; #数据流
  server {
    listen 80;
    server_name localhost;
    default_type text/html; #根据server修改
    # / 默认匹配所有的请求
    location / {
      echo "hello nginx"; 
    }
    # = 匹配级别最高（完全匹配）
    location = /a {
      echo "=/a"; 
    }
    
    # ^~ 以..开头 匹配级别次高（完全匹配）
    # 以/a开头
    location ^~ /a {
      echo "^~/a"; 
    }
    # 匹配程度或者长度越高 匹配级别越高
    location ^~ /a/b {
      echo "^~/a/b"; 
    }
    
    # ~  正则表达式
    location ~ /\w {
      echo "^~/\w"; 
    }
    # 正则 当匹配程度相同 优先级别也相同，则以写在上面的为优先
    location ~ ^/[a~z] {
      echo "~ ^/[a~z]"; 
    }
  }
}
```



## 二、正向代理与反向代理

文件：nginx.conf

缓存问题：刷新 或者F5 + 刷新 （强制刷新）

**注意：编写过程注意空格，空格错误将会报错**

```nginx
	#集群的声明  backserver集群名称
	upstream backserver{
		server 127.0.0.1:8081 weight=1; #第一个服务器地址 配置权重
		server 127.0.0.1:8082 weight=2; #第二个服务器地址 配置权重
	}
	
	server {
	 listen 80; #默认端口为8080
	 server_name localhost;
	 
	 location / {
	 	proxy_pass http://backserver; #接收到请求时，进入集群中选择一个服务器访问
	 													 #backserver就是 上面集群中声明名字
    #proxy_pass http://192.168.0.12:80; #这是一个apach的服务
	 	root html;
	 	index index.html index.htm;
	 }
  
  #将路径配置在proxy_pass 的/路径下 配置技巧:
  #早location后面加"/" 同时 在proxy_pass后面的地址也加"/"
  #此时http://192.168.0.12:80/a/ 能够正常访问
  #注意：如果访问http://192.168.0.12:80/a/a无法访问，因为他是从后面的/a开始转发的
  location /a/ {
    proxy_pass http://192.168.0.12:80/; #这是一个apach的服务
   
  }
}
```



## 三、负载均衡

###3.1、负载均衡的策略



#### 3.1.1 轮询

顾名思义，轮番询问（默认情况），每个用户请求相对均匀的发送到不同的Tomcat中，比较适合使用在所有Tomcat服务器**性能一致**的时候使用

若性能不一致 容易造成请求的堆积

```nginx
upstream backserver{
	server 127.0.0.1:8081;
	server 127.0.0.1:8082;
}
```



####3.1.2 权重

每个请求按照一定比例分发到不同的后端服务器，weight值越大访问的比例就越大，用于后端服务器性**能不均匀**的情况

```nginx
upstream backserver{
	server 127.0.0.1:8081 weight=5;
	server 127.0.0.1:8082 weight=2;
}
```

举例 权重如果是5：2 一般情况下是：

81接受2个 -> 82接受1个 -> 81接受2个 -> 82接受1个 -> 81接受1个

#### 3.1.3 最少连接

web请求会被转发到连接数量最少的服务器上

nginx去判断谁的请求数量少，就分发到该服务器上

```nginx
upstream backserver{
	least_conn;
	server 127.0.0.1:8081;
	server 127.0.0.1:8082;
}
```





**轮询、权重、最少连接 这三种负载均衡策略都会造成一个问题：**

用户第一次访问 8081的Tomcat，用户第二次访问8082的Tomcat，第三次继续访问8081的Tomcat，以此类推.... 用户不停的切换服务器。

**这样造成的问题是：session丢失**



为了解决这种问题，提供了一种叫ip_hash的负载策略

#### 3.1.4 ip_hash策略

Ip_hash也叫做IP绑定，每个请求按访问ip的hash值分配，这样每个访问客户端会固定访问一个后端服务器，可以解决会话Session丢失的问题



IPv4会考虑前3个octet，IPv6会考虑所有的地址位



根据用户的IP计算哈希值，根据哈希值求模，求完摸以后将请求分发到不同服务器上去

**算法：hash("124.207.55.82") % 2 = 0, 1**

```nginx
upstream backserver{
	ip_hash;
	server 127.0.0.1:8081;
	server 127.0.0.1:8082;
}
```

深入剖析：

![image-20201006001540141](/Users/g/Library/Application Support/typora-user-images/image-20201006001540141.png)



Ip_hash存在的问题：

核心算法实现：

```c++
for (i = 0; i < 3; i++) {
  hash = (hash * 113 + iphp->addr[i])%6271;
}
p = hash % iphp->rrp.peers->number;
```

可以看到，hash值既与ip有关又与后端机器的数量有关。经测试，上述算法可以连续产生1045个互异的value，这是此算法硬限制。nginx使用了保护机制，当经过20次hash仍然找不到可用的机器时，算法退化成轮询。

因此，**从本质上说，ip hash算法是一种变相的轮询算法，如果两个ip的初始hash值恰好相同，那么来自这两个ip的请求将永远落在同一台服务器上，这为均衡性埋下了较深隐患。**

#### 3.1.5 fair(第三方扩展策略)

按后端服务器的响应时间来分配请求，响应时间短的优先分配

fair策略是扩展策略，**默认不被编译进nginx内核**。它根据后端服务器的响应时间判断负载情况，从中选出负载最轻的机器进行分流。

这种策略具有**很强的自适应性**，但是实际的网络环境往往不是那么简单，因此须慎用。

```nginx
upstream backserver{
	server 127.0.0.1:8081;
	server 127.0.0.1:8082;
  fair;
}
```



#### 3.1.6 url_hash(第三方扩展策略)

（扩展策略）按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。



```nginx
upstream backserver{
	server 127.0.0.1:8081;
	server 127.0.0.1:8082;
  hash $request_uri; #实现每个url定向到同一个后端服务器
  hash_method crc32;
}
```









```nginx
proxy_pass http://backserver/;
upstream backserver{
	ip_hash;
	server 127.0.0.1:9090 down; #(down 表示单前的server暂时不参与负载)
	server 127.0.0.1:8080 weight=2; #(weight 默认为1.weight越大，负载的权重就越大)
	server 127.0.0.1:6060;
	server 127.0.0.1:7070 backup; #(其它所有的非backup机器down或者忙的时候，请求backup机器)
  server ip or domain:port weight=2 max_fails=3 fail_timeout=15 max_conns=1000; # 使用weight设置权重为20%
}
max_fails: 1; #允许请求失败的次数默认为1.当超过最大次数时，返回proxy_next_upstream 模块定义的错误

fail_timeout:1; #max_fails次失败后，暂停的时间
```





### 3.2负载均衡测试

####3.2.1 对比测试

本测试主要为了对比各个策略的均衡性、一致性、容灾性等，从而分析出其中的差异性，并据此给出各自的适用场景。为了能够全面、客观的测试nginx的负载均衡策略，我们采用了两个测试工具、在不同场景下做测试，以此来降低环境对测试结果造成的影响。首先简单介绍测试工具、测试网络拓扑和基本的测试流程。

####3.2.2测试工具

**easyABC**

easyABC是公司内部开发的性能测试工具，采用epool模型实现，简单易上手，可以模拟GET/POST请求，极限情况下可以提供上万的压力，在公司内部得到了广泛的使用。由于被测试对象为反向代理服务器，因此需要在其后端搭建桩服务器，这里用nginx作为桩webserver，提供最基本的静态文件服务。

**polygraph**

polygraph是一款免费的性能测试工具，以对缓存服务、代理、交换机等方面的测试见长。它有规范的配置语言PGL（Polygraph Language），为软件提供了强大的灵活性。其工作原理如下图所示：

polygraph提供client端和server端，将测试目标nginx放在二者之间，三者之间的网络交互均走http协议，只需配置ip+port即可。client端可以配置虚拟robot的个数以及每个robot发请求的速率，并向代理服务器发起随机的静态文件请求，server端将按照请求的url生成随机大小的静态文件做响应。这也是选用这个测试软件的一个**主要原因**：可以产生随机的url作为nginx各种hash策略的key。

另外，polygraph还提供了日志分析工具，功能比较丰富，感兴趣的同学可以参考附录中的相关材料。

####3.2.3测试环境

####3.2.4测试方案

####3.2.5测试结果





## 四、问题

- 使用Nginx的反向代理，让同一个用户的请求一定转发到同一台服务器上，这种均衡策略会消耗更多的服务器资源，也增加了代理服务器的负担；
- 使用其他策略作为负载均衡时，会出现用户Session丢失的情况，为避免出现这种情况，可以将用户的Session存放到缓存服务器中，比较常用的方案时redis/memchache；
- 反向代理服务器也可以开启缓存服务，但是开启该项服务会增加代理服务器的负担，影响整体的负载均衡效率；
- 使用Nginx反向代理布置负载均衡，操作相对金丹，但是会有“单点故障”的问题，如果后台某台服务器宕机，会带来很多的麻烦，后期如果后台服务器继续增加，反向代理服务器会成为负载均衡方案的瓶颈。
