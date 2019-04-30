---
title: "使用Docker多阶段构建和打包python项目"
date: 2019-04-29T21:11:55+08:00
categories: ["technology"]
tags: ["docker", "python", "ci"]
---

最近在新厂做了一个Python技术栈相关项目，是通过[Gitlab CI](https://docs.gitlab.com/ee/ci/)用[docker-in-docker](https://hub.docker.com/_/docker)方式[多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)项目的，遇到一些问题，记录一下解决方法。

## Python项目打包发布

首先要从Python项目的打包发布说起。

### 压缩包整包发布流程

其实很多的Python开发者是这么干的，这种方式只是给源码打个压缩包，然后通过`pip freeze > requirements.txt`生成requirements.txt文件记录项目依赖。生产环境安装好了二进制依赖，通过`pip install -r requirements.txt`安装完全部依赖，把项目压缩包解压，拉起进程就完事了。

### wheel包发布流程

[wheel](https://pythonwheels.com/)包是一种标准。这种方式跟源码压缩包的方式不同的是，setup.py可以容易把requirements.txt内容打包进wheel包，容易配置各种依赖的静态文件放进wheel包（比如Django编译完的翻译文件），容易把需要在bin目录生成一个可执行文件的脚本放进wheel包。生产环境在安装完二进制依赖和`pip install your_package_name.whl`之后拉起进程就完事了。

### 这两种方式的问题

以上两种方式，运行时二进制依赖需要安装，编译时依赖也需要安装，源码包需要安装。
那么问题就来了，使用Docker的多阶段构建，是为了构建过程和最终打包镜像的过程可以分离并能在一个Dockerfile体现出来。通常构建过程是需要执行代码静态扫描和自动化测试的，也就是扫描成功后，需要安装一次依赖执行测试，测试通过了，再继续之后的打包流程。
如此打好的包，在Docker的最终构建阶段，还是需要安装一次编译时依赖和运行时依赖，网络传输耗时和编译耗时会重新体现一次，并且在Dockerfile里面，二进制依赖安装的语句不写注释就无法体现哪些时编译时需要，哪些是运行时需要。


## 多阶段构建多耗时问题的解决

既然清楚问题出在哪里了，那就可以针对性的解决。问题出在需要多下载一次和多编译一次，那是不是有办法可以只下载一次和只编译一次，把结果打包在一起，两个阶段只需要解包就行了呢？pip这个工具给了答案。

首先，看看pip提供了啥功能：
```shell
pip -h
```
然后你会很惊喜的发现有个`pip wheel --wheel-dir=dir -e .`指令可以把setup.py定义的依赖都用wheel包的形式导出到某个指定的目录。而且导出的包已经包含编译好的二进制依赖。

那这个存放依赖包的目录怎么用呢？可以通过
```shell
pip install --no-index --find-links=dir your_package_name.whl
```
强行指定不走网络，只使用本地的目录安装依赖。

如此，问题得解。


## 题外话

在打包的时候也思考了下如果项目不能源码发布的时候，应该怎么处理。其实Python3之后，到源码目录执行：
```shell
python -m compileall -b .  # 产生pyc文件
```
然后递归目录把py文件和__pycache__目录删除掉即可。
