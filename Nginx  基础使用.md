![1676251873066](E:\nginx\图片\1676251873066.png)



## 1、目录结构



进入Nginx的主目录我们可以看到这些文件夹

```
client_body_temp 
conf 
fastcgi_temp 
html 
logs 
proxy_temp 
sbin 
scgi_temp 
uwsgi_temp
```



其中这几个文件夹在刚安装后是没有的，主要用来存放运行过程中的临时文件

```
client_body_temp 
fastcgi_temp 
proxy_temp 
scgi_temp
```



### 1) conf 

用来存放配置文件相关

### 2) html 

用来存放静态文件的默认目录 html、css等

### 3) sbin

nginx的主程序



## 2、基本运行原理



![1676253180762](E:\nginx\图片\1676253180762.png)



Nginx分为master/workers结构，一个master主进程，负责管理和维护多个worker进程，真正处理用户请求的是worker进程，master不对用户请求进行处理。master主进程负责分析并加载配置文件，管理worker进程，接收用户信号传递及平滑升级等功能。nginx具有强大的缓存功能，其中Cache Loader负责载入缓存对象，Cache Manager负责管理缓存对象

补充：默认启动nginx是不会产生 Cache Loader 和 Cache Manager 进程的，此两个进程负责将后端的一些数据缓存到本地磁盘或内存中，用户访问进来时，如果在缓存有效期内则会直接将本地的缓存内容交给用户，不需要再从后端读取，提升nginx响应速度



## 3、Nginx配置与应用场景

### 1) nginx默认配置文件示例

```
user nginx;									# (1) 进程所属用户 
worker_processes auto;						#开启worker进程数量，默认auto表示有多少CPU则设置多少个worker进程
error_log /var/log/nginx/error.log;			#错误日志，针对不同的vhost或uri可以定义不同的错误日志
pid /run/nginx.pid;							# (2) nginx主进程编号
include /usr/share/nginx/modules/*.conf;	#包含nginx的模块配置文件
events {									#用于定义事件驱动配置，此配置于连接的处理密切相关
    worker_connections 1024;					#每个worker进程可以建立的连接数，默认为1024
}

#以上属于主配置段
#----------------------------------------------------------------------------------------
#以下为http配置段

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile            on;					# (3) 使用linux的 sendfile(socket, file, len) 高效网络传输，也就是数据0拷贝。
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;
    
    include             /etc/nginx/mime.types;   #include引用指定文件，mime.types这个文件规定了 相关的
    default_type        application/octet-stream;
    
    include /etc/nginx/conf.d/*.conf;
    
    server {									# (4) 虚拟主机 vhost，一个server代表一个主机			
        listen       80 default_server;			# listen监听 
        listen       [::]:80 default_server;
        
        server_name  _;							# (5) 设置主机的域名、主机名
        root         /usr/share/nginx/html;
        include /etc/nginx/default.d/*.conf;
        location / {
        }
        
        error_page 404 /404.html;
        location = /404.html {
        }
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
    
}
```



#### 主配置段

这些配置为nginx的核心配置，也被称为main配置，具体的参数可参考[官方文档](http://nginx.org/en/docs/)或nginx的[中文文档](https://www.nginx.cn/doc/)，这涉及到nginx的性能优化，主配置段的配置对下面所有段都生效



**(1) user nginx;**

在linux系统下，root用户可以监听任何端口，而普通用户只能监听1024以上的端口，为了使nginx服务能够监听80端口，nginx的master进程以root用户进程来运行，而worker进程以nginx用户进程运行，master进程收到用户访问请求后会将请求交给worker进程处理，所以用户的请求能否处理取决于worker进程的用户权限是否足够，而是否能监听某端口则是由master进程的用户权限决定的

> 补充：验证worker进程，可通过修改html文件的权限为600，只允许root用户读，再次访问nginx时http的状态码就变成了403，而将html文件的所属用户和组更改为nginx后能够再次访问



(2) pid /run/nginx.pid;**

通过 nginx -s 选项向nginx发信号 时，需要通过此文件找到nginx的master进程的进程号，然后向进程号发送信号，如果此文件被移动或修改权限，向nginx传递信号时则会因为找不到此文件而报错。使用systemctl工具时也与nginx -s一致

nginx信号报错示例：

```shell
[root@loadbase1 run]# nginx -s stop
nginx: [error] open() "/run/nginx.pid" failed (2: No such file or directory)
```





#### http模块配置段

除去一些配置参数，http简单的架构示例：

```shell
http{
	...
	server{				#用于配置vhost虚拟主机，可基于IP、PORT或域名的方式配置vhost
		server_name	centos.example.com;
		...
		location / {	#在一个server段内可配置多个location段，此字段用于配置vhost不同的uri响应方式
		...
		}
	}
}
```



**(3) sendfile on;** 

​	使用linux的 sendfile(socket, file, len) 高效网络传输，也就是数据0拷贝。

- 未开启sendfile ：

	比如 用户想要读取一个文件， 用户请求打到nginx上，nginx就要将这个文件 read进来 再write到应用程序内存里，再通过应用程序内存读出这个文件 复制给 计算机操作系统的网络接口（网卡的驱动程序，在里面还会经历 dms的调度、网卡驱动程序的缓存 以及我们内核的缓存）

	**整个过程经历几个 Read and write 的过程**

	![1676255808385](E:\nginx\图片\1676255808385.png)



- 开启sendfile ：

	用户请求打到nginx上，nginx发送一个信号给操作系统的网络接口（ 信号就是 sendfile()方法，这个方法的参数会传递 socket、文件描述符等信息，这些信息都会发送给 网络接口），通过网络接口来读取这个文件，再通过网络就直接发送给用户了

	**就省去了几个 Read and write 的过程**

	![1676255863991](E:\nginx\图片\1676255863991.png)





**(4) 虚拟主机** 

原本一台服务器只能对应一个站点，通过虚拟主机技术可以虚拟化成多个站点同时对外提供服务

![1676259297969](E:\nginx\图片\1676259297969.png)

```
server {

    listen 80; 								#监听端口号
    
    server_name localhost; 					#主机名
    
    location / { 							#匹配路径（/是相对路径 为nginx安装目录 /usr/local/nginx/）
    	root html; 							#文件根目录 （/usr/local/nginx/htnml/）
    	index index.html index.htm;	 		#默认页名称
	}

	error_page 500 502 503 504 /50x.html; 	#报错编码对应页面
	location = /50x.html {
    	root html;
    }
}
```



**(5) server_name 设置主机的域名、主机名**

server_name 匹配规则：

- 完整匹配

	我们可以在同一servername中匹配多个域名

	```
	server_name vod.mmban.com www1.mmban.com;
	```

- 通配符匹配

	```
	server_name *.mmban.com
	```

- 通配符结束匹配

	```
	server_name vod.*;
	```

- 正则匹配

	```
	server_name ~^[0-9]+\.mmban\.com$;
	```



#### 配置虚拟主机vhost



nginx配置虚拟主机(vhost)有三种方式：基于IP配置vhost、基于端口配置vhost、基于域名配置vhost



基于**IP**配置vhost

```shell
[root@centos ~]# vim /etc/nginx/conf.d/ip.conf
server {
    listen 192.168.42.128;
    root /srv/nginx/ip/42;
    index index.html;
}

server {
    listen 192.168.179.128;
    root /srv/nginx/ip/128;
    index index.html;
}
#后续再创建root目录及index.html文件即可
```



基于**端口**配置vhost

```shell
[root@centos ~]# vim /etc/nginx/conf.d/port.conf
server {
    listen          81;
    root            /srv/nginx/port/81;
}

server {
    listen          82;
    root            /srv/nginx/port/82;
}
#此配置文件中省略了index参数，默认读取index.html文件。没有指定IP则表示本机所有IP的81/82端口都能访问
```



基于**域名**配置vhost

```shell
[root@centos ~]# vim /etc/nginx/conf.d/domain.conf
server {
    listen          80;
    server_name     centos.example1.com example1.com *.example1.com;
    root            /srv/nginx/domain/1;
}

server {
    listen          80;
    server_name     centos.example2.com example2.com *.example2.com;
    root            /srv/nginx/domain/2;
}
#server_name选项可以有多个参数，可使用正则表达式，但不建议使用，会降低nginx的性能
#使用域名vhost之前要先将ip.conf屏蔽，否则nginx会直接解析到IP，不会再解析域名
```



### 2) 虚拟机主机实战介绍

#### (1) 域名、dns、ip地址的关系 

因为在网络上机器彼此连接只能互相识别IP，而数字标识较难记忆，所以才演化出域名来代替IP地址，当**我们将在地址栏输入域名欲跳转到某个页面时，点击提交后会由专门的域名解析服务器（DNS服务器）对我们的域名进行解析**，得出域名对应的IP地址再进行连接。所以如果我们直接在地址栏输入与域名对应的IP也可以跳转到同一个页面。

虽然同一个域名只能绑定一个IP地址，**但是因为一个域名可以设定多个DNS服务或者服务器进行解析，同一个域名的每个解析就可以指向不同的IP地址**，**这样应答快的DNS优先进行解析**，能保证最快定向到指定的网站空间去，这是用户所不知道的。

**总结:**

- 地址栏输入域名后，会由专门的域名解析服务器

- 同一个域名只能绑定一个IP地址

- 一个域名可以设定多个DNS服务或者服务器进行解析，同一个域名的每个解析就可以指向不同的IP地址

- 应答快的DNS优先进行解析

	

**DNS**: 域名和IP地址相互映射的一个分布式数据库，能够使用户更方便地访问互联网，而不用去记住能够被机器直接读取的IP数串。



#### (2) 浏览器、Nginx与http协议

![img](E:\nginx\图片\3c77fac147e649a4be81e2da7d412f21.png)



- **TCP协议**：tcp是一个广泛的浏览器协议,它是以流的形式,进行传递数据的(数据是二进制的)

	（数据流：相当于一个水龙头,开启之后一直流通,没有关掉的动作）

- **HTTP协议**：是一个在TCP之上的协议,它会在里面告诉TCP协议什么时候关掉流,当用户的浏览器和Nginx服务器都遵守和实现了HTTP协议之后,他们之间就可以进行信息的交流、传递了。

- **HTTPS协议**：HTTP之上的一个协议,加了一层**数据加密**的措施,保护数据。

- **请求流程**：用户浏览器发送请求---> 网关(层层网关)--->--->互联网--->Nginx服务器--->解析请求--->找到资源--->返回给用户




#### (3) 虚拟主机原理

如果有这样一个场景,假如一台主机只挂载了一个站点,当这个站点并没有太多的访问量时,就会造成资源过剩(有剩余资源),这时我们可以开启虚拟主机,挂载多个站点,合理的利用主机的资源。

一个IP地址可以对应多个域名,根据域名的不同,我们去寻找这些域名对应的资源目录,找到这些资源之后,返回给用户。

当然,我们**需要在请求报文中加上这个域名**,不然服务器不知道我们需要哪个域名的资源

![img](E:\nginx\图片\3c0eb57ea00d42d1b18477fb0147e215.png)





#### (4) 域名解析与泛域名解析实战

1、配置本机的域名解析 （只能在本机访问）

​	在 C:\Windows\System32\drivers\etc\hosts.file 文件中配置：

​	![1676281935916](E:\nginx\图片\1676281935916.png)



2、公网域名配置



阿里云购买一个域名，配置该域名解析的ip地址

![1676339956362](E:\nginx\图片\1676339956362.png)



#### (5) 域名解析相关企业项目实战技术架构



多用户二级域名

短网址

httpdns



#### (6) Nginx中的虚拟主机配置



在 /www 目录下创建2个站点目录资源 （www 和 vod）



1、创建两个文件夹 www 和 vod 

![1676286989151](E:\nginx\图片\1676286989151.png)



2、在 www 中创建一个 index.html 文件 （访问这个站点的默认页面）

![1676287750475](E:\nginx\图片\1676287750475.png)

![1676287323267](E:\nginx\图片\1676287323267.png)



3、在 vod 中创建一个 index.html 文件 （访问这个站点的默认页面）

![1676288978393](E:\nginx\图片\1676288978393.png)



4、配置nginx虚拟主机（配置多个server，访问不同站点（不同目录的资源））

编辑 /usr/local/nginx/nginx.conf 文件

```
...

http {

	...
	
	#虚拟主机vhost (站点1)
    server {
        listen       80;
	
		#域名、主机名
        server_name  www.bigwu.love; 

        location / {
            root   /www/www;
            index  index.html index.htm;
        }
    
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }

    #虚拟主机vhost (站点2)
    server {

        listen       88;
	
		#域名、主机名
        server_name  vod.bigwu.love; 

        location / {
            root   /www/vod;
            index  index.html index.htm;
        }
    
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
    
	...
	
}
```

浏览器访问 www.bigwu.love 就会走到第一个server配置中

![1676339899365](E:\nginx\图片\1676339899365.png)

浏览器访问 vod.bigwu.love 就会走到第二个server配置中

![1676339917735](E:\nginx\图片\1676339917735.png)



## 4、反向代理与负载均衡



### 1) 正向代理

用户和外网不能互通，通过 代理服务器（网关）将用户请求发送给外网

![img](E:\nginx\图片\54fb6301c9194acc9974ef03a069dcc1.png)



### 2) 反向代理



用户和nginx是互通的，用户和应用服务器是不互通的，用户发送请求到nginx，nginx作为代理将请求发送给应用服务器（如tomcat等），tomcat接受nginx的请求处理好后将结果发送给nginx，nginx将结果发送给用户



**反向代理和正向代理的区别**是代理服务器的提供方不同，对于反向代理，nginx是由服务提供方提供的，对于正向代理，代理服务器是由用户自己提供的



#### (1) 隧道式代理

如下图，是隧道式代理（瓶颈在于nginx代理服务器，即使应用服务器的带宽很大，但是如果nginx代理服务器的带宽很小，那么请求都会阻塞到代理服务器）

![image-20220502173846436](E:\nginx\图片\image-20220502173846436.png)

#### (2) 解决隧道式代理的问题

使用DR模型：用户发送请求到代理服务器，代理服务器正常将请求发送给应用服务器，在应用服务器处理完请求后直接将结果数据发送给用户，不经过代理服务器，（比如用户需要下载一个500m的数据包，发送一个 get请求给nginx（1k大小），在发送给应用服务器，应用服务器处理完请求之后 将文件（500m大小） 直接发送给用户，不经过nginx，就不会被低带宽的代理服务器阻塞）



LVS负载均衡器既可以使用DR模型，也可以使用隧道式代理模型



#### (3) 反向代理在企业中的应用

用户使用电脑向目标主机发送请求，请求会先经过路由器，由路由器转发给DNS等服务器进行域名解析，然后数据到达目标服务器所处的机房网关，在机房网关内如果合法能够通过防火墙，进入Nginx服务器使用Nginx服务器进行负载均衡打到相应的业务服务器上即可，业务服务器通常会有一个专门进行身份校验。


**传统公司系统架构：**

![img](E:\nginx\图片\035ea9600328468ebd70e246f9219eb7.png)







#### (4) Nginx反向代理配置



准备工作：基于原虚拟机（192.168.200.130）再克隆2个虚拟机（192.168.200.131 ,192.168.200.132）



##### 1、反向代理到外网的主机配置

再原虚拟机上演示（192.168.200.130）

**nginx.conf配置文件：**

- 启用proxy_pass，root和index字段就会失效

- proxy_pass后的地址必须写完整 `http://www.atguigu.com;`，不支持https

- 当访问localhost时（Nginx服务器），网页打开的是`http://www.atguigu.com;`（应用服务器），网页地址栏写的还是localhost

	```shell
	http{ 	
	
		server {
	    	
	    	listen       80;
	        server_name  localhost;
	
			location / { 
	        	proxy_pass http://www.atguigu.com;
	            #root   html/test; 
	            #index  index.html index.htm;
	      	}
	
			error_page   500 502 503 504  /50x.html;
	        location = /50x.html {
	        	root   html;
	        }
		}
		
	}
	```



访问浏览器：localhost（192.168.200.130、www.bigwu.love）

![1676343026315](E:\nginx\图片\1676343026315.png)





##### 2、反向代理到内网的主机配置

在主机 192.168.200.130 上修改 **nginx.conf配置文件：**

	http{ 
	
		server {
		
			listen       80;
	    	server_name  localhost;
	
			location / { 
	            proxy_pass http://192.168.200.131;
	            #root   html/test; 
	            #index  index.html index.htm;
	  		}
	
	        error_page   500 502 503 504  /50x.html;
	        location = /50x.html {
	            root   html;
	      	}
	  	}
	  	
	}

当访问192.168.200.130（www.bigwu.love）

网页打开的是`http://192.168.200.131;`（应用服务器），网页地址栏写的还是localhost	

![1676344098002](E:\nginx\图片\1676344098002.png)



### 3) 负载均衡

把请求，按照一定算法规则(轮询)，分配给多台业务服务器（即使其中一个坏了/维护升级，还有其他服务器可以继续提供服务）

![image-20220502174023144](E:\nginx\图片\image-20220502174023144.png)







### 4) 基于反向代理的负载均衡



在原主机的 nginx.conf配置文件 中使用upstream定义一组地址【在server字段下】

访问localhost，访问都会代理到`192.168.200.131:80`和`192.168.200.132:80`这两个地址之一，每次访问这两个地址轮着切换（后面讲到，因为默认权重相等）

```
http{

	upstream httpds{
		server 192.168.200.131:80; #如果是80端口，可以省略不写
		server 192.168.200.132:80;
	}
	
	server {
        listen       80;
        server_name  localhost;

        location / { 
        		proxy_pass http://httpds;  #定义地址别名
        }

        error_page   500 502 503 504  /50x.html; 
        location = /50x.html {
            root   html;
        }
	}
}
```



#### (1) 负载均衡策略



**1、weight：权重**

访问使用哪个地址的权重

```text
upstream httpds{
	server 192.168.200.131:80 weight=10;
    server 192.168.200.132:80 weight=80;
}
```

**2、down : 当前server暂不参与负载均衡**

```
upstream httpds{
	server 192.168.200.131:80 weight=10 down;
    server 192.168.200.132:80 weight=80;
}
```

**3、backup : 预留的备份服务器； 其它所有的非backup机器down或者忙的时候，请求backup机器。**

如果`192.168.174.131:80`出现故障，无法提供服务，就用使用backup的这个机器

```
upstream httpds{
		server 192.168.200.131:80 weight=10;
		server 192.168.200.132:80 weight=80 backup;
}
```

**4、max_fails : 请求失败次数限制**

**5、fail_timeout : 经过max_fails后服务暂停时间**

**6、max_conns : 限制最大的连接数**





#### (2) 负载均衡调度算法

##### 1、轮询

​	默认算法按时间顺序逐一分配到不同的后端服务器;

- RR（默认轮询）每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉能自动剔除

- 加权轮询:Weight值越大，分配到访问几率越高（但不是百分百按比例）;

	```
	upstream httpds{
			server 192.168.200.131:80 weight=10;
			server 192.168.200.132:80 weight=80;
	}
	```

	

**缺陷：无法保持会话**

上述2种轮询方式都有一个问题，那就是下一个请求来的时候请求可能分发到另外一个服务器，当我们的程序不是无状态的时候（采用了session保存数据），这时候就有一个很大的很问题了，比如把登录信息保存到了session中，那么跳转到另外一台服务器的时候就需要重新登录了，所以很多时候我们需要一个客户只访问一个服务器，那么就需要用ip_hash了



**适用场景：**

权重模式适合后端服务器性能不均的场景



##### 2、ip_hash 

​	为每一个请求访问的IP的hash结果分配，可以将来自一个IP的固定访问一个后端服务器;

- ip_hash 会话粘连，iphash的每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。

- 会话粘粘可以理解为用户持续访问一个后端机器

	```
	upstream httpds {
		ip_hash;
		server 192.168.200.131:80;
	    server 192.168.200.132:80;
	}
	```

- 当用户的ip发生改变了，就会改变之前持续访问的后端服务器，session会话中断



**优点：**可以使同一个客户端的请求发送到同一台服务器上，避免了权重，轮询无法适用会话保持的的需求

**缺陷**：可能会出现单台服务器的压力陡增，而其他服务器空闲的不均衡的情况

**适用场景：**适合保证同一客户端请求始终被分配到同一台服务器上的场景。



##### 3、least_conn 

​	将请求分配到连接数最少的服务上

- 这种策略就是为了使 负载更加均衡 

	```
	upstream dalaoyang-server {
		least_conn;
		server 192.168.200.131:80;
	    server 192.168.200.132:80;
	}
	```



**不合理的地方**：

比如两个服务器 ，服务器1配置比较高，权重配置为80，服务器2配置比较低 所以只给他 权重配10，这样就会让服务器1的用户访问量远高于服务器2（这样是合理的）。如果你一旦配置了 least_conn 就会然后面的用户都去访问 连接数少的服务器2。可能导致服务器2就负载过大，吃不消



**使用场景：**此负载均衡策略适合请求处理时间长短不一造成服务器过载的情况。

有些请求占用的时间很长，会导致其所在的后端负载较高。这种情况下，least_conn这种方式就可以达到更好的[负载均衡](https://so.csdn.net/so/search?q=负载均衡&spm=1001.2101.3001.7020)效果。





##### 4、url_hash

​	需要安装模块安装访问的URL的hash结果来分配，这样每个**URL定向**到同一个后端服务器

- 按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。

- 在upstream中加入hash语句，server语句中不能写入weight等其他的参数，hash_method是使用的hash算法

- nginx默认不支持这种调度算法，要使用的话需要安装nginx的hash软件包

	```
	upstream httpds {
		hash $request_uri;
		hash_method crc32;
		server 192.168.200.131:80;
	    server 192.168.200.132:80;
	}
	```

- **不能保持会话**

	比如说用户注册账号 访问 http://atguigu/register，根据这个url计算的hash值 定向到一台服务器上 访问这台服务器

	注册完了之后登录 访问 http://atguigu/login，又根据这个url计算的hash值 定向到了另外一台服务器上 访问另一台服务器



**优点**：使用了DNS分流，成本较低，而且dns性能高 不用维护



**缺点**：

- 可用性方面：如果有一台机器宕机，则指向这台机器的请求无法读取

- 分流方面：只能全部同步，成本较高

- 不能保持会话



**适用场景**：只适用于面向用户的系统

​	**用户需要去访问 固定资源的时候，而不是维持会话**

​	比如 我们有100个文件资源 他们散落在 5台服务器上（每台20个），此时用户需要去访问其中某个资源，就必须去指定服务器上找

​	此时就可以根据url定位到某台服务器 找到该资源



##### 5、fair

​	按后端服务器 的响应时间来分配请求，响应时间短的优先分配

- 智能调整调度算法，动态的根据后端服务器的请求处理到响应的时间进行均衡分配，响应时间短处理效率高的服务器分配到请求的概率高，响应时间长处理效率低的服务器分配到的请求少；

- 需要注意的是nginx默认不支持fair算法，如果要使用这种调度算法，请安装upstream_fair模块

	```
	upstream httpds {
		fair;
		server 192.168.200.131:80;
	    server 192.168.200.132:80;
	}
	```



​	**缺点：可能导致流量倾斜**



##### 6、总结 

以上几种调度算法都有各种的优缺点，在实际企业中 **RR轮询的方式是最常用的**

解决无法保持会话 可以通过 **基于服务端的session会话共享（NFS，MySQL，memcache，redis，file）**

> **session共享的方法**
>
> 1.把多台后端服务器session文件目录挂载到NFS同一目录
>
> 2.通过程序将session存储到mysql数据库 
>
> ​	一般不推荐，这样会给数据库带来大量的读写请求，应用服务器是负载均衡了，可是数据库这样就炸了
>
> 3.通过程序将session存储到redis、Memcached缓存
>
> ​	Redis和Memcached是把数据存放到内存中，一般不做持久化，IO速度要快于普通数据库，并且Redis比较容易做集群，可以防止单点故障，	  	是目前比较流行的一种方法 （redis对JAVA支持比较好、Memcached在PHP中有官方支持）
>
> 
>
> 最推荐使用redis  







## 5、动静分离



### 1) 什么是动静分离

动静分离是指在web服务器架构中，将静态页面与动态页面或者静态内容接口和动态内容接口分开不同系统访问的架构设计方法，进而提升整个服务访问性能和可维护性。

**nginx的动静分离指的是：**

由 nginx将客户端请求进行分类转发，静态资源请求（如html、css、图片等）由静态资源服务器处理，动态资源请求（如 jsp页面、servlet程序等）由 tomcat 服务器处理，tomcat 本身是用来处理动态资源的，同时 tomcat 也能处理静态资源，但是 tomcat 本身处理静态资源的效率并不高，而且还会带来额外的资源开销。利用 nginx 实现动静分离的架构，能够让 tomcat 专注于处理动态资源，静态资源统一由静态资源服务器处理，从而提升整个服务系统的性能 。
![1676358292935](E:\nginx\图片\1676358292935.png)





### 2) nginx动静分离配置

​	让 nginx服务器（192.168.200.130 的 /usr/local/nginx 目录）作为 静态资源服务器 直接处理静态资源请求，

​	动态请求 由这个nginx服务器打到 192.168.200.130:8080（tomcat服务器）上，让tomcat服务器处理 



(1) 准备工作

① 在 服务器（192.168.200.130） `/usr/local/nginx` 目录下新建 static 目录，并在此目录下分别新建 css、js、img、目录，在这些目录中准备好静态资源 【del.jpg 、demo.css、demo.js 】

② 在 服务器（192.168.200.131） `/opt/apache-tomcat-8.5.85/webapps/ROOT` 目录下 准备好动态资源  【index.html】





**配置反向代理 （修改 192.168.200.130 的 nginx配置文件）** 



将动态请求 转发给 192.168.200.131:8080

```
location / {
	
	proxy_pass http://192.168.200.131:8080

}
```



**增加多一个location**

指定 静态资源的访问目录

```
location /css {
	root /usr/local/nginx/static;
	index index.html index.htm;
}

location /img {
	root /usr/local/nginx/static;
	index index.html index.htm;
}

location /js {
	root /usr/local/nginx/static;
	index index.html index.htm;
}

```



浏览器访问 192.168.200.130

![1676427432385](E:\nginx\图片\1676427432385.png)





**补充：**

使用`nginx location`可以控制访问网站，一个`server`块中可以有多个`location`配置

多个`location`配置则产生了优先级的区别 ，`location`语法优先级从高到低排列：

| 匹配符号 |           匹配规则           | 优先级 |
| :------: | :--------------------------: | :----: |
|    =     |           精确匹配           |   1    |
|    ^~    |        以某字符串开头        |   2    |
|    ~     |     区分大小写的正则匹配     |   3    |
|    ~*    |    不区分大小写的正则匹配    |   4    |
|    !~    |     区分大小写的正则取反     |   5    |
|   !~*    |    不区分大小写的正则取反    |   6    |
|    /     | 通用匹配，任何请求都会匹配到 |   7    |
|          |                              |        |



**将上面配置的多个location 合并为一个location** 

location匹配顺序

- 多个正则location直接按书写顺序匹配，成功后就不会继续往后面匹配
- 普通（非正则）location会一直往下，直到找到匹配度最高的（最大前缀匹配）
- 当普通location与正则location同时存在，如果正则匹配成功,则不会再执行普通匹配
- 所有类型location存在时，“=”匹配 > “^~”匹配 > 正则匹配 > 普通（最大前缀匹配）

```
location ~*/(css|img|js) {
	root /usr/local/nginx/static;
	index index.html index.htm;
}
```





**alias与root**

```
location /css {
	alias /usr/local/nginx/static/css;
	index index.html index.htm;
}
```

root用来设置根目录，而alias在接受请求的时候在路径上不会加上location。

- 1）alias指定的目录是准确的，即location匹配访问的path目录下的文件直接是在alias目录下查找的； 
- 2）root指定的目录是location匹配访问的path目录的上一级目录,这个path目录一定要是真实存在root指定目录下的； 
- 3）使用alias标签的目录块中不能使用rewrite的break（具体原因不明）；另外，alias指定的目录后面必须要加上"/"符号！！ 
- 4）alias虚拟目录配置中，location匹配的path目录如果后面不带"/"，那么访问的url地址中这个path目录后面加不加"/"不影响访问，访问时它会自动加上"/"； 但是如果location匹配的path目录后面加上"/"，那么访问的url地址中这个path目录必须要加上"/"，访问时它不会自动加上"/" 。如果不加上"/"，访问就会失败！ 
- 5）root目录配置中，location匹配的path目录后面带不带"/"，都不会影响访问。



## UrlRewrite（伪静态）

rewrite是实现URL重写的关键指令，根据 regex (正则表达式)部分内容，重定向到replacement，结尾是flag标记。



**rewrite**语法格式及参数语法:

```
rewrite <regex> <replacement> [flag];
关键字    	正则	  替代内容     flagt标记
		
		关键字：其中关键字error_log不能改变
		正则：perl兼容正则表达式语句进行规则匹配
		替代内容：将正则匹配的内容替换成replacement
		flag标记：rewrite支持的flag标记

flag标记说明：

	last 		#本条规则匹配完成后，继续向下匹配新的location URI规则
	break 		#本条规则匹配完成即终止，不再匹配后面的任何规则
	redirect 	#返回302临时重定向，浏览器地址会显示跳转后的URL地址
	permanent 	#返回301永久重定向，浏览器地址栏会显示跳转后的URL地址
```



浏览器地址栏访问 `xxx/123.html`实际上是访问`xxx/index.jsp?pageNum=123`

```
server {
        listen       80;
        server_name  localhost;
				
		location / { 
				
			rewrite ^/([0-9]+).html$ /index.jsp?pageNum=$1 break;
        	proxy_pass http://xxx;
        
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root html;
        }
}
```



演示

```
location / {
	rewrite ^/([0-9]+).html$	 /indexjsp?pageNum=$1 redirect;
	proxy_pass http://192.168.200.130:8080; 
	
}
```





## 网关服务器

![image-20220503121135171](E:\nginx\图片\image-20220503121135171.png)

上图中，应用服务器，不能直接被外网访问到，只能通过Nginx服务器进行访问（使用proxy_pass），这时候这台Nginx服务器就成为了网关服务器（承担入口的功能）

所以，**我们启动应用服务器的防火墙，设置其只能接受这台Nginx服务器的请求**

```
#1、设置防火墙，外网就无法直接访问 应用服务器了
systemctl start firewalld

#2、添加rich规则，指定端口个ip访问
#这里的 192.168.200.130 是 网关服务器地址（图中的nginx服务器） 
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="192.168.200.130" port protocol="tcp" port="8080" accept"

#3、移除rich规则
firewall-cmd --permanent --remove-rich-rule="rule family="ipv4" source address="192.168.200.130" port port="8080" protocol="tcp" accept"

#4、重启
#移除和添加规则都要重启才能生效
firewall-cmd --reload

#5、查看所有规则
firewall-cmd --list-all #所有开启的规则
```



**伪静态同时配置负载均衡**

```
...

http {

    ...
    
    #配置负载均衡
    upstream httpds {
		server 192.168.200.131:8080 weight=1;
		server 192.168.200.132:8080 weight=8;
	}

    #虚拟主机vhost 
    server {

        listen       80;
	
	#域名、主机名
        server_name  localhost; 

        location / {
			rewrite ^/([0-9]+).html$  /indexjsp?pageNum=$1 redirect;  	#伪静态(url重写)
			proxy_pass http://httpds; 									#使用别名 配置负载均衡
	
		}
	
	...

}
```





## 防盗链配置

### 1、防盗链概念

防盗链简单来说就是存在我们服务中的一些资源，只有我们规定的合法的一类人才能去访问，其他人就不能去访问的资源（如css，js，img等资源）。

​    具体点就是用户发送请求给nginx服务器，nginx服务器根据请求去寻找资源，请求的比如说是有个index.html文件，这个文件中会包含很多js，css，img等资源，这些文件在这个骨架中会被二次请求，在第二次请求时，会在请求头部上加上有个referer，这个referer只会在第二次请求时才会被加上。（referer表示第二次资源的来源地址）



   当我们请求到一个页面后，这个页面一般会再去请求其中的静态资源，这时候请求头中，会有一个refer字段，表示当前这个请求的来源，**我们可以限制指定来源的请求才返回，否则就不返回，这样可以节省资源 ，防盗链**

![img](E:\nginx\图片\350c3ad357b9441eb723a083361f4667.png)



### 2、Nginx防盗链的具体实现



valid_referers： nginx会通过 查看referer 自动和 valid_referers 后面的内容进行匹配，如果匹配到了就将`$invalid_referer`变量置0，如果没有匹配到，则将变量置为1，匹配的过程不区分大小写



```
1、语法：valiad_referers none|blocked|server_names|string

	none：如果header中的referer为空，允许访问
	blocked：在header中的referer不为空，检测 Referer头域的值被防火墙或者代理服务器删除或伪装的情况。这种情况该头域的值不以“http://” 或 “https://” 开头。
	server_names：指定具体的域名或者IP，refer显示是从这个地址来的，则有效（server_name必须是完整的http://xxxx）

2、默认值：-
3、位置（可以书写的地方）：server，location

```



**例子**：这里设置nginx服务器中的img目录下的图片必须refer为http:192.168.200.130才能访问

```
server {
        
        listen       80;
        server_name  localhost;
				
		location / { 
        	proxy_pass http://xxx;
        }
      
		location /img{
		
			#如果请求头部的refer为 http:192.168.200.130，那么就 $invalid_referer 就会被赋值为0，否则为1
        	valid_referers http:192.168.200.130;
                
          	if ($invalid_referer){ 	# 0-不进入 1-进入
            	return 403;			# 返回状态码403
       		}
                
            root html;
            index  index.html index.htm;
        
        }
        
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
}
```





**设置盗链图片**



![image-20220503153401325](E:\nginx\图片\image-20220503153401325.png)

将提示图片放在 html/img/x.png，访问设置防盗链图片时，就返回这x.png张图

```
location /img{

	valid_referers http:192.168.200.130;
	
    if ($invalid_referer){				# 1-无效的
    	rewrite ^/  /img/x.png break; 	# url重写，访问提示图片
    }
    
    root html;
    index  index.html index.htm;
    
}
```



### 3、使用curl测试防盗链



安装 curl 

```
yum install -y curl
```



使用 curl 测试

```
curl -I http://192.168.200.130
```



带 refer 的测试 

```
curl -e "http://atguigu.com" -I http://192.168.200.130
```







## 高可用配置



### 1、关键词

- KeepAlived（主服务器 和 备份服务器 故障时 IP 瞬间无缝交接）
- VRRP协议（路由器组，提供虚拟IP，一个master和多个backup，组播消息，选举backup当master）
- Nginx+keepalived 双机主从模式（热备服务器）、双机主主模式（俩公网虚拟IP，负载）

![在这里插入图片描述](E:\nginx\图片\watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA56m_5Z-O5aSn6aW8,size_20,color_FFFFFF,t_70,g_se,x_16)



### 2、为什么要使用nginx的高可用

因为nginx作为反向代理服务器时，有可能出现宕机的情况，而由于其反向代理的特性，就会导致应用服务器（tomcat等）无法被访问，这样项目就停止工作了。所以结合**keepalived**对前端nginx实现**HA高可用**

### 3、什么是高可用

nginx的高可用简单来说就是配置了两台(或更多)的nginx服务器，当主服务器宕机时，就会**自动切换**到备用服务器，从而保证项目的持续运行。

### 4、高可用的原理 (双机主从模式)

nginx的实现需要借助其他工具（keepalived）来实现。在 keepalived中 配置一个虚拟IP（VIP），同时keepalived会定时检查主服务器的工作状态（通过脚本实现）。在主服务器正常工作时，VIP就会映射到主服务器的IP，此时，**虚拟ip对应的物理地址和主服务器IP对应的物理地址是相同的**，所以访问虚拟IP即访问主服务器。

当主服务器失效时，通过脚本可以监测到，然后会去停止主服务器的keepalived（其它备用机上配置的 keepalived就会检测到 主服务器中keepalived停止了，认定该主服务器宕机了），从而根据预先的配置，找到优先级最高的备用服务器，并将虚拟IP映射到该备用服务器的ip，此时，这两个ip对应的物理地址时相同的。再主服务器回复正常时，又会被检测到，又会自动切换到主服务器。这样就实现了nginx的高可用。

![在这里插入图片描述](E:\nginx\图片\1_se,x_16)

### 5、高可用配置 (双机主从模式)



1) 安装keepalived软件 （主、从都安装）

```
安装位置：/url/local/keepalived/ 

#使用yum命令安装keepalived
yum install keepalived -y
#检查是否安装完成
rpm -q -a keepalived

```



2) 修改keepalived.conf配置文件 （主、从都修改）

​	添加注释的为重点，其他不关键

```
global_defs {

   router_id lb111 								#起一个名字
   
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_pid.sh"   	# 检查nginx状态的脚本
    interval 2      							# 检查时间间隔
    weight 3    								# 权重
}

vrrp_instance VI_1 {
    state MASTER     							# 主服务器为MASTER、 备用服务器为BACKUP
    interface ens33  							# 对应本机网卡的名字，用ifconfig查看
    virtual_router_id 66   						# 主备机virtual_router_id必须一致
    priority 100       							# 主服务器为100、 备用服务器要小于100 可配置成90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {							
        192.168.11.11    						#配置外网访问的 虚拟IP（vip） ，有多个可在下面继续增加
    }
    track_script {
        chk_nginx
    }
}
```



3) 添加脚本信息

在上面的配置文件中有一行：`script “/etc/keepalived/nginx_pid.sh”` 即脚本位置

```
#转到该目录
cd /etc/keepalived/
#创建文件
touch nginx_pid.sh
#编辑文件
vi nginx_pid.sh
```

脚本内容：

```
#!/bin/bash
A=`ps -C nginx --no-header |wc -l`
if [ $A -eq 0 ];then
     systemctl restart docker    #重启docker容器
      sleep 3
            if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
            #如果重启失败，关闭keepalived
                  systemctl stop keepalived
fi
fi
```

**两个虚拟机中都要添加**



4) 重启两个虚拟机中的nginx和keepalived

```
#重启nginx
docker stop nginx-MASTER
docker start nginx-MASTER
#开启keepalived
systemctl start keepalvied.service
```



5) 测试

使用虚拟IP访问：http://192.168.11.11:8000，即可看到nginx欢迎界面

当关闭主nginx时：

```
docker stop nginx-MASTER
systemctl stop keepalvied.service
```

该访问url依然可以访问，表示高可用实现



**关于url：这里需要添加端口，因为nginx是在docker容器中，在上面也可以看到，80端口映射了宿主机的8000端口，所以要添加端口；**

**由此推导，两个nginx的映射端口也必须一致**，否则无论如何只能访问其中一个nginx



## Https证书配置

### 不安全的http协议

### openssl

### 自签名

OpenSSL

图形化工具 XCA 

### CA 签名

