# 使用 Hexo+github来创建自己个性域名的博客

写这篇文章也是为了巩固一下使用Hexo搭建博客的流程，比较流行的大部分也是通过Hexo和GitHub来搭建个人博客，毕竟GitHub提供免费的服务器搭建过程也踩了不少坑，大致总结一下吧，也是通过网上搜索了我遇到的相关情况，把这些问题以及搭建的过程写下来后续我也能看看巩固一下，好了下面开始操练起来了。(目前是基于mac OS操作系统搭建的)
 
 详情请参照Hexo的官方网站：https://hexo.io/zh-cn/docs/
 
# 环境搭建
### nodeJs环境搭建：https://nodejs.org/en/
直接打开官网，下载安装包，进行安装即可。
### git环境搭建：https://git-scm.com/
直接打开官网，下载安装包，进行安装即可。


## GitHub上创建repository
   这里注意一下Repository name一定是GitHub的用户名.github.io的形式，否则不能成功部署。到了这里点击Create repository就算创建成功了一个repository。
   例如笔者的：FrewenWong.github.io
   
## 1.搭建本地Hexo博客系统
首先安装Hexo,执行：

`$ npm install hexo-cli -g`

安装完成之后，显示 输入命令
`hexo -v`

![-w926](https://i.loli.net/2020/04/04/xYs1pTWJbDr6HiM.jpg)


```
$ hexo init FrewenWong（你自己想建立的目录名）
$ cd FrewenWong
$ npm install
$ hexo g  (或者hexo generate)
$ hexo s (或者hexo server)  
```
执行结果，如图所示：

![-w925](https://i.loli.net/2020/04/04/C5o3ikPXDFeUwux.jpg)


可以在http://localhost:4000/ 查看

![-w923](https://i.loli.net/2020/04/04/rqYLDlV1THEcMwa.jpg)

好了，至此本地博客系统已经搭建完毕。

## 2.将Hexo部署到github
```
npm install hexo-deployer-git --save
```
在 Hexo 文件夹下找到 config.yml 文件, 找到其中的 deploy 标签，改成下图所示形式，并保存。
下图是_config.yml的比较重要的配置
```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: https://github.com/FrewenWong/FrewenWong.github.io
  branch: master

```

执行部署命令：

```
$ hexo clean
$ hexo generate
$ hexo deploy
```
现在我们的Hexo已经部署到github.下面我们就可以通过 https://frewenwong.github.io/ 来访问我们的网站。

#### 故障排查

我们更换域名的时候遇到一个问题：
```
The CNAME www.frewen.wang is already taken. Check out https://help.github.com/articles/troubleshooting-custom-domains/#cname-already-taken for more information.
```
解决办法：

https://wangqy.cc/2018/05/26/CNAME/
http://wanghaorui.com/2018/05/10/gitpage%E5%BC%80%E8%AE%BE%E7%9A%84%E9%97%AE%E9%A2%98/





#### gitee 部署

除了可以在github上进行部署，我们也可以在gitee 上进行部署。具体的部署步骤，可以参考官方文档：
https://gitee.com/help/categories/56


## 三.设置Hexo主题
网站：https://hexo.io/zh-cn/docs/themes.html
网站 https://hexo.io/themes/index.html
网址：https://github.com/hexojs/hexo/wiki/Themes

###1.安装
进入/FrewenWong/themes目录下面：
```
git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia
```
###2.配置
修改hexo根目录下的 `_config.yml ： theme: yilia`

###3.更新
```
$ hexo clean
$ hexo generate
$ hexo deploy

<!-- 或者 -->
$ hexo clean && hexo g && hexo d

```


## 四、设置个性域名
参考：https://www.jianshu.com/p/39562a0d8eb6

1.购买域名
现在使用的域名是Github提供的二级域名，也可以绑定为自己的个性域名。购买域名，可以到GoDaddy官网，网友亲切称呼为：狗爹，也可以到阿里万网购买。我是在万网买的，可直接在其网站做域名解析。
我们购买域名的地址：https://wanwang.aliyun.com/


2.域名解析

如果将域名指向一个域名，实现与被指向域名相同的访问效果，需要增加CNAME记录。登录万网，在你购买的域名后边点击：解析–> 添加解析
首先你的ping一下你的域名https://username.github.io/（例如：[https://frewenwong.github.io/](https://frewenwong.github.io/)）来获取的IP地址。

```
 ✘ frewen@bogon  ~/03.Program_Study/19.Blog_Study/01.WorkSpace/FrewenWong  ping frewenwong.github.io
PING frewenwong.github.io (185.199.108.153): 56 data bytes
64 bytes from 185.199.108.153: icmp_seq=0 ttl=55 time=50.734 ms
64 bytes from 185.199.108.153: icmp_seq=1 ttl=55 time=59.878 ms
64 bytes from 185.199.108.153: icmp_seq=2 ttl=55 time=52.631 ms
64 bytes from 185.199.108.153: icmp_seq=3 ttl=55 time=51.384 ms
```
然后设置DNS解析，如图：

![-w925](https://i.loli.net/2020/04/04/QAvTymoBr1ZEV5h.jpg)



然后，我们可以在我们Hexo工程目录下面，新建一个CNAME文件。

```
例如，我的目录就是：
~/03.Program_Study/19.Blog_Study/01.WorkSpace/FrewenWong/source 

```
然后进行文件编辑，修改为下面的内容

```
tech.frewen.wang
```
然后重新部署，并发布我们的博客。

大功告成！


## 五.发布文章
```
$ hexo new  "postName"
```
即为创建一个名为postName.md的文件会建在目录/blog/source/_posts下，postName是文件名，为方便链接不建议掺杂汉字。
文章编辑完成之后执行：
```
hexo generate 			//生成静态页面
hexo deploy			 //将文章部署到Github
```


至此，Mac上搭建基于Github的Hexo博客就完成了，请继续吧。

## 六.设置图床
我们写博客最重要的就是选择自己的图床了。关于图床的相关内容，这里我就不再赘述，大家可以看我的自己的另一篇博客。


至此，Mac上搭建基于Github的Hexo博客就完成了，请继续吧。
