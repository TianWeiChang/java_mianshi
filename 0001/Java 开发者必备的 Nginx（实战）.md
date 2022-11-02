Java 开发者必备的 Nginx（实战）

说到 Nginx，项目中一般是运维人员在负责这块内容，Java 程序员在开发中反而不怎么会具体使用。程序员知道 Nginx 可以实现反向代理、负载均衡、动静分离等功能，工作中不会实际地去配置这些内容，也不清楚具体是怎么在 Nginx 中实现这些功能。那么通过看这篇文章也许会让你有所收获。

本篇以实例演示， 从以下 10 个方面对 Java 程序员应掌握的 Nginx 知识进行讲解：

1. 常用的 Web 服务器介绍；
2. Nginx 在分布式架构中的作用；
3. Nginx 的下载与安装；
4. Nginx 的配置文件介绍；
5. 实例演示环境介绍（CentOS + Nginx + Tomcat）；
6. Nginx 的反向代理演示；
7. Nginx 的负载均衡演示；
8. Nginx 的动静分离演示；
9. Nginx 解决跨域访问问题演示；
10. Nginx 的防盗链配置演示。

希望通过本场文章的学习，使一线程序员快速了解常用的 Nginx 相关知识点，了解 Nginx 是如何做到反向代理、负载均衡、动静分离、实现防盗链，以及解决跨域问题。

### 一、常用的 Web 服务器介绍

#### 服务器介绍

Web 服务器分为静态服务器和动态服务器，静态服务器就是处理静态资源的，比如 HTML、CSS、JS，常用的静态服务器有 Apache、Nginx；动态服务器就是处理动态请求的，比如 JSP，Servlet 等，常用的有 Tomcat、Weblogic。

Nginx 是一个高性能的 HTTP 服务器和反向代理服务器，能够支持 5 万个并发连接，内存、CPU 消耗非常低，是基于七层协议的服务。

#### 反向代理介绍

我们平时说的代理指的是代理客户端，这个是正向代理，反向代理指的是代理服务端。

我们作为用户想访问一个服务资源 URL，如果我们的浏览器直接打不开这个 URL，一般会通过 VPN 或者其他代理服务器中转，这种情况下的代理就是正向代理，也就是我们通常说的代理的意思。

而反向代理是指，作为服务资源提供方，内部有很多服务器，这些服务器不能全部暴露给第三方用户，因此需要在内部服务器的前面加一个代理服务器，用户访问的是代理服务的 IP，而不知道具体访问的是服务端的哪台机器，这种情况就是反向代理，指的是代理服务端。

### 二、Nginx 在分布式架构中的作用

#### 分布式架构的演进

一个网站的初期，访问的流量比较小，选用的架构可能就是用户通过域名访问，经过域名解析（DNS 服务器），拿到后端服务器的 IP 地址，直接访问到这个 IP 所对应的 Tomcat 服务器，这是最简单的一个网站架构。

随着用户流量的增大，一台 Tomcat 服务器无法满足用户的请求，因此人们会想到 2 个办法：

1. 升级这台服务器更换更强大的硬件，这就是垂直扩展；
2. 增加新的服务器来分担前端流量，这就是水平扩展。

垂直扩展的方式就是对一台服务器不断加强硬件，但是服务器的可扩展内存会有上限，总会达到瓶颈，而且成本比较高，因此人们一般会选择水平扩展。一台不够再加一台，不行再加一台......

为了应对流量的增加，不断地增加后端服务器的数量，那么服务器的 IP 会越来越多，通过 DNS 服务器管理这些服务器 IP 带来了新的问题：

1. 后端的某台机器宕机后，DNS 服务器不知道该机器宕机，仍然解析到了这个 IP，如果用户访问到了这个宕机的 IP，那么系统无法为用户提供服务；
2. DNS 服务器配置新的 IP 后，它不会立即生效，那么在它生效的这个时间段，新加的服务器不会为用户提供服务。

反向代理和负载均衡服务器可以很好地解决上面的问题。

用了反向代理服务器后，网站的架构就进一步演进，变成了用户通过域名访问，DNS 服务器返回反向代理服务器的 IP，反向代理服务器根据被代理服务器的 IP 配置和负载均衡的策略，转发用户的请求到不同的后端服务器，后端服务器返回响应结果到反向代理服务器，反向代理服务器返回结果给用户。

反向代理服务器因为只转发用户的请求而不做具体处理，因此它的性能比应用服务器强大。

随着流量的继续增大，单台反向代理服务器成为了瓶颈，我们就会对其做集群，解决高性能的问题，对其做主备解决高可用的问题。

至此，一个网站的分布式架构的演进过程介绍完了。

#### Nginx 在分布式架构的作用

通过上面的架构演进介绍，我们知道分布式架构中最重要的思路就是水平扩展，水平扩展最重要的一个环节就是反向代理和负载均衡。

Nginx 就是一个高性能的反向代理和负载均衡服务器，它可以支持 5 万的并发访问，同时它可以做动静分离，可以解决跨域访问和防盗链的问题。

### 三、Nginx 的下载与安装

要了解一个软件或者学习一个技术，第一步就是安装，只有安装好了，我们才可以对其进行练习和使用，最终达到掌握的目的。

Nginx 是一个开源的软件，我们可以从官网上下载其最新的版本进行安装和使用。

#### Nginx 的下载

> 官网：http://nginx.org/en/download.html

我们下载 nginx-1.14.1 版本，下载的是 Nginx 的源码，C 语言写的。

![1543484625005.png](E:\other\网络\assets\21-5-4_14-28-54.701_54341.png)

#### Nginx 的安装过程

源码的安装一般有三个步骤：

- 配置（configure）
- 编译（make）
- 安装（make install）

**1.** 在 CentOS 7 的虚拟机根目录下建立 /data/program 的文件夹，用来存放应用程序文件。

**2.** 把下载的 nginx-1.14.1.tar.gz 文件上传到 /data/program 的文件夹下。

**3.** 解压 nginx-1.14.1.tar.gz 文件。

```
tar -zxvf nginx-1.14.1.tar.gz
```

**4.** 在 /data/program 目录下创建 nginx 文件夹把 Nginx 软件安装到该目录下，便于管理。

**5.** 对 Nginx 进行安装前的配置检查。

```
tar -zxvf nginx-1.14.1.tar.gz
./configure --prefix=/data/program/nginx
```

**6.** 在配置检查这一步，可能会遇到一些缺少依赖库的问题，按照提示用 `yum` 命令进行安装即可。

**7.** 依赖安装后，再一次执行命令对 Nginx 进行安装前的配置检查，不通过则继续 `yum` 安装依赖，直到配置检查通过。

- 找不到 C 编译器：`yum -y install gcc`
- 找不到 PCRE library：`yum -y install pcre-devel`
- 找不到 zlib 包：`yum -y install zlib-devel`

**8.** 配置好之后，进行编译和安装。

```
make && make install
```

**9.** 进入 nginx 目录下，出现 conf、html、logs、sbin 目录，则说明安装完成。

```
[root@base-1 nginx]
conf  html  logs  sbin
[root@base-1 nginx]
```

#### Nginx 的启动和停止

Nginx 的启动

到 /data/program/nginx/sbin 目录下执行 ./nginx

```
[root@base-1 sbin]
/data/program/nginx/sbin
[root@base-1 sbin]
nginx
[root@base-1 sbin]
```

查看效果

- 启动后，访问 Nginx，浏览器输入 http://192.168.1.8
- 如果访问不到，说明防火墙的问题
- 临时关闭防火墙 `systemctl stop firewalld`
- 防火墙禁止开机启动 `systemctl disable firewalld`

访问到下图的界面，说明 Nginx 已经生效。

![1543502970798.png](E:\other\网络\assets\21-5-4_14-28-54.855_70363.png)

Nginx 的停止

到 /data/program/nginx/sbin 目录下执行 ./nginx -s stop

```
[root@base-1 sbin]
```

### 四、Nginx 的配置文件介绍

Nginx 的配置文件是 **nginx.conf**，它在 /data/program/nginx/conf 目录下。

```
[root@base-1 conf]
/data/program/nginx/conf
```

#### nginx.conf 介绍

nginx.conf 配置文件里有 3 个部分，分别是：

- 全局块
- events 块
- http 块

在 http 块中又包含 http 全局块、多个 server 块；每个 server 块中又包含 server 全局块和多个 location 块。

1. 全局块是配置文件从开始到 events 块之间的内容，主要有 worker_processes，它表示工作进程数，一般设置为服务器 CPU 的核数。
2. events 块主要有 worker_connections，它表示 Nginx 服务器与用户的网络连接数，Nginx 支持的网络连接数就是 worker_processes 配置数字乘以 worker_connections 配置数字。假如 worker_processes = 4，worker_connections = 1024，那么该 Nginx 服务器支持的连接数就是 4 * 1024 = 4096 个。
3. http 块是 Nginx 服务器配置中的重要组成部分，反向代理，负载均衡，动静分离等多数功能都在这里配置。
4. server 块里进行虚拟主机的配置，有 3 种配置，分别是基于 IP、基于端口号、基于域名的虚拟主机。
5. location 块在整个 Nginx 配置文档中起着非常重要的作用，很多功能的实现需要在 location 块种进行配置。

#### location 使用介绍

##### **配置语法介绍**

**location 的语法结构**

```
location [ = | ~ | ~* | ^~ ] uri { ... }
```

其中，uri 是待匹配的请求字符串，可以是标准 uri（不含正则表达式的 uri），也可以是正则 uri（使用正则表达式的 uri）；方括号里的部分是可选项，用来表示请求字符串与 uri 的匹配方式。

- “=”，用于标准 uri 前，要求请求字符串与 uri 完全匹配，一般称之为精准匹配。
- “^~”，用于标准 uri 前，一般称为前缀匹配。
- “~*”，用于表示 uri 包含正则表达式，不区分大小写。
- “~”，用于表示 uri 包含正则表达式，且区分大小写。

##### **配置规则介绍**

在没有可选项的情况下，指的就是通用匹配。

1. Nginx 服务器首先在 server 块的多个 location 中搜索是否有标准 uri 和请求字符串匹配，如果有多个可以匹配，就记录匹配度最高的一个。
2. 然后，服务器再用 location 块中的正则 uri 和请求字符串匹配，当第一个正则 uri 匹配成功，结束搜索，并使用这个 location 块处理该请求；
3. 如果正则匹配全部失败，就使用刚才记录的匹配度最高的 location 块处理该请求。

使用 “=” 修饰 uri，指的是精准匹配：

- 如果匹配成功，就停止继续向下搜索，立即用匹配到的 location 块处理该请求。

使用“^~”修饰 uri，指前缀匹配：

- 要求 Nginx 服务器找到标识 uri 和请求字符串匹配度最高的 location 后，立即使用此 location 块处理该请求，而不再使用正则 uri 匹配。

使用“~*”或者“~”修饰 uri，指正则匹配：

- 正则匹配会覆盖通用匹配。

##### **规则的优先级**

从上面的配置规则可以看出，精准匹配的优先级最高，前缀匹配的优先级第二，正则匹配第三。

#### nginx.conf 文件简化版配置示例

```
worker_processes  1;events {    worker_connections  1024;
}http {    include       mime.types;    default_type  application/octet-stream;    sendfile        on;    keepalive_timeout  65;    server {        listen       80;        server_name  localhost;        location / {            root   html;            index  index.html index.htm;
        }        
        error_page   500 502 503 504  /50x.html;        location = /50x.html {            root   html;
        }
    }
}
```

至此 Nginx 的基础介绍完毕，下面将进行实例演示。

### 五、实例演示环境介绍（CentOS + Nginx + Tomcat）

- 操作系统是 CentOS  7.5；
- Nginx 服务器版本是 nginx-1.14.1，IP=192.168.1.8；
- Tomcat 服务器版本是 apache-tomcat-8.5.35，两台 Tomcat 服务器 Tomcat1 的 IP=192.168.1.9，Tomcat2 的 IP=192.168.1.10。

**架构图如下：**

![1543566621084.png](E:\other\网络\assets\21-5-4_14-28-54.774_25816.png)

### 六、Nginx 的反向代理演示

Nginx 默认自带 proxy_pass 指令，通过在 nginx.conf 配置文件中设置 proxy_pass 的参数就可以实现反向代理。

proxy_pass 既可以是域名，也可以是 IP 地址，还可以指定端口。

**示例说明**：

用户在浏览器输入 http://192.168.1.8 本来访问到的是 Nginx 的首页，我们希望 Nginx 代理 Tomcat1 服务器，用户访问 http://192.168.1.8 打开 Tomcat1 服务器的首页。

**nginx.conf 配置**

在 http 块的 server 块的 location 块写入如下代码：

```
proxy_pass http://192.168.1.9:8080;
```

如下图：

![1543573195981.png](E:\other\网络\assets\21-5-4_14-28-54.979_93045.png)

启动 Tomcat1 服务器的 Tomcat，然后重启 Nginx，浏览器访问 http://192.168.1.8。

重启 Nginx，在 /data/program/nginx 目录下输入以下命令：

```
../nginx -s reload
```

启动 Tomcat1 服务器的 Tomcat，在 /data/program/tomcat8/bin 目录下输入以下命令：

```
sh startup.sh
```

浏览器输入 http://192.168.1.8，看访问结果如下：

![1543572197064.png](E:\other\网络\assets\21-5-4_14-28-54.556_40382.png)

说明反向代理设置成功。

### 七、Nginx 的负载均衡演示

#### 准备篇

为了进行负载均衡的演示，Tomcat 服务器的 /data/program/tomcat8/webapps/ROOT 目录下，我们把原始 index.jsp 备份，修改其内容，以便区分两台 Tomcat 服务器。

Tomcat1 的 index.jsp 的上部增加以下代码：

```
<h1> 访问到的是Tomcat1  <h1>
```

如下图：

![1543572394074.png](E:\other\网络\assets\21-5-4_14-28-54.742_76093.png)

Tomcat2 不用修改，就用原始的 Tomcat 首页，这样就能区分出用户实际访问到的是哪台 Tomcat 服务器了。

我们访问 Tomcat1 服务器的地址 http://192.168.1.9:8080。

![1543572499873.png](E:\other\网络\assets\21-5-4_14-28-54.542_55229.png)

接下来，我们启动 Tomcat2 服务器的 Tomcat，访问 Tomcat2 服务器的地址 http://192.168.1.10:8080。

![1543572822675.png](E:\other\网络\assets\21-5-4_14-28-54.563_40184.png)

**nginx.conf 原始配置引用外部配置文件**

为了修改方便，我们在 nginx.conf 的 http 块里通过 include 指令引入外部的配置文件，这样可以把不同功能的配置放到不同的配置文件中，避免每次都去修改原始 nginx.conf 配置。

我们在 /data/program/nginx/conf 目录下新建 userconf 文件夹，在 userconf 文件夹下建立 proxy.conf 文件，用于写 Nginx 的配置信息。

![1543570401752.png](E:\other\网络\assets\21-5-4_14-28-54.746_51267.png)

修改 nginx.conf 配置，在 http 块里写入如下代码，表示引用 userconf 文件夹下的所有 conf 文件。

```
include userconf/*.conf;
```

同时删除刚才配置的反向代理代码。

如下图：

![1543570401752.png](E:\other\网络\assets\21-5-4_14-28-54.767_98214.png)

#### 负载均衡篇

负载均衡的原理是用一些策略把网络的访问平衡地分配到集群的各个节点上，使得大量并发访问可以均衡到后端服务器的集群处理，提高网站的性能。

Nginx 的负载均衡是通过 upstream 模块实现的，它提供了一些调度算法来实现客户端 IP 到服务端的负载均衡。

Nginx 默认轮询算法，当服务器宕机后，自动屏蔽，不再往宕机的 IP 分发用户请求。

IP 哈希算法，根据客户端的请求 IP 进行哈希，一个 IP 访问到了后端服务器，以后个用户 IP 每次都会请求到第一次访问的后端服务器，这种方式可以解决 session 共享的问题。

**示例说明**

用户在浏览器输入 http://192.168.1.8 本来访问到的是 Tomcat1 服务器，我们加了一台 Tomcat2 的服务器，想达到用户访问 http://192.168.1.8 后，负载均衡到 2 台 Tomcat 服务器上，既能打开 Tomcat1 服务器的首页，也能打开 Tomcat2 服务器的首页。

**在 proxy.conf 配置文件里加入负载均衡的代码**

```
upstream tomcat8 {
    server 192.168.1.9:8080;
    server 192.168.1.10:8080;
}

server{
    listen 80;
    server_name localhostdomin;
    location / {
        proxy_pass http:
    }
}
```

重启 Nginx 后，访问 http://192.168.1.8。

![1543575041685.png](E:\other\网络\assets\21-5-4_14-28-54.923_37700.png)

![1543575325618.png](E:\other\网络\assets\21-5-4_14-28-55.232_86130.png)

第一次访问到了 Tomcat1，第二次访问到了 Tomcat2，说明负载均衡的功能实现了。

### 八、Nginx 的动静分离演示

- 静态资源：HTML、CSS、JS、图片、XML、MP4 等（不需要依赖 Tomcat 容器），可参考 /data/program/nginx/conf/mime.types 文件里的内容。
- 动态资源：JSP、Servlet（需要 Tomcat 容器）。

动静分离，就是将 HTML、CSS、JS 等静态资源和 JSP 等动态资源分开部署，从而达到提高服务器响应速度，提高服务器性能的目的。

#### 示例说明

依然以 Tomcat1 和 Tomcat2 服务器的首页为例：

1. 首先需要把静态资源从 Tomcat 服务器删掉，分别访问 2 台 Tomcat 服务器，发现网站的样式和图片没有了；
2. 通过代理服务器访问，也是一样的，样式和图片没有了；
3. 最后，我们把 Tomcat 的静态资源放到 Nginx 的根目录的 static_resource 文件夹下，在 location 块里用正则匹配，做好动静分离的配置，重启 Nginx 服务后，通过代理服务器 IP 访问，发现 Tomcat 首页可以正常访问。

#### 准备阶段

**1.** 我们在 Tomcat1 服务器 /data/program/tomcat8/webapps/ROOT 目录下新建 static_resource，把静态资源拷贝到 stataic_resource 文件夹下，这样相当于静态资源删掉了，如下图。

![1543586216327.png](E:\other\网络\assets\21-5-4_14-28-54.915_20568.png)

**2.** 我们在 Tomcat2 服务器 /data/program/tomcat8/webapps/ROOT 目录下新建 static_resource，把静态资源拷贝到 stataic_resource 文件夹下，这样相当于静态资源删掉了，如下图。

![1543586279106.png](E:\other\网络\assets\21-5-4_14-28-55.002_85982.png)

**3.** 访问 Tomcat1 服务器，显示如下：

![1543586388398.png](E:\other\网络\assets\21-5-4_14-28-54.805_86533.png))

**4.** 访问 Tomcat2 服务器，显示如下：

![1543586470526.png](E:\other\网络\assets\21-5-4_14-28-55.487_97411.png)

**5.** 通过 Nginx 代理访问，显示如下：

![1543586510431.png](E:\other\网络\assets\21-5-4_14-28-54.859_97936.png)

说明静态资源访问不到了。

#### Nginx 动静分离配置

**1.** 在 Nginx 的 /data/program/nginx 目录下新建 static_resource 文件夹，里面放 Tomcat 服务器的静态资源，如下图：

![1543587162580.png](E:\other\网络\assets\21-5-4_14-28-54.799_96381.png)

**2.** 修改 Nginx 的 /data/program/nginx/conf/userconf/proxy.conf 文件并保存，用正则来匹配 JS、CSS、图片等静态资源，不区分大小写。

```
   upstream tomcat8 {
       server 192.168.1.9:8080;
       server 192.168.1.10:8080;
   }

   server{
       listen 80;
       server_name localhostdomin;
       location / {
           proxy_pass http:
       }
       location ~* .+\.(js|css|png|svg|ico|jpg)$ {
           root static_resource;
           expires 1d;
       }
   }
```

如下图：

![1543587512785.png](E:\other\网络\assets\21-5-4_14-28-54.760_91581.png)

**3.** 重启 Nginx 服务，通过 Nginx 代理访问，如下：

![1543587339831.png](E:\other\网络\assets\21-5-4_14-28-54.971_75711.png)

![1543587396377.png](E:\other\网络\assets\21-5-4_14-28-54.840_61420.png)

**4.** 两台 Tomcat 服务器的样式和图片能正常显示，说明 Nginx 的动静分离已经配置成功。

#### 动静分离的优势

1. 静态文件从后端服务器分离出来单独部署，可以减轻后端服务器的访问压力，同时 Nginx 是一个高性能的静态 Web 服务器，用它可以做静态资源服务器。
2. 静态文件一般变化不大，动静分离之后，可以对静态文件进行缓冲，以提高网站的性能。

上面的 location 块配置中 `[expires 1d;]` 就是表示缓冲一天的时间。

### 九、Nginx 解决跨域访问问题演示

#### 什么是跨域访问

如果 2 个服务器节点的协议、域名、端口有一个不同，那么这 2 台服务器之间互相访问就会出现跨域访问的问题，跨域限制的根本原因是浏览器的限制，浏览器为了安全从而限制跨域访问。

#### 跨域示例说明

**验证一**

假设 Tomcat2 服务器部署了一个 hello.json 文件，里面是一个 JSON 格式的数据，用它来模拟跨域访问问题。

hello.json 内容如下：

```
{    "hello":"world"}
```

Tomcat1 服务器上的 index.jsp 通过一段 JS 代码要获取到 Tomcat2 服务器上的 hello.json 内容，并通过 alert 输出 key=hello 的值 world。

在 Tomcat1 服务器的 ROOT 下上传 jquery-2.1.1.min.js 文件，在 index.jsp 文件的 head 中增加如下代码：

```
<script src="jquery-2.1.1.min.js"></script><script>
    $(function(){
        $.get("http://192.168.1.10:8080/hello.json",{},function(result){
            alert(result.hello);
        });             
    });</script>
```

如下图：

![1543589625181.png](E:\other\网络\assets\21-5-4_14-28-54.865_7445.png)

通过浏览器输入 http://192.168.1.9:8080，提示“**Failed to load http://192.168.1.10:8080/hello.json: No 'Access-Control-Allow-Origin'** ”，说明出现了跨域访问问题。

![1543589842876.png](E:\other\网络\assets\21-5-4_14-28-54.558_75354.png)

**验证二**

如果我们在 Tomcat1 的 index.jsp 里通过 Nginx 代理获取 Tomcat2 服务器的 hello.json 文件，是否可行呢？

**1.** 修改 /data/program/tomcat8/webapps/ROOT 目录下的 index.jsp 文件，把 IP 地址改为 Nginx 代理的地址：

```
   <script src="jquery-2.1.1.min.js"></script>
   <script>
           $(function(){
                   $.get("http://192.168.1.8/hello.json",{},function(result){
                           alert(result.hello);
                   });             
           });   </script>
```

**2.** 修改 /data/program/nginx/conf/userconf 目录下的 proxy.conf 配置，注释掉对 192.168.1.9:8080 的反向代理，为了演示，让其只代理 192.168.1.10:8080。

```
   upstream tomcat8 {      #server 192.168.1.9:8080;
       server 192.168.1.10:8080;
   }

   server{
       listen 80;
       server_name localhostdomin;
       location / {
           proxy_pass http:
       }
       location ~* .+\.(js|css|png|svg|ico|jpg)$ {
           root static_resource;
           expires 1d;
       }
   }
```

**3.** 重启 Nginx 服务器，通过浏览器输入 http://192.168.1.9:8080，提示“**Failed to load http://192.168.1.8/hello.json: No 'Access-Control-Allow-Origin'**”，说明依然存在跨域访问问题。

**验证三：通过 Nginx 的配置解决以上出现的跨域问题**

再次修改 /data/program/nginx/conf/userconf 目录下的 proxy.conf 配置，通过 add_header 设置解决跨域访问问题。

```
upstream tomcat8 {   #server 192.168.1.9:8080;
    server 192.168.1.10:8080;
}

server{
    listen 80;
    server_name localhostdomin;
    location / {
        proxy_pass http:
        add_header 'Access-Control-Allow-Origin' '*';                
        add_header 'Access-Control-Allow-Methods' 'GET,POST,DELETE';
        add_header 'Access-Control-Allow-Header' 'Content-Type,*';
    }
    location ~* .+\.(js|css|png|svg|ico|jpg)$ {
        root static_resource;
        expires 1d;
    }
}
```

如下图：

![1543591025925.png](E:\other\网络\assets\21-5-4_14-28-54.790_19840.png)

保存配置，并重启 Nginx 服务器。

通过浏览器输入 http://192.168.1.9:8080，可以弹出 world 的值，如下：

![1543591135694.png](E:\other\网络\assets\21-5-4_14-28-54.856_13627.png)

因此，说明通过 Nginx 的配置，我们解决了上面的跨域访问问题，可以正常获取到 Tomcat2 服务器上的 JSON 资源。

**上面的网页没有样式，是因为之前我们把 Tomcat 里面的静态资源挪走导致的。**

### 十、Nginx 的防盗链配置演示

防盗链就是指别人的网站不可以访问我们自己网站的JS、CSS、图片等静态资源。

- 第一，图片有版权，我们不希望别人引用；
- 第二，别人引用了我们的地址，消耗的是我们网站的流量，因为很多网站买的是云服务，是按流量收费的，如果不加防盗链的限制，会提升我们的运营成本，所以一个网站最好要考虑防盗链配置。

**1.** 为了做防盗链的演示，修改 Tomcat1 的 /data/program/tomcat8/webapps/ROOT 目录下的 index.jsp，把 Tomcat 图片改成访问 Nginx 代理服务器里的静态图片资源，代码如下：

```
    <img src="http://192.168.1.8/tomcat.png" alt="[tomcat logo]" />
```

如图：

![1543594229127.png](E:\other\网络\assets\21-5-4_14-28-54.730_31050.png)

**2.** 访问 Tomcat1 的地址 http://192.168.1.9:8080，如果没有防盗链配置，则可以访问到 Nginx 服务器上的静态 Tomcat 图标，如下：

![1543594266821.png](E:\other\网络\assets\21-5-4_14-28-55.122_983.png)

**3.** 修改 /data/program/nginx/conf/userconf 目录下的 proxy.conf 配置，通过 valid_referers 设置解决防盗链问问题，让除了 192.168.1.8 IP 以外的服务器不能访问 Nginx 服务器上的静态资源。

```
   upstream tomcat8 {
       server 192.168.1.9:8080;
       server 192.168.1.10:8080;
   }

   server{       listen 80;
       server_name localhostdomin;
       location / {
           proxy_pass http://tomcat8;
           add_header 'Access-Control-Allow-Origin' '*';
           add_header 'Access-Control-Allow-Methods' 'GET,POST,DELETE';
           add_header 'Access-Control-Allow-Header' 'Content-Type,*';
       }
       location ~* .+\.(js|css|png|svg|ico|jpg)$ {
           valid_referers none blocked 192.168.1.8;           
           if ($invalid_referer){               return 404;
           }
           root static_resource;
           expires 1d;
       }   
   }
```

如下图：

![1543592255323.png](E:\other\网络\assets\21-5-4_14-28-54.811_36258.png)

**4.** 保存配置，重启 Nginx 服务器，再次访问 Tomcat1 的地址 http://192.168.1.9:8080，因为上面设置了防盗链配置，因此不能访问到 Nginx 服务器上的静态 Tomcat 图标，如下：

![1543594529777.png](E:\other\网络\assets\21-5-4_14-28-54.844_21111.png)

以上就是防盗链产生了作用，导致 192.168.1.9 服务器部署的应用不可以访问 192.168.1.8 的图片。

这里 192.168.1.9 模拟的是别人的网站，192.168.1.8 模拟自己的网站。





























