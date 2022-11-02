Nginx 面试中最常见的 18 道题

Nginx的并发能力在同类型网页服务器中的表现，相对而言是比较好的，因此受到了很多企业的青睐，我国使用Nginx网站的知名用户包括腾讯、淘宝、百度、京东、新浪、网易等等。Nginx是网页服务器运维人员必备技能之一，下面为大家整理了一些比较常见的Nginx相关面试题，仅供参考：

## 1、请解释一下什么是Nginx?

Nginx---Ngine X，是一款免费的、自由的、开源的、高性能HTTP服务器和反向代理服务器；也是一个IMAP、POP3、SMTP代理服务器；Nginx以其高性能、稳定性、丰富的功能、简单的配置和低资源消耗而闻名。

也就是说Nginx本身就可以托管网站（类似于Tomcat一样），进行Http服务处理，也可以作为反向代理服务器 、负载均衡器和HTTP缓存。

Nginx 解决了服务器的C10K（就是在一秒之内连接客户端的数目为10k即1万）问题。它的设计不像传统的服务器那样使用线程处理请求，而是一个更加高级的机制—事件驱动机制，是一种异步事件驱动结构。

## 2、请列举Nginx的一些特性

- 跨平台：可以在大多数Unix like 系统编译运行。而且也有Windows的移植版本。 
- 配置异常简单：非常的简单，易上手。 
- 非阻塞、高并发连接：数据复制时，磁盘I/O的第一阶段是非阻塞的。官方测试能支持5万并发连接，实际生产中能跑2~3万并发连接数（得益于Nginx采用了最新的epoll事件处理模型（消息队列）。 
- Nginx代理和后端Web服务器间无需长连接； 
- Nginx接收用户请求是异步的，即先将用户请求全部接收下来，再一次性发送到后端Web服务器，极大减轻后端Web服务器的压力。 
- 发送响应报文时，是边接收来自后端Web服务器的数据，边发送给客户端。 
- 网络依赖性低，理论上只要能够ping通就可以实施负载均衡，而且可以有效区分内网、外网流量。 
- 支持内置服务器检测。Nginx能够根据应用服务器处理页面返回的状态码、超时信息等检测服务器是否出现故障，并及时返回错误的请求重新提交到其它节点上。 
- 此外还有内存消耗小、成本低廉（比F5硬件负载均衡器廉价太多）、节省带宽、稳定性高等特点。

## 3、请列举Nginx和Apache 之间的不同点

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/gUFHIUJJ6iazWAmQmibuPXgDLEzQvJyGVZY72UxZUX6OeEcshwgOlfEibtcowOdNlJ36LNiatKgm18CkVbrUcLdic9w/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 4、请解释Nginx如何处理HTTP请求。

Nginx 是一个高性能的 Web 服务器，能够同时处理大量的并发请求。它结合多进程机制和异步机制 ，异步机制使用的是异步非阻塞方式 ，接下来就给大家介绍一下 Nginx 的多线程机制和异步非阻塞机制 。

**1、多进程机制**

服务器每当收到一个客户端时，就有 服务器主进程 （ master process ）生成一个 子进程（ worker process ）出来和客户端建立连接进行交互，直到连接断开，该子进程就结束了。

使用进程的好处是各个进程之间相互独立，不需要加锁，减少了使用锁对性能造成影响，同时降低编程的复杂度，降低开发成本。其次，采用独立的进程，可以让进程互相之间不会影响 ，如果一个进程发生异常退出时，其它进程正常工作， master 进程则很快启动新的 worker 进程，确保服务不会中断，从而将风险降到最低。

缺点是操作系统生成一个子进程需要进行 内存复制等操作，在资源和时间上会产生一定的开销。当有大量请求时，会导致系统性能下降 。

**2、异步非阻塞机制**

每个工作进程 使用 异步非阻塞方式 ，可以处理 多个客户端请求 。

当某个 工作进程 接收到客户端的请求以后，调用 IO 进行处理，如果不能立即得到结果，就去 处理其他请求 （即为 非阻塞 ）；而 客户端 在此期间也 无需等待响应 ，可以去处理其他事情（即为 异步 ）。

当 IO 返回时，就会通知此 工作进程 ；该进程得到通知，暂时 挂起 当前处理的事务去 响应客户端请求 。

## 5、在Nginx中，如何使用未定义的服务器名称来阻止处理请求?

只需将请求删除的服务器就可以定义为：

![图片](https://mmbiz.qpic.cn/mmbiz_png/gUFHIUJJ6iay23sEXiaRGraGvdj7G5yy6ibTl7UrMM5DQkGlNtTcrFloUSUdaUt5AiaO1icO4sC2W96Y8iaSyQvBHsJQ/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里，服务器名被保留为一个空字符串，它将在没有“主机”头字段的情况下匹配请求，而一个特殊的Nginx的非标准代码444被返回，从而终止连接。

## 6、 使用“反向代理服务器”的优点是什么?

反向代理服务器可以隐藏源服务器的存在和特征。它充当互联网云和web服务器之间的中间层。这对于安全方面来说是很好的，特别是当您使用web托管服务时。

## 7、请列举Nginx服务器的最佳用途。

Nginx服务器的最佳用法是在网络上部署动态HTTP内容，使用SCGI、WSGI应用程序服务器、用于脚本的FastCGI处理程序。它还可以作为负载均衡器。

## 8、请解释Nginx服务器上的Master和Worker进程分别是什么?

- 主程序 Master process 启动后，通过一个 for 循环来 接收 和 处理外部信号 ；
- 主进程通过 fork() 函数产生 worker 子进程 ，每个子进程执行一个 for循环来实现Nginx服务器对事件的接收和处理 。

![图片](https://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdYJzfyu1L8rI8asia8NkuKt5RiajKOy5hzmpA2NxqZBqakbyVj7QtricyBia1MEgtMOkzoiaF7Z1sFc1hg/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

一般推荐 worker 进程数与CPU内核数一致，这样一来不存在大量的子进程生成和管理任务，避免了进程之间竞争CPU 资源和进程切换的开销。而且 Nginx 为了更好的利用 多核特性 ，提供了 CPU 亲缘性的绑定选项，我们可以将某一个进程绑定在某一个核上，这样就不会因为进程的切换带来 Cache 的失效。

对于每个请求，有且只有一个工作进程 对其处理。首先，每个 worker 进程都是从 master进程 fork 过来。在 master 进程里面，先建立好需要 listen 的 socket（listenfd） 之后，然后再 fork 出多个 worker 进程。

所有 worker 进程的 listenfd 会在新连接到来时变得可读 ，为保证只有一个进程处理该连接，所有 worker 进程在注册 listenfd 读事件前抢占 accept_mutex ，抢到互斥锁的那个进程注册 listenfd 读事件 ，在读事件里调用 accept 接受该连接。

当一个 worker 进程在 accept 这个连接之后，就开始读取请求、解析请求、处理请求，产生数据后，再返回给客户端 ，最后才断开连接。这样一个完整的请求就是这样的了。我们可以看到，一个请求，完全由 worker 进程来处理，而且只在一个 worker 进程中处理。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdYJzfyu1L8rI8asia8NkuKt5X9ZBNgaawiaFg1KEAwZaibXuichhPNgHTZRqXLicd2avQzMMxOTc2mBDXA/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在 Nginx 服务器的运行过程中， 主进程和工作进程 需要进程交互。交互依赖于 Socket 实现的管道来实现。

## 9、请解释代理设计中的正向代理和反向代理?

首先，代理服务器一般指局域网内部的机器通过代理服务器发送请求到互联网上的服务器，代理服务器一般作用在客户端。例如：GoAgent翻墙软件。我们的客户端在进行翻墙操作的时候，我们使用的正是正向代理，通过正向代理的方式，在我们的客户端运行一个软件，将我们的HTTP请求转发到其他不同的服务器端，实现请求的分发。

![图片](https://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdYJzfyu1L8rI8asia8NkuKt5vXdia1RicjhSpzVkvmOuHDVRWMcljQh60Z90cmLHFQ4Md8cJq4Kn0thg/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

反向代理服务器作用在服务器端，它在服务器端接收客户端的请求，然后将请求分发给具体的服务器进行处理，然后再将服务器的相应结果反馈给客户端。Nginx就是一个反向代理服务器软件。

![图片](https://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdYJzfyu1L8rI8asia8NkuKt5EGFQBBLqOG2wMNh27CPcndwibeMrRaQsU9q553ZkwsQaB7SNtr8aibJw/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上图可以看出：客户端必须设置正向代理服务器，当然前提是要知道正向代理服务器的IP地址，还有代理程序的端口。 
反向代理正好与正向代理相反，对于客户端而言代理服务器就像是原始服务器，并且客户端不需要进行任何特别的设置。客户端向反向代理的命名空间（name-space）中的内容发送普通请求，接着反向代理将判断向何处（原始服务器）转交请求，并将获得的内容返回给客户端。

![图片](https://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdYJzfyu1L8rI8asia8NkuKt5UOLvTTtAFI04Dbicmz7Cb23rjVwTia9bJicPYkhB7lSKUAO7YOzmXwE8A/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 10、请解释是否有可能将Nginx的错误替换为502错误、503?

502 =错误网关

503 =服务器超载

有可能，但是您可以确保fastcgi_intercept_errors被设置为ON，并使用错误页面指令。

![图片](https://mmbiz.qpic.cn/mmbiz_png/gUFHIUJJ6iay23sEXiaRGraGvdj7G5yy6ib18waolmZ4ia4z150Baw2uBIkRickyuAXIYhQpDmuxNic4RzmccYOaJSdA/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 11、在Nginx中，解释如何在URL中保留双斜线?

要在URL中保留双斜线，就必须使用merge_slashes_off;

语法:merge_slashes [on/off]

默认值: merge_slashes on

环境: http，server

## 12、请解释ngx_http_upstream_module的作用是什么?

ngx_http_upstream_module用于定义可通过fastcgi传递、proxy传递、uwsgi传递、memcached传递和scgi传递指令来引用的服务器组。

## 13、请解释什么是C10K问题?

C10K问题是指无法同时处理大量客户端(10,000)的网络套接字。

## 14、请陈述stub_status和sub_filter指令的作用是什么?

Stub_status指令：该指令用于了解Nginx当前状态的当前状态，如当前的活动连接，接受和处理当前读/写/等待连接的总数；

Sub_filter指令：它用于搜索和替换响应中的内容，并快速修复陈旧的数据；

## 15、解释Nginx是否支持将请求压缩到上游?

您可以使用Nginx模块gunzip将请求压缩到上游。gunzip模块是一个过滤器，它可以对不支持“gzip”编码方法的客户机或服务器使用“内容编码:gzip”来解压缩响应。

## 16、解释如何在Nginx中获得当前的时间?

要获得Nginx的当前时间，必须使用SSI模块、$date_gmt和$date_local的变量。

Proxy_set_header THE-TIME $date_gmt;

## 17、用Nginx服务器解释-s的目的是什么?

用于运行Nginx -s参数的可执行文件。

## 18、解释如何在Nginx服务器上添加模块?

在编译过程中，必须选择Nginx模块，因为Nginx不支持模块的运行时间选择。