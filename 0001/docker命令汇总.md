docker命令汇总

docker作为轻量级的、高性能的沙箱容器，使用频率极高，功能非常强大。

强大的功能需要繁杂的命令来支撑，虽然docker命令很多，多的记不住。

好记性不如一个烂笔头，本文汇总了docker常用的命令，并对每个命令进行说明和举例，可以随用随取

![图片](https://mmbiz.qpic.cn/mmbiz_png/wibgWibeaanUngOggxNSfEfIeyricu1chtoroSpA0HKnwpep9Do50VCIqdHzGkIxqx8poabFIbc1IlmfdnHv7xibVQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

镜像仓库用来保存镜像，可分为远程镜像仓库和本地镜像仓库。

通过pull命令可以把远程仓库的镜像下载到本地，通过push命令可以把本地仓库的镜像推送到远程

本地仓库中的镜像可以用来创建容器，一个镜像可以创建多个容器

容器也可以通过commit命令打包成镜像，提交到本地仓库。

# 操作远程仓库的命令

## login：登录到远程仓库

login命令可以登录到远程仓库，登录到远程仓库后可可以拉取仓库的镜像了

**login语法**

```
docker login [OPTIONS] [SERVER]
```

> `OPTIONS`：可选参数
> `SERVER`：远程仓库的地址，默认为docker官方仓库，也就是 https://hub.docker.com/

**OPTIONS的常用值**

> `-u` string：用户名
> `-p` string：密码

**login常用写法**
使用helianxiaowu用户登录远程仓库，密码为123456

```
docker login -u helianxiaowu -p 123456 192.168.10.10/docker-lib
```

不指定用户登录到远程仓库，这时会提示输入用户名或密码

```
docker login 192.168.10.10/docker-lib
```

不指定用户登录到默认的远程仓库，也会提示输入用户名或密码

```
docker login
```

## search：从远程仓库搜索镜像

search命令可以从远程仓库搜索镜像
![图片](https://mmbiz.qpic.cn/mmbiz_png/wibgWibeaanUngOggxNSfEfIeyricu1chtodErCibibZ6RMP7Puubpx8QVBxsxlJPfSmiaxDTDQkumNjY6HLGknfxaow/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 列含义如下

| NAME     | DESCRIPTION | STARS                          | OFFICIAL       | AUTOMATED    |
| -------- | ----------- | ------------------------------ | -------------- | ------------ |
| 镜像名称 | 镜像描述    | 镜像热度，类似于github的starts | 是否是官方发布 | 是否自动构建 |

**search语法**

```
docker search [OPTIONS] TERM
```

> `OPTIONS`：可选参数
> `TERM`：镜像的关键词

**OPTIONS的常用值**

> `-f` filter: 根据条件过滤镜像，过滤条件详见下文
> `no-trunc`：显示完整的镜像描述。默认情况下，搜索出来的镜像的描述太长的话会隐藏，no-trunc参数会让镜像信息完整的展示出来
> `--limit` int: 限制搜索出来的镜像的个数，最大不能超过100个，默认25个
> `--format` string: 指定镜像显示的格式，格式详见下文

- `-f`参数表示根据条件过滤搜索出来的镜像，语法如下

> docker search `-f KEY=VALUE` TERM

- KEY的可选值如下

> `stars` int: 根据热度过滤，如：stars=10表示过滤热度大于10的镜像
> `is-automated` boolean: 根据是否自动构建过滤，如：is-automated=false表示过滤非自动构建的镜像
> `is-official` boolean: 根据是否官方发布过滤，如：is-official=false表示过滤非官方发布的镜像

- `--format`参数用来指定搜索出来的镜像的显示的格式，语法如下。`table`表示使用表格的方式显示，支持`\t`格式

> docker search `--format "[table] {{COLUMN}}[{{COLUMN}}...]"` TERM

- COLUMN的可选值如下：

> `.Name`：显示镜像的名称列
> `.Description`：显示镜像的描述列
> `.StarCount`：显示镜像的热度一列
> `.IsOfficial`：显示镜像是否是官方发布一列
> `.IsAutomated`：显示镜像是否是自动构建一列

**search常用写法**
搜索centos镜像

```
docker search centos
```

搜索centos镜像，只展示5个

```
docker search --limit 5 centos
```

搜索热度大于100并且不是自动构建的centos镜像

```
docker search -f stars=100 -f is-automated=true centos
```

搜索非官方发布的centos镜像，搜索结果只展示名称和热度，列之间用TAB键隔开

```
docker search -f is-official=false --format "table{{.Name}}\t{{.StarCount}}" centos
```

## push：把本地镜像推送到远程仓库

push可以把本地仓库中的镜像推送到远程仓库，不过需要先登录远程仓库

**push用法**

```
docker push [OPTIONS] NAME[:TAG]
```

> `OPTIONS`：可选参数
> `NAME`：镜像名称
> `TAG`：镜像版本号，可省略，默认为latest

**OPTIONS的常用值**

> `--disable-content-trust`：推送时远程仓库不校验签名，默认为true

**push常用写法**
将my-image镜像的1.1.0版本推送到远程仓库

```
docker push my-image:1.1.0
```

将my-image镜像推送到远程仓库，不指定版本时默认为latest版本

```
docker push my-image
```

## pull：从远程仓库拉取或更新镜像

pull命令可以从远程仓库拉取镜像，如果本地仓库已经存在该镜像，则会更新

**pull语法**

```
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

> `OPTIONS`：可选参数
> `NAME`：镜像名称
> `TAG`：镜像版本号，可省略，默认为latest
> `DIGEST`：镜像的摘要，每个镜像都有对应的名称、id、摘要信息，每个摘要信息能唯一代表一个镜像，比如下图

![图片](https://mmbiz.qpic.cn/mmbiz_png/wibgWibeaanUngOggxNSfEfIeyricu1chtoQdnViaPv1YhWQaP13bmMLD1LM0scnK10prYNch0o1XaicB9VZbDRQw1Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**OPTIONS的常用值**

> `-a`: 拉取镜像的所有版本号
> `--disable-content-trust`：推送时远程仓库不校验签名，默认为true
> `-q`: 安静模式，推送过程中不展示详细信息

**pull常用写法**
从远程仓库拉取centos镜像，不指定版本时默认为latest版本

```
docker pull centos
```

使用安静模式从远程仓库拉取版本号为5.11的centos镜像

```
docker pull -q centos:5.11
```

使用安静模式从远程仓库拉取所有版本号的centos镜像

```
docker pull -a -q centos
```

# 操作本地镜像的命令

## images：显示所有镜像

images命令可以显示本地存在的所有镜像
![图片](https://mmbiz.qpic.cn/mmbiz_png/wibgWibeaanUngOggxNSfEfIeyricu1chtonPXYxibQFjW1A3FPxPFrduHhiamlQmYfOrEed7vSObMZbqmBibYS36eIA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 列含义如下

| REPOSITORY | TAG      | IMAGE ID | CREATED  | SIZE     |
| ---------- | -------- | -------- | -------- | -------- |
| 仓库路径   | 镜像版本 | 镜像id   | 创建时间 | 镜像大小 |

**images语法**

```
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

> `OPTIONS`：可选参数
> `REPOSITORY`：镜像路径
> `TAG`：镜像版本

**OPTIONS的常用值**

> `-a`: 显示所有镜像，包含中间映像（默认情况下中间映像是隐藏的）
> `-f` filter: 根据条件过滤镜像，过滤条件详见下文
> `-q`: 只显示镜像id
> `no-trunc`：显示完整的镜像id。默认情况下，镜像的id只显示前12位，no-trunc参数会将镜像id完整的显示出来
> `--digests`：显示镜像的摘要信息
> `--format` string: 指定镜像显示的格式，格式详见下文

- `-f`参数表示根据条件过滤要显示的镜像，语法如下

> docker images `-f KEY=VALUE` [REPOSITORY[:TAG]]

- KEY的可选值如下

> `dangling` boolean：过滤悬挂的镜像，如：dangling=true表示只显示悬挂的镜像
> `label` string: 根据标签过滤，如：label=version表示显示有version标签的镜像，label=version=1.0表示显示version=1.0的镜像
> `before` image: 显示在某个镜像之前创建的镜像，如：before=centos:5.8表示显示在centos:5.8这个镜像之前创建的镜像
> `since` image: 显示在某个存在之后创建的镜像，如：since=centos:5.8表示显示在centos:5.8这个镜像存在之后的镜像
> `reference` string：模糊匹配，如：reference=cent*:5*, 显示名称已cent开头版本号已5开头的镜像

- `--format`参数用来指定镜像显示格式，语法如下。`table`表示使用表格的方式显示，支持`\t`格式

> docker images `--format "[table] {{COLUMN}}[{{COLUMN}}...]"` [REPOSITORY[:TAG]]

- COLUMN的可选值如下：

> `.ID`：显示镜像的名称列
> `.Repository`：显示镜像的描述列
> `.Tag`：显示镜像的热度一列
> `.Digest`：显示镜像是否是官方发布一列
> `.CreatedSince`：显示镜像是否是自动构建一列
> `.CreatedAt`：显示镜像是否是自动构建一列
> `.Size`：显示镜像是否是自动构建一列

**images常用写法**
显示本地所有镜像

```
docker images 
```

显示本地所有镜像，只显示id列并且不截断

```
docker images -q --no-trunc
```

显示centos镜像信息

```
docker images centos
```

显示列中包含cent关键字的所有镜像

```
docker images | grep cent
```

显示本地所有镜像，并显示摘要列

```
docker images --digests
```

显示在cengos:latest镜像之后创建的latest版本的所有镜像

```
docker images -f since=centos:latest -f reference=*:latest
```

显示所有镜像信息，只显示镜像id、摘要、创建时间3列，列之间用TAB键隔开

```
docker images --format "table {{.ID}}\t{{.Digest}}\t{{.CreatedAt}}"
```

显示在centos:5.11镜像之前创建的镜像，只显示镜像仓库路径、版本号、创建时间3列，列之间用TAB键隔开

```
docker images -f before=centos:5.11 --format "table {{.Repository}}\t{{.Tag}}\t{{.CreatedAt}}"
```

## rmi：删除本地镜像

rmi命令可以删除一个或多个本地镜像，通常情况应该用rm表示删除命令，但是在doker命令中rm表示删除容器，所以用rmi表示删除镜像，其中的`i`是image的首字母

**rmi语法**

```
docker rmi [OPTIONS] IMAGE [IMAGE...]
```

> `OPTIONS`：可选参数
> `IMAGE`：镜像id或仓库路径名称

**OPTIONS的常用值**

> `-f`: 强制删除，如果镜像有对应的容器正在运行，则不允许直接删除镜像，需要强制删除
> `--no-prune`：不删除该镜像的过程镜像，默认是删除的

**rmi常用写法**
删除centos镜像

```
docker rmi tomcat
```

删除centos:5.11镜像

```
docker rmi centos:5.11
```

删除id为621ceef7494a的镜像

```
docker rmi 621ceef7494a
```

同时删除tomcat、centos和redis镜像

```
docker rmi tomcat centos redis
```

强制删除tomcat镜像，就算此时有tomcat容器正在运行，镜像也会被删除

```
docker rmi -f tomcat
```

## tag：标记镜像，将其归入仓库

tag命令可以基于一个镜像，创建一个新版本的镜像并归入本地仓库，此时该镜像在仓库中存在两个版本，可以根据这两个镜像创建不同的容器

**tag语法**

```
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

> `SOURCE_IMAGE`: 原镜像
> `TARGET_IMAGE`：新镜像
> `TAG`：镜像的版本号

**tag的常用写法**
基于redis:latest镜像创建my-redis1.0镜像，并把新镜像归入redis-lib仓库

```
docker tag redis:latest redis-lib/my-redis:1.0
```

基于621ceef7494a镜像创建my-redis:test-100m镜像，并把新镜像归入redis-lib仓库

```
docker tag 621ceef7494a redis-lib/my-redis:test-100m
```

## history：查看镜像的创建历史

history命令用来查看某一个镜像的创建历史，也就是镜像的提交记录

**history语法**

```
docker history [OPTIONS] IMAGE
```

> `OPTIONS`：可选参数
> `IMAGE`：镜像

**OPTIONS的常用值**

> `-H` boolean: 已可读的格式打印日期和大小，默认为true
> `-q`: 只显示镜像id
> `no-trunc`：输出结果不截取，正常情况下查看到的结果如果某一列太长会被截取
> `--format` string: 指定镜像显示的格式，格式详见下文

- `--format`用来指定镜像的显示的格式，语法如下。`table`表示使用表格的方式显示，支持`\t`格式

> docker history `--format "[table] {{COLUMN}}[{{COLUMN}}...]"` IMAGE

- COLUMN的可选值如下：

> `.ID`：镜像的ID
> `.CreatedSince`：镜像创建的时长
> `.CreatedAt`：镜像创建的时间戳
> `.CreatedBy`：镜像创建使用的命令
> `.Size`：镜像的大小
> `.Comment`：镜像的评论

**history常用写法**
显示centos镜像的创建历史

```
docker history centos
```

显示centos镜像的创建历史，时间和大小转换为人类可读的格式

```
docker history -H=true centos
```

显示centos镜像的创建历史，只显示ID、创建时间戳和创建时的命令3列，列之间使用TAB键隔开

```
docker history --format "table {{.ID}}\t{{.CreatedAt}}\t{{.CreatedBy}}" centos
```

## save：将镜像打包成文件

save命令可以把一个镜像或多个镜像打包到一个文件中，需要特别注意和export命令的区分

save命令打包的是镜像，包含镜像的所有信息

exprot命令打包的是容器，只是保存容器当时的快照，历史记录和元数据信息将会丢失，详见exprot命令介绍

**save语法**

```
docker save [OPTIONS] IMAGE [IMAGE...]
```

> `OPTIONS`：可选参数
> `IMAGE`：镜像

**OPTIONS的常用值**

> `-o` string: 指定目标文件，和linux原生命令`>`有相同作用

**save常用写法**
将centos镜像打包成my-images.tar

```
docker save centos > /home/my-images.tar
```

将centos镜像和redis镜像打包到my-images.tar

```
docker save centos redis > /home/my-images.tar
```

将centos镜像和redis镜像打包到my-images.tar

```
docker save -o /home/my-images.tar centos redis  
```

## load：从指定文件中加载镜像

load命令可以从指定文件中加载镜像，该文件需要是save命令保存的文件

**load语法**

```
docker load [OPTIONS]
```

> `OPTIONS`：可选参数

**OPTIONS的常用值**

> `-i` string: 指定文件的路径
> `-q`：安静模式输出

**load常用写法**
从my-images.tar文件中加载镜像

```
docker load < /home/my-images.tar
```

从my-images.tar文件中加载镜像

```
docker load -i /home/my-images.tar
```

使用安静模式从my-images.tar文件中加载镜像

```
docker load -i /home/my-images.tar -q
```

# 操作容器的命令

## run：创建一个容器并运行

run命令可以创建一个容器并运行，如果创建容器的镜像不存在则会从远程镜像仓库下载

运行容器的同时还能给容器发送一个命令

**run语法**

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

> `OPTIONS`：可选参数
> `IMAGE`：镜像
> `COMMAND`：需要运行的命令
> `ARG`：命令的参数

**OPTIONS的常用值**
由于run命令的OPTIONS的可选值比较多，这里只列出使用频率最高的一些可选值。使用`docker run --help`可以查看run命令的所有可用参数

> `-i`: 以交互模式运行，通常与-t一起使用
> `-t`: 为容器分配一个伪终端，通常与-i一起使用
> `-d`: 后台模式运行容器，并返回容器id
> `-p` list: 指定端口映射，格式为`宿主机端口:容器端口`
> `-P`: 随机分配端口映射
> `--name` string: 给容器指定一个名称
> `-m` bytes: 限制容器可以使用的内存大小，单位可选b、k、m、g
> `-v` list: 把宿主机的磁盘路径挂载到容器的某个路径
> `--volumes-from` list: 绑定别的容器某个路径到此容器的某个路径
> `-w`: 指定容器的工作目录，默认是根目录
> `--rm`: 当容器停止运行时自动删除
> `--hostname` string: 指定容器的主机名

**run常用写法**
创建一个centos容器，并运行

```
docker run centos
```

创建一个centos容器，并以交互模式运行

```
docker run -it centos
```

创建一个centos容器，并后台模式运行

```
docker run -d centos
```

创建一个centos容器，重命名为my-centos，并以交互模式运行，并在容器中运行bash命令

```
docker run -it --name my-centos centos /bin/bash
```

创建一个spring-boot容器并以交互模式运行，容器重命名为my-boot，并把主机的80端口映射到容器的8080端口，此时访问主机ip+80端口即可访问容器中的sping-boot项目

```
docker run -it --name my-boot -p 80:8080 spring-boot
```

创建一个spring-boot容器并以交互模式运行，容器重命名为my-boot，并把主机/logs/my-boot/的目录绑定到容器的/logs目录，此时my-boot项目的日志可以在主机的/logs/my-boot目录中查看

```
docker run -it --name my-boot -v /logs/my-boot/:/logs/ spring-boot
```

创建一个spring-boot容器并以交互模式运行，容器重命名为my-boot；把主机的80端口映射到容器的8080端口；把主机/logs/my-boot/的路径绑定到容器的/logs目录；给容器分配最大500M的内存；指定spring-boot的配置文件为test

```
docker run -it --name my-boot -p 80:8080 -v /logs/my-boot/:/logs/ -m 500M spring-boot --spring.profiles.active=test
```

## start：启动容器

start命令可以启动一个或多个已经停止的容器

**start语法**

```
docker start [OPTIONS] CONTAINER [CONTAINER...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-a`: 将容器的标准输出或标准错误附加到终端
> `-i`: 为容器附加一个标准输入终端

**start常用写法**
启动已经停止的tomcat容器

```
docker start tomcat
```

启动已经停止的tomcat和centos容器

```
docker start tomcat centos
```

启动已经停止的my-spring-boot容器，并输出日志

```
docker start -a my-spring-boot
```

启动已经停止centos容器，并附加一个输入终端

```
docker start -i centos
```

## restart：重启容器

restart可以对一个或多个容器进行重启。如果容器是未启动的则会启动，如果是正在运行中的，则会重启

**restart语法**

```
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-t` int: 在重启之前等待几秒，默认10秒

**restart常用写法**
重启centos容器

```
docker restart centos
```

20秒之后重启centos和tomcat容器

```
docker restart -t 20 centos tomcat
```

## stop：停止容器

stop命令可以停止一个或多个正在运行的容器

kill命令也可以用来停止容器

不同的是stop命令允许容器在停止之前有一定的时间来进行额外操作，如释放链接、关闭请求等

kill命令则会直接强制杀死容器

**stop语法**

```
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-t` int: 等待n秒后如果还没停止，直接杀死，默认10秒

**stop常用写法**
停止tomcat容器

```
docker stop tomcat
```

停止tomcat和centos容器

```
docker stop tomcat centos
```

停止tomcat容器，如果5秒内还未停止则直接杀死

```
docker stop -t 5 tomcat
```

## restart：重启容器

restart命令可以重启一个或多个容器，不管容器是运行或停止

**restart语法**

```
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-t` int: 如果重启的容器正在运行，等待n秒还没停止，直接杀死然后重启，默认10秒

**restart常用写法**
重启tomcat容器

```
docker restart tomcat
```

重启tomcat和centos容器

```
docker restart tomcat centos
```

重启正在运行的tomcat容器，如果5秒内还未停止则直接杀死然后重启

```
docker restart -t 5 tomcat
```

## kill：杀死容器

kill命令可以杀死一个或多个正在运行的容器

**kill语法**

```
docker kill [OPTIONS] CONTAINER [CONTAINER...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-s` string: 给容器发送一个信号，信号编号和linux原生命令kill的信号编号一致，默认值9，下文列出一些常用值

- `-s`参数信号编号常用值

> `1`：杀死并重新加载，也可用`HUP`表示
> `9`：强制杀死，也可用`KILL`表示，默认值
> `15`：正常停止，也可用`TERM`表示

**kill常用写法**
杀死tomcat容器

```
docker kill tomcat
```

强制杀死tomcat容器

```
docker kill -s 9 tomcat
```

强制杀死tomcat容器

```
docker kill -s KILL tomcat
```

杀死tomcat和centos容器

```
docker kill tomcat centos
```

## rm：删除容器

rm命令可以删除一个或多个容器

如果容器正在运行，则需要通过-f参数强制删除

**rm语法**

```
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-f`：强制删除，即使容器正在运行也可以删除
> `-l`：删除容器之间的网络关系，而不是容器本身
> `-v`: 删除容器和它挂载的卷

**rm常用写法**
删除centos容器

```
docker rm centos
```

强制删除centos容器，即使容器正在运行也会被删除

```
docker rm -f centos
```

删除centos容器，并删除它挂载的卷

```
docker rm -f centos
```

删除所有已经停止的容器

```
docker rm $(docker ps -a -q)
```

移除容器my-nginx对容器my-db的连接，连接名db

```
docker rm -l db 
```

## pause：暂停容器

pause命令可以暂停一个或多个正在运行的容器

**pause语法**

```
docker pause CONTAINER [CONTAINER...]
```

> `CONTAINER`：容器

**pause常用写法**
暂停正在运行的centos容器

```
docker pause centos
```

暂停正在运行的centos和tomcat容器

```
docker pause centos tomcat
```

## unpause：取消暂停容器

unpause命令可以对一个或多个暂停的容器取消暂停

**pause语法**

```
docker unpause CONTAINER [CONTAINER...]
```

> `CONTAINER`：容器

**unpause常用写法**
取消暂停的centos容器

```
docker unpause centos
```

取消暂停centos和tomcat容器

```
docker unpause centos tomcat
```

## create：创建一个容器

create命令可以创建一个容器，但不运行它，在需要的时候可以使用start命令启动

和run命令的用法几乎一致，都会创建一个容器，如果容器依赖的镜像不存在都会从远程仓库拉取

run命令创建容器后会运行容器

create命令只是创建容器，不运行
**create语法**

```
docker create [OPTIONS] IMAGE [COMMAND] [ARG...]
```

> `OPTIONS`：可选参数
> `IMAGE`：镜像
> `COMMAND`：需要运行的命令
> `ARG`：命令的参数

**OPTIONS的常用值**
create命令和run命令的可选参数一样

由于可选参数比较多，这里只列出使用频率最高的一些可选值。使用`docker create --help`可以查看create命令的所有可用参数

> `-i`: 以交互模式运行，通常与-t一起使用
> `-t`: 为容器分配一个伪终端，通常与-i一起使用
> `-d`: 后台模式运行容器，并返回容器id
> `-p` list: 指定端口映射，格式为`宿主机端口:容器端口`
> `-P`: 随机分配端口映射
> `--name` string: 给容器指定一个名称
> `-m` bytes: 限制容器可以使用的内存大小
> `-v` list: 把宿主机的磁盘路径挂载到容器的某个路径
> `--volumes-from` list: 绑定别的容器某个路径到此容器的某个路径
> `-w`: 指定容器的工作目录，默认是根目录
> `--rm`: 当容器停止运行时自动删除
> `--hostname` string: 指定容器的主机名

**create常用写法**
创建一个centos容器

```
docker create centos
```

创建一个centos容器，start启动时以交互模式运行

```
docker create -it centos
```

创建一个centos容器，start启动时后台模式运行

```
docker create -d centos
```

创建一个centos容器，重命名为my-centos，start时以交互模式运行，并在容器中运行bash命令

```
docker create -it --name my-centos centos /bin/bash
```

创建一个spring-boot容器，重命名为my-boot，并把主机的80端口映射到容器的8080端口，start时以交互模式运行，此时访问主机ip+80端口即可访问容器中的sping-boot项目

```
docker create -it --name my-boot -p 80:8080 spring-boot
```

创建一个spring-boot容器，容器重命名为my-boot，并把主机/logs/my-boot/的目录绑定到容器的/logs目录，start时以交互模式运行，此时my-boot项目的日志可以在主机的/logs/my-boot目录中查看

```
docker create -it --name my-boot -v /logs/my-boot/:/logs/ spring-boot
```

创建一个spring-boot容器，容器重命名为my-boot；把主机的80端口映射到容器的8080端口；把主机/logs/my-boot/的路径绑定到容器的/logs目录；给容器分配最大500M的内存；指定spring-boot的配置文件为test；start时以交互模式运行

```
docker create -it --name my-boot -p 80:8080 -v /logs/my-boot/:/logs/ -m 500M spring-boot --spring.profiles.active=test
```

## exec：在容器中执行命令

exce命令可以在一个运行中的容器中执行一个命令

**exec语法**

```
 docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器
> `COMMAND`：要执行的命令
> `ARG`：命令的参数

**OPTIONS的常用值**

> `-d`: 命令在后台运行
> `-i`：保持标准输入，通常和`-t`一起使用
> `-t`: 分配一个伪终端，通常和`-i`一起使用
> `-w` string: 指定容器的路径

**exec常用写法**
在centos容器中运行pwd命令

```
docker exec centos pwd
```

为centos容器分配一个输入终端

```
docker exec -it centos /bin/bash
```

在centos镜像中的bin目录执行ls命令

```
docker exec -w /bin centos ls
```

## ps：查看容器列表

ps命令可以列出所有容器的列表，查看容器的基本信息。不加任何参数的情况下，默认只展示正在运行的容器
![图片](https://mmbiz.qpic.cn/mmbiz_png/wibgWibeaanUngOggxNSfEfIeyricu1chtoRk5iaDA7GibAkMuqpyGn33GiaZXicRURQ75u7NyBulgcLoAPvdHGcWptHw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 列含义如下

| CONTAINER ID | IMAGE      | COMMAND              | CREATED  | PORTS        | NAMES    |
| ------------ | ---------- | -------------------- | -------- | ------------ | -------- |
| 容器id       | 对应的镜像 | 容器启动时运行的命令 | 创建时间 | 绑定的的端口 | 容器名称 |

**ps语法**

```
docker ps [OPTIONS]
```

> `OPTIONS`：可选参数

**OPTIONS的常用值**

> `-a`: 显示所有容器，默认只显示正在运行的
> `-f` filter: 根据条件过滤容器，过滤条件详见下文
> `-n` int：显示最后创建的n个容器，包含所有状态
> `-l`: 显示最新创建的容器，包含所有状态
> `-q`: 只显示容器id
> `-s`: 显示容器大小，默认不显示该列
> `--no-trunc`：显示内容不截断，默认情况下显示的容器是截断后的信息

- `-f`参数表示根据条件过滤搜索出来的镜像，语法如下

> docker ps `-f KEY=VALUE`

- KEY的可选值如下

> `id`: 根据容器id过滤
> `name`: 查看容器名称中包含给定字段的容器
> `exited`: 根据容器退出的错误码进行过滤
> `status`: 根据容器的状态进行过滤，状态可选值有：created、paused、exited、dead、running、restarting、removing
> `before`: 只显示在某个容器之前创建的容器
> `since`: 只显示在某个容器之后创建的容器
> `volume`: 过滤绑定了某个目录的容器，只针对运行中的容器
> `publish`: 根据宿主机端口过滤，只针对运行中的容器
> `expose`: 根据容器端口过滤，只针对运行中的容器

**ps常用写法**
查看运行中的容器

```
docker ps
```

查看所有容器

```
docker ps -a
```

查看所有容器，并显示容器大小

```
docker ps -a -s
```

查看所有容器，显示内容不截断

```
docker ps -a --no-trunc
```

查看容器名称中包含cent的容器

```
docker ps -f name=cent
```

查看状态是created的容器

```
docker ps -f status=created
```

查看在centos之前创建的容器

```
docker ps -f before=centos
```

查看绑定了宿主机80端口并且正在运行的容器

```
docker ps -f publish=80
```

## inspect：获取容器或镜像的元数据

inspect命令可以获取一个或多个容器或者镜像的元数据信息

元数据信息可以理解为容器或者镜像的详情，它比`ps`命令显示的内容要详细的多。比如说端口映射、挂载目录等，显示格式为json类型

**inspect语法**

```
docker inspect [OPTIONS] CONTAINER|IMAGE [CONTAINER|IMAGE...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器
> `IMAGE`：镜像

**OPTIONS的常用值**

> `-f` string: 格式化输出结果，inspect默认显示整个文件的详情，-f参数可以指定只显示某些属性
> `--s`: 只对容器有效，显示容器的配置文件行数和大小，显示的结果中会多出SizeRw、SizeRootFs两个参数
> `--type` string: 指定要inspect的类型，container表示容器，image表示镜像，默认是容器。比如我有一个tomcat镜像，同时有一个名称为tomcat的容器，就可以用--type参数来指定要inspect是tomcat容器还是tomcat镜像

**inspect常用写法**
查看tomcat容器的元数据信息

```
docker inspect tomcat
```

查看tomcat镜像的元数据信息

```
docker inspect --type=image tomcat
```

查看tomcat容器的ip地址

```
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' tomcat
```

查看tomcat容器的ip地址

```
docker inspect tomcat | grep IPAddress
```

查看tomcat容器的端口映射

```
docker inspect -f "{{.HostConfig.PortBindings}}" tomcat
```

查看tomcat容器的挂载目录

```
docker inspect -f "{{.HostConfig.Binds}}" tomcat
```

## stats：监控容器的资源使用情况

stats命令可以可以监控容器的资源使用情况，如cpu使用情况、内存使用情况等。每秒刷新一次，直到使用`ctrl+c`退出
![图片](https://mmbiz.qpic.cn/mmbiz_png/wibgWibeaanUngOggxNSfEfIeyricu1chtoN0sOAaz9hicaV38JszDBuRdKudXbic7ZTafgWV8D071Julw3aSiaEykqA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 列含义如下

| CONTAINER ID | NAME     | CPU %         | MEM USAGE/LIMIT           | MEM %          | NET I/O | BLOCK I/O | PIDS                   |
| ------------ | -------- | ------------- | ------------------------- | -------------- | ------- | --------- | ---------------------- |
| 容器id       | 容器名称 | cpu使用百分比 | 使用内容大小/最大可用内存 | 内存使用百分比 | 网络IO  | 磁盘IO    | 容器内线程或进程的数量 |

**stats语法**

```
docker stats [OPTIONS] [CONTAINER...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-a` filter: 显示所有容器的资源使用情况，默认只显示正在运行的
> `--format` string：格式化输出结果
> `--no-stream`: 不间隔刷新，只显示第一次统计结果
> `--no-trunc`: 不截断显示信息，默认情况下有些字段只显示简略信息，如容器id

**stats常用写法**
监控所有正在运行的容器的资源使用情况

```
docker stats
```

监控所有容器的资源使用情况，包含未启动的容器

```
docker stats -a
```

只监控centos容器的资源使用情况

```
docker stats centos
```

监控centos容器的资源使用情况，显示结果不刷新

```
docker stats --no-stream centos
```

## top：查看容器中运行的进程信息

top可以查看容器的进程信息，`docker exec CONTAINER ps`也可以查看容器的进程。

不同的是，前者查看的是容器运行在宿主机的进程id。后者查看的是容器内的进程id

**top语法**

```
docker top CONTAINER [ps OPTIONS]
```

> `CONTAINER`：容器
> `OPTIONS`：ps命令的可选参数

**top常用写法**
查看centos镜像的宿主机进程id

```
docker top centos
```

## rename：重命名容器

rename可以对容器进行重命名，在容器run时如果没有使用--name参数指定容器名称，可以使用rename进行命名

**rename语法**

```
docker rename CONTAINER NEW_NAME
```

**rename常用写法**
将centos容器重命名为my-centos

```
docker rename centos my-centos
```

## attach：连接到容器内

attach可以连接到容器内，这个容器必须是正在运行的容器，不是运行状态时，会报错

当使用`ctrl+c`或`exit`等命令退出容器时，会导致容器停止运行。所以，不建议在生产环境使用该命令。生产环境可以使用exec命令进入容器

**attach语法**

```
docker attach [OPTIONS] CONTAINER
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `--sig-proxy=false` boolean: 默认true，为false时可以防止容器遇到`ctrl+c`退出信号时停止运行

**attach常用写法**
进入正在运行的centos镜像内

```
docker attach centos
```

## update：更新一个或多个容器的配置

update可以对容器的配置进行更新

**update语法**

```
docker update [OPTIONS] CONTAINER [CONTAINER...]
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-m` bytes: 指定容器的内存大小，单位可选b、k、m、g
> `--memory-swap` bytes：`--cpu` demecial：cpu资源，如1.5表示可以使用宿主机的1.5个cpu资源
> `--cpuset-cpus` string：容器可以使用宿主机的cpu内核编号，`0-3`表示4个内核都可以使用，`1,3`表示只能使用1和3号内核
> `--restart` string: 指定容器的退出的重启策略。no：不重启；on-failure：容器非正常退出时重启；on-failure:3：非正常退出时重启3次；alaways：总是重启；unless-stopped：在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器
> `--pids-limit` int: 限制容器进程或线程的数量，默认-1表示不限制

**update常用写法**
更新centos镜像的内存为2G

```
docker update --memory-swap -1 -m 2g centos
```

更新容器的重启策略

```
docker update --restart on-failure:3 centos
```

更新tomcat容器的最大线程数为2000

```
docker update --pids-limit 2000 tomcat
```

## logs：查看容器的日志

**logs语法**

```
docker logs [OPTIONS] CONTAINER
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-f`: 跟踪日志的实时输出
> `--until` string：查看某个时间点之前的日志，格式：2021-06-03T19:30:20Z。或使用相对时间10m，表示10分钟之前
> `--since` string：查看某个时间点之后的日志，格式：2021-06-03T19:30:20Z。使用相对时间10m，表示10分钟之内
> `-n` int: 查看最后几行日志，默认显示全部
> `-t`: 日志中显示时间戳

**logs常用写法**
查看tomcat最后10行日志

```
docker logs -n 10 tomcat
```

查看tomcat最后10行日志，并实时监控日志输出

```
docker logs -n 10 -f  tomcat
```

查看最近10分钟的日志

```
docker logs --since 10m tomcat
```

查看6月3号9点到10点之间的日志

```
docker logs --since 2021-06-03T9:00:00  --until 2021-06-03T10:00:00 tomcat
```

## wait：阻塞容器，直到容器退出并打印它的退出代码

wait命令可以阻塞一个或多个容器直到容器退出并打印出他们的退出代码

**wait语法**

```
docker wait CONTAINER [CONTAINER...]
```

> `CONTAINER`：容器

**wait常用写法**
阻塞centos容器，直到它退出并打印退出状态码

```
docker wait centos
```

此时新打开一个终端，将centos容器stop掉，切换到wait的终端就可以看到打出一个状态码

## port：列出端口的映射关系

**port语法**

```
docker port CONTAINER [PRIVATE_PORT[/PROTO]]
```

> `CONTAINER`：容器
> `PRIVATE_PORT`：容器端口
> `PROTO`：端口使用的协议

**port常用写法**
查看my-boot容器的端口映射

```
docker port my-boot
```

查看my-boot容器的8080端口映射的宿主机端口

```
docker port my-boot 8080
```

查看my-boot容器使用tcp协议的8080端口映射的宿主机端口

```
docker port my-boot 8080/tcp
```

## export：将容器打包成一个文件

export命令可以将容器打包到一个文件中，它和save命令比较容易混淆

export和save的不同之处在于：export打包的是容器，save打包的是镜像

export打包的是容器当时的快照，至于容器的历史记录和元数据信息都会丢失。还有，export的文件在被import成一个镜像时，可以重新指定镜像的名称和版本号

**export语法**

```
docker export [OPTIONS] CONTAINER
```

> `OPTIONS`：可选参数
> `CONTAINER`：容器

**OPTIONS的常用值**

> `-o` string: 指定打包文件

**export常用写法**
将my-boot容器打包到my-boot.tar文件

```
docker export -o /tmp/my-boot.tar my-boot
```

## import：从本地文件或远程文件导入镜像到本地仓库

import可以从本地文件或远程文件中导入镜像到本地仓库

如果是从文件中导入，这个文件需要是export命令导出的文件

**import语法**

```
docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]
```

> `OPTIONS`：可选参数
> `file`：文件地址
> `URL`：URL地址
> `-`：从标准输入终端导入，通常和linux中的cat命令一起使用
> `REPOSITORY`：本地镜像仓库地址
> `TAG`：镜像版本号

**OPTIONS的常用值**

> `-m` string: 添加描述信息
> `-c` list: 对创建的容器使用dokerfile指令

**import常用写法**
从my-boot.tar文件创建镜像

```
cat /tmp/my-boot.tar | docker import -
```

从my-boot.tar文件创建镜像

```
docker import /tmp/my-boot.tar
```

从my-boot.tar文件创建镜像，并指定镜像名称为my-boot-test、版本号为1.0

```
docker import /tmp/my-boot.tar my-boot-test:1.0
```

从my-boot.tar文件创建镜像，备注信息为测试，并指定镜像名称为my-boot-test、版本号为1.0

```
docker import --message '测试' /tmp/my-boot.tar my-boot-test:1.0
```

从远程服务器的my-boot.tar文件创建镜像

```
docker import http://192.168.100.1:8080/images/my-boot.tar
```

 



