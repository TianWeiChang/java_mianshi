Spring Boot 实现万能文件在线预览

你好，我是田哥

推荐一个用 Spring Boot 搭建的文档在线预览解决方案：kkFileView，一款成熟且开源的文件文档在线预览项目解决方案，对标业内付费产品有永中office、office365、idocv等，免费！

**项目地址：**

- 支持 office/pdf/cad 等办公文档
- 支持 txt/java/php/py/md/js/css 等所有纯文本
- 支持 zip/rar/jar/tar/gzip 等压缩包
- 支持 jpg/jpeg/png/gif 等图片预览（翻转，缩放，镜像）
- 使用 Spring Boot 开发，预览服务搭建部署非常简便
- rest 接口提供服务，跨平台特性 (java/php/python/go....) 都支持，应用接入简单方便
- 抽象预览服务接口，方便二次开发，非常方便添加其他类型文件预览支持

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBof4H8KX9sqthxXvZpNIvAuMmFIeZsuj5Yt2Oh5cvGe67oqYJv7zTv1A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBojo91dsiatWXxlpT1d4l27EPcgMpGuA2nFrupeQHZibYojsezw1P1Df5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

支持doc/docx文档预览，word预览有两种模式：一种是每页word转为图片预览，另一种是整个word文档转成pdf，再预览pdf。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBo4uWick6hu7D5kkElPovwicjBibx4Hibt7rcaMiac5O4fETeLtUibLh81JHvw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBo71IL99P7CEGAzunJjdxjy66lL7Dt0PKuKCjVIFqWttxibl2yQX62poQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBon6Xo2fnQiaMbFmBJ94NgoxVCb1o2aLeO1urSEiag7QrakXb24UdxlO5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBoOgvVWBYKWfctzXNs4m7u1VGicb9oibOB4CIicTNQ2QaVrJqOeSyOqvuAg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBoWBOObdhZhsrRSe20NsvauKEcmm8mzSN2ZqPpF7AO1krtawkad9KY3g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可点击压缩包中的文件名，直接预览文件，预览效果如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBot7hTEv0yNCaD5mtOHctia2yGTw4Y0Ub4afkcwn3kR0bviaN3oatjoLOQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)理论上支持所有的视频、音频文件，由于无法枚举所有文件格式，默认开启的类型如下：mp3/wav/mp4/flv

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBoOQ3XEIqysfJhQ1nibRIR5WQFrqSYELexJceG4f2Z9EaB3YG9WJiaheCg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsiaQRR140lERFnASEHEnoZBo0OKnCfbEtAM7yP8KT68tV7khNibpSvJ8E1K0qrHicvzobLic8fmvytkqw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

当然，以上展示的只是部分格式文件的预览效果，

如果你想自己亲手部署一下，那就通过后面的链接，前往项目主页查看具体的操作文档吧：https://gitee.com/kekingcn/file-online-preview

阿萨德