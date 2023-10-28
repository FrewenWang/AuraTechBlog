---
title: 使用Hexo搭建个人技术博客网站
date: 2014-07-12 00:00:00
updated: 2014-07-12 00:00:00
tags: [博客,Blog]
---

## 安装NodeJS

我们先看一下自己本地是否有node环境

```shell
(base)  frewen@FreweniMacBookPro  ~  npm -v
8.11.0
(base)  frewen@FreweniMacBookPro  ~  node -v
v16.15.1
```

可以看到我的电脑上已经有了Node环境，我们不再需要安装Node环境。如果您的电脑安装NodeJS具体可参见：https://nodejs.org/en/

直接打开官网，下载安装包，进行安装即可。



## 安装GIT

安装git的步骤具体参见：https://git-scm.com/



## 安装Hexo

在cmd或者其他命令行工具下输入如下：

```shell
npm install hexo-cli -g

// 安装完成之后，显示 输入命令
hexo -v

(base) ➜  AuraTechBlog git:(main) ✗ hexo -v
INFO  Validating config
INFO  
  ===================================================================
                                                                     
      #####  #    # ##### ##### ###### #####  ###### #      #   #    
      #    # #    #   #     #   #      #    # #      #       # #     
      #####  #    #   #     #   #####  #    # #####  #        #     
      #    # #    #   #     #   #      #####  #      #        #      
      #    # #    #   #     #   #      #   #  #      #        #    
      #####   ####    #     #   ###### #    # #      ######   #  

                            4.4.0
  ===================================================================
hexo: 6.3.0
hexo-cli: 4.3.0
os: darwin 21.6.0 12.6

node: 18.11.0
v8: 10.2.154.15-node.12
uv: 1.43.0
zlib: 1.2.11
brotli: 1.0.9
ares: 1.18.1
modules: 108
nghttp2: 1.47.0
napi: 8
llhttp: 6.0.10
openssl: 3.0.5+quic
cldr: 41.0
icu: 71.1
tz: 2022b
unicode: 14.0
ngtcp2: 0.8.1
nghttp3: 0.7.0
```

#### 初始化博客

```shell
$ hexo init <your_blog>
$ cd <your_blog>
$ npm install
```









### 设置博客主题

文章参考：https://butterfly.js.org/

安装主题的网站：https://hexo.io/themes/

这是我比较喜欢的主题：https://github.com/jerryc127/hexo-theme-butterfly

具体安装当时见主题的介绍页面

安装命令：

```
// npm安装
npm i hexo-theme-butterfly

// 更新
npm update hexo-theme-butterfly
```

还有另外一种安装方式：

```shell
git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
# 如果想要安裝比較新的 dev 分支，可以
git clone -b dev https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
```

修改 Hexo 根目錄下的 _config.yml，把主題改為butterfly

```shell
theme: butterfly
```



关于这个主题还有一系列的优化，咱们后续在看：https://butterfly.js.org/posts/21cfbf15/







## 部署到GitPage

### GitHub上创建repository

这里注意一下Repository name一定是GitHub的用户名.github.io的形式，否则不能成功部署。到了这里点击Create repository就算创建成功了一个repository。
例如笔者的：[FrewenWong.github.io](http://frewenwong.github.io/)

<img src="images/image-20220918234435993.png" alt="image-20220918234435993" style="zoom: 33%;" />



参照教程：https://hexo.io/zh-cn/docs/one-command-deployment

首先，安装：

```
npm install hexo-deployer-git --save
```

修改根目录下的\_config.xml文件,修改内容如下：

```shell
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repo: https://github.com/FrewenWong/FrewenWong.github.io.git
  branch: master

```

结果出现如下错误：

```shell
remote: Support for password authentication was removed on August 13, 2021.
remote: Please see https://docs.github.com/en/get-started/getting-started-with-git/about-remote-repositories#cloning-with-https-urls for information on currently recommended modes of authentication.
fatal: Authentication failed for 'https://github.com/FrewenWong/FrewenWong.github.io.git/'
FATAL Something's wrong. Maybe you can find the solution here: https://hexo.io/docs/troubleshooting.html
Error: Spawn failed
    at ChildProcess.<anonymous> (/Users/frewen/03.ProgramSpace/01.WorkSpace/02.GithubBlog/FrewenTechBlog/node_modules/hexo-util/lib/spawn.js:51:21)
    at ChildProcess.emit (node:events:527:28)
    at Process.ChildProcess._handle.onexit (node:internal/child_process:291:12)
```

大致意思是，密码验证于2021年8月13日不再支持，也就是今天intellij不能再用密码方式去提交代码。请用使用 **personal access token** 替代。

我们修改成：

```
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repo: git@github.com:FrewenWong/FrewenWong.github.io.git
  branch: master
```



## 设置个性域名

1.购买域名
现在使用的域名是Github提供的二级域名，也可以绑定为自己的个性域名。购买域名，可以到GoDaddy官网，网友亲切称呼为：狗爹，也可以到阿里万网购买。我是在万网买的，可直接在其网站做域名解析。
我们购买域名的地址：https://wanwang.aliyun.com/

2.域名解析
如果将域名指向一个域名，实现与被指向域名相同的访问效果，需要增加CNAME记录。登录万网，在你购买的域名后边点击：解析–> 添加解析
首先你的ping一下你的域名https://username.github.io/（例如：https://frewenwong.github.io/）来获取的IP地址。

```shell
(base)  frewen@FreweniMacBookPro  ~/03.ProgramSpace/01.WorkSpace/02.GithubBlog/FrewenTechBlog  ping frewenwong.github.io
PING frewenwong.github.io (185.199.108.153): 56 data bytes
64 bytes from 185.199.108.153: icmp_seq=0 ttl=48 time=279.827 ms
64 bytes from 185.199.108.153: icmp_seq=1 ttl=48 time=202.576 ms
64 bytes from 185.199.108.153: icmp_seq=2 ttl=48 time=316.546 ms
64 bytes from 185.199.108.153: icmp_seq=3 ttl=48 time=203.390 ms
```

然后设置DNS解析，如图：

然后，我们可以在我们Hexo工程目录下面，新建一个CNAME文件。

```shell
cd FrewenTechBlog
cd source
touch CNAME
open CNAME
// 输出域名
frewen.wang
```



## 图床设置

我们写博客最重要的就是选择自己的图床了。关于图床的相关内容，这里我就不再赘述，大家可以看我的自己的另一篇博客。

至此，Mac上搭建基于Github的Hexo博客就完成了，请继续吧。





## 文章发布

```shell
hexo new  "postName"
```



