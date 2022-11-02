# 20 个最常用的 Git 命令，码住！

你好，我是田哥



上周一位星友去面试，就因为简历上写了一句：掌握开发工具Git。于是面试官就开始抠问各种命令，不过，最后还是轻松拿下。

下面我给大家整理一些常用命令，希望大家能抽几分钟时间看看，不管是工作还是面试，这都是很有用。

以下是这些Git命令：

> git config
>
> git initgit clone
>
> git add
>
> git commit
>
> git diff
>
> git reset
>
> git status
>
> git rm
>
> git log
>
> git show
>
> git tag
>
> git branch
>
> git checkout
>
> git merge
>
> git remote
>
> git push
>
> git pull
>
> git stash

下面让我们逐一介绍。

**Git 命令**

git config

用法：git config –global user.name “[name]”  

用法：git config –global user.email “[email address]”

该命令将分别设置提交代码的用户名和电子邮件地址。

**git init**

用法：git init [repository name]

该命令可用于创建一个新的代码库。

**git clone**

用法：git clone [url]

该命令可用于通过指定的URL获取一个代码库。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWo5HVibezGvDNma9rNxFibicSEiaBGbHUfDawlxzt9LlJcict4icC7z8gSHM7Q/640?wxfrom=5&wx_lazy=1&wx_co=1)

**git add**

用法：git add [file]

该命令可以将一个文件添加至stage(暂存区)。

用法：git add *

该命令可以将多个文件添加至stage(暂存区)。

**git commit**

用法：git commit -m “[ Type in the commit message]”  

该命令可以在版本历史记录中永久记录文件。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWokDhbqLSLCW52dHW3S5iaVYNDdKXicicHgicfx3xDrvqgC7I87ozicHBqMsA/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git commit -a

该命令将提交git add命令添加的所有文件，并提交git add命令之后更改的所有文件。

**git diff**

用法：git diff

该命令可以显示尚未添加到stage的文件的变更。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWonnBlsm5ZIqiah6VOsNZ4mddYV7gsPyWVs45oQO5qloLMIia3J4Jiasjng/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git diff –staged

该命令可以显示添加到stage的文件与当前最新版本之间的差异。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWoVI7ZKQ6EbRX7R0STy6S8gvpsMm2rok04Vnia4a6CAeWibCRjicdHbYibkA/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git diff [first branch] [second branch]

该命令可以显示两个分支之间的差异。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWoCB6lOo4ZQ4gVRYNvnPOibkboqVgsVb16gBbxMcl7tyhOA1l0t5wTARw/640?wxfrom=5&wx_lazy=1&wx_co=1)

**git reset**

用法：git reset [file]

该命令将从stage中撤出指定的文件，但可以保留文件的内容。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWopJ2BcphOBq8fyUJiaZdIdl8x7RTMEEjsrVn05WMRqAfBmZAE9P7iauxA/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git reset [commit]

该命令可以撤销指定提交之后的所有提交，并在本地保留变更。

用法：git reset –hard [commit]

该命令将丢弃所有的历史记录，并回滚到指定的提交。

**git status**

用法：git status

该命令将显示所有需要提交的文件。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWojibFuFXx2j8ljGQwD05HJAaDKVRm9v6H9XjJW5rLHt96zOJxI6qWQHQ/640?wxfrom=5&wx_lazy=1&wx_co=1)

**git rm**

用法：git rm [file]

该命令将删除工作目录中的文件，并将删除动作添加到stage。

**git log**

用法：git log

该命令可用于显示当前分支的版本历史记录。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWoVBrx5SOzjfPic7BywbQfkYhkp7kkmnKHmL3BUmibCkUzZcOROF9vQdicw/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git log –follow[file]

该命令可用于显示某个文件的版本历史记录，包括文件的重命名。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWomjhE6D6tib6jwBj4dAaSeMaorgR0coMRGTDnbWYRmmFTVAWqdQDjGlw/640?wxfrom=5&wx_lazy=1&wx_co=1)

**git show**

用法：git show [commit]

该命令经显示指定提交的元数据以及内容变更。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWoBysJJVAbNgxSHfP2CT665Hdb4XL8KW3nMI1SsLHyp56eMd8pLM5Y6Q/640?wxfrom=5&wx_lazy=1&wx_co=1)

**git tag**

用法：git tag [commitID]

该命令可以给指定的提交添加标签。

**git branch**

用法：git branch

该命令将显示当前代码库中所有的本地分支。

用法：git branch [branch name]

该命令将创建一个分支。

用法：git branch -d [branch name]

该命令将删除指定的分支。

**git checkout**

用法：git checkout [branch name]

你可以通过该命令切换分支。

用法：git checkout -b [branch name]

你可以通过该命令创建一个分支，并切换到新分支上。

**git merge**

用法：git merge [branch name]

该命令可以将指定分支的历史记录合并到当前分支。

**git remote**

用法：git remote add [variable name] [Remote Server Link]

你可以通过该命令将本地的代码库连接到远程服务器。

**git push**

用法：git push [variable name] master

该命令可以将主分支上提交的变更发送到远程代码库。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWo1gxM4icXsibuLq84DFdOJ0EdNQl5fesP3IQ5GBREDz4aBjTy3klXVCAw/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git push [variable name] [branch]

该命令可以将指定分支上的提交发送到远程代码库。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWoCMZJ0SjSub6lYHnOBMwW0VqblwUlQuQvDq5EtnRee5hLicqu3EGibNYg/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git push –all [variable name]

该命令可以将所有分支发送到远程代码库。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWoxGlVlLUAml0LoqnuBSernNQAZ6KkUXlz1FVFIs4JubYwqlYRMPJtrw/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git push [variable name] :[branch name]

该命令可以删除远程代码库上的一个分支。

**git pull**

用法：git pull [Repository Link]

该命令将获取远程服务器上的变更，并合并到你的工作目录。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWok0DRicAYPyku8ibQThoZEGeOGoFqYxKsHOedELkLeFkBIp9uQ1nu06Uw/640?wxfrom=5&wx_lazy=1&wx_co=1)

**git stash**

用法：git stash save

该命令将临时保存所有修改的文件。

用法：git stash pop

该命令将恢复最近一次stash（储藏）的文件。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oTKHc6F8tsjGPVUGJhaIyqMTAKSdqAWo9hYVdVF8gc88jclR17QavzH3j2dZcvH66ptlNFsibkHtmQJu7KIIh1w/640?wxfrom=5&wx_lazy=1&wx_co=1)

用法：git stash list

该命令将显示stash的所有变更。

用法：git stash drop

该命令将丢弃最近一次stash的变更。

