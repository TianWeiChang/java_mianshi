Linux 最常用命令：解决 95% 以上的问题

Linux是目前应用最广泛的服务器操作系统，基于Unix，开源免费，由于系统的稳定性和安全性，市场占有率很高，几乎成为程序代码运行的最佳系统环境。linux不仅可以长时间的运行我们编写的程序代码，还可以安装在各种计算机硬件设备中，如手机、路由器等，Android程序最底层就是运行在linux系统上的。

## 一、Linux的目录结构

**![图片](https://mmbiz.qpic.cn/mmbiz_jpg/tuSaKc6SfPrCdhhPgOiaoBcFPce4tpjykKa6Mib0INI8WR69mJnFnqycibJJyudibgmAPfx8eDgL3p9Phe3tgFmVzA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)**

/的下级目录

- bin (binaries)存放二进制可执行文件
- sbin (super user binaries)存放二进制可执行文件，只有root才能访问
- etc (etcetera)存放系统配置文件
- usr (unix shared resources)用于存放共享的系统资源
- home 存放用户文件的根目录
- root 超级用户目录
- dev (devices)用于存放设备文件
- lib (library)存放跟文件系统中的程序运行所需要的共享库及内核模块
- mnt (mount)系统管理员安装临时文件系统的安装点
- boot 存放用于系统引导时使用的各种文件
- tmp (temporary)用于存放各种临时文件
- var (variable)用于存放运行时需要改变数据的文件

## 二、Linux常用命令

**命令格式：**命令 -选项 参数 （选项和参数可以为空）

```
如：ls -la /usr
```

**2.1 操作文件及目录**

**![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCiaxBcpuWAufxawefmBjR3svV0XfIDe91ANDN7POcialBPIRX4UCVdHmew/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)**

**2.2 系统常用命令**

**![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCiaykCib7bstwWapYzXfQtvlZPPVdLpyB61yoIh62Ss300C5eYRTeObMzQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)**

**2.3 压缩解压缩**

**![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCiayWYRKtESM2QO2PibplMMtwbtlWDV5GD7y0BlP03xyUJmrkkZIBmbwkA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)**

**2.4 文件权限操作**

- linux文件权限的描述格式解读

![图片](https://mmbiz.qpic.cn/mmbiz_png/FeiahiakEJkVvhiaEib2ZPwqKVguRTccM9v33cUkTWeKcOxxMtyHA25FdkSzYOeGdDdib1RJMoN9KVgr0X86nS1Jp3w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- r 可读权限，w可写权限，x可执行权限（也可以用二进制表示 111 110 100 --> 764）
- 第1位：文件类型（d 目录，- 普通文件，l 链接文件）
- 第2-4位：所属用户权限，用u（user）表示
- 第5-7位：所属组权限，用g（group）表示
- 第8-10位：其他用户权限，用o（other）表示
- 第2-10位：表示所有的权限，用a（all）表示

![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCiaJYiauCAFjempsSiaUsNEQKU5muRNamqCt6XKxX51uUcUJQIfauCBGb7g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 三、Linux系统常用快捷键及符号命令

![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCiaF8T3HwDibZb6EoGzCuTP6KGkCtByUcHRfpWL288lJPFkNQjicYWdnKoQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 四、vim编辑器

vi / vim是Linux上最常用的文本编辑器而且功能非常强大。只有命令，没有菜单，下图表示vi命令的各种模式的切换图。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/tuSaKc6SfPrCdhhPgOiaoBcFPce4tpjykeSGAqQ6vwicJcNf9AtHRpxAb8efawNibqgx2zmkCReVbLYSCuyvXIM6w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**4.1 修改文本**

![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCiapsRoicnGDROYs7K2Nby9K0cPTQwvicia5pg58pPFRbeYQX23bUriaP9c3Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**4.2 定位命令**

![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCiaW9ubIw2eADU27yyNHrgc3qUnjicHWCGV13iaCAcaegLtEXlujR2gyz4w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**4.3 替换和取消命令**

![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCialCmRBeFam8sY6ibjib0qx75hBPrISqQwywtjq4LZESCD7JYwnmuJvhgg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**4.4 删除命令**

![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCiaJddPfgVP6wh2MZB6Rx7gLLtofaBP6s7j1Ao12RKJcvzZt4IWMHIGdw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**4.5 常用快捷键**

![图片](https://mmbiz.qpic.cn/mmbiz_png/YrLz7nDONjFfwIxuqgHaxR6TVoWwicBCia5JB4PkqGw6Qo1icQsc57GaxNUF7AuSzEULiaBomtvcnuhAAicCtpdhqzQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

SaaS