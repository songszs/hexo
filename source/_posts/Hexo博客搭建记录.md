title: Hexo博客搭建记录
date: 2016-01-18 19:27:22
category: 问题记录
tag: hexo  
---

&emsp;&emsp;Hexo是一款基于[Node.js](http://baike.baidu.com/link?url=9BBFfsrnFUV-GOIvdk1sxwv89nntYkV-DxHTCu3wxMNeZd_NFICNxGQFyvNVAm9AYx225IUzRYPKqiRH-Po0K_)的静态博客框架。博客搭建的大致原理就是本地安装hexo，基于hexo框架生成静态博客文件，上传到服务器空间。
### 本地安装hexo
#### 安装nodejs运行环境
&emsp;&emsp;由于要在本地运行hexo生成静态文件，自然要安装nodejs的运行环境，因为hexo是运行在nodejs环境下的。windows下分步安装nodejs运行环境是比较麻烦的，推荐直接下载安装，下载地址[参考此处](https://nodejs.org/en/)。
#### 安装Git工具
&emsp;&emsp;Git是一个分布式代码管理工具，git本身是不支持windows，windows下的客户端大都是封装了Cygwin以支持其在UNIX环境中运行，推荐[安装github客户端](https://desktop.github.com/)。
#### 安装hexo
&emsp;&emsp;安装完github客户端后，打开git shell客户端，执行：
``` bash
$ npm install -g hexo
```
&emsp;&emsp;该命令就是使用nodejs安装包工具安装hexo。然后进入到需要创建hexo的文件夹下执行：
``` bash
$ hexo init
```
&emsp;&emsp;该命令是在该路径下生成hexo静态文件。执行完后便会在该文件夹下生成相应的文件。然后执行：
``` bash
$ hexo generate
$ hexo server
```
&emsp;&emsp;这两条命令分别是生成静态文件，打开本地服务器。然后在浏览器上输入：localhost:4000 即可查看默认静态博客，自此本地环境已经搭建完成。
### GitHub上注册免费GitPages空间
#### 申请免费GitPages空间
&emsp;&emsp;GitHub上新建一个代码库，但是repository name 格式需要 username.github.io ，这样GitHub才会认为是GitPages库。创建完代码库后setting中可以看到如下说明已经成功。
![1](/image/Hexo博客搭建记录/1.png)
#### 生成本地ssh添加
&emsp;&emsp;ssh key可以认为是两个git端的链接秘钥。关于git生成 ssh key，可以[参考此处](http://blog.csdn.net/hustpzb/article/details/8230454/)。生成完成本机的ssh key后需要填写到github上，这样本机git就能和github上代码库进行数据传输了。如图打开GitHub个人设置页面，找到ssh keys，添加一个新的ssh keys即可。
![2](/image/Hexo博客搭建记录/2.png)
### 上传本地hexo静态文件到GitPages空间
#### 配置hexo部署地址
&emsp;&emsp;修改Hexo的配置文件中的部署地址。打开hexo的根目录:
![3](/image/Hexo博客搭建记录/3.png)
&emsp;&emsp;修改_config.yml文件中的结尾的deploy部分如下，其中repo填写刚才在github上申请的gitpages库的地址：
![3](/image/Hexo博客搭建记录/4.png)
#### 上传文件
&emsp;&emsp;本地执行
``` bash
$ hexo generate
$ hexo deploy
```
&emsp;&emsp;其中generate执行后会生成静态网页文件。deploy为部署到github上命令。成功后即可在gitpages上访问到自己的blog了。
### Markdown使用
&emsp;&emsp;Markdown算是一种伪超文本标记语言，使用起来很简单。具体教程可以[参考此处](http://sspai.com/25137)。

&emsp;&emsp;Hexo搭建博客需要一定的技术门槛，这里只是记录一下大致流程。具体细节可自行Google。
