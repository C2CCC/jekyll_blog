---
title: CentOS 7部署Node.js+MongoDB：在VPS上从安装到Hello world
categories: Linux
tags:
- CentOS
- Node.js
- MongoDB
- VPS
---

* 目录
{:toc}

写好代码，花钱买了VPS，看着Charges一直上涨却无从下手？记一位新手司机从购买VPS到成功访问的过程

# 0.购买VPS

首先，选择VPS提供商，部署一个新的服务器（Deploy New Server），我使用的是Vultr提供的VPS
操作系统可以自由选择，我这边使用的是CentOS 7，选择其他操作系统的胖友可以搜一下相应操作系统的部署教程

# 1.使用PuTTY连接远程VPS

安装[PuTTY][]

打开PuTTY，在 Host Name(or IP address) 那一栏填上VPS提供商给你的IP地址，然后点Open开启一个新会话（也可以点底下的Save保存一下，下次直接双击Saved Sessions中保存的的会话打开就行，无须输入IP）

![PuTTY interface](http://img.blog.csdn.net/20160829010214660)

打开后会出现一个Terminal，显示login as: ，这边输入VPS提供商给你的Username，默认应该是root

然后出现root@your ip address 's password: ，输入VPS提供商提供的Password（可以在本机上复制，然后在窗口中点击鼠标右键，复制的内容就被粘贴了），然后回车
注意：输入密码的时候不会显示你输入的东西

成功登录

# 2.准备工作

用yum安装gcc

```bash
yum -y install gcc make gcc-c++ openssl-devel wget
```

# 3.安装Node.js

这边以v4.5.0版本、使用源码安装为例，也可以使用EPEL

先用cd命令进入到你要安装Node.js的目录

到Node.js官网上复制源码的地址，然后执行：

```bash
wget https://nodejs.org/dist/v4.5.0/node-v4.5.0.tar.gz
```

![nodejs website](http://img.blog.csdn.net/20160829010402367)

然后解压提取文件

```bash
tar zxvf node-v4.5.0.tar.gz
```

执行后会生成node-v4.5.0文件夹，cd进入,里面有个configure文件

配置并编译#这步执行得比较久，可以先去喝杯咖啡

```bash
./configure
make
```

然后安装

```bash
make install
```

使用node -v检查是否安装成功

# 4.安装MongoDB

这边以v3.2.9、使用yum安装为例

MongoDB官网上提供了用yum安装的[教程][Install MongoDB Community Edition on Red Hat Enterprise or CentOS Linux &mdash; MongoDB Manual 3.2]

我写了一篇翻译别人教程的[Blog](http://blog.csdn.net/azureternite/article/details/52349304)，跟官网上的类似
步骤是类似的，但是我们用官网上的命令进行安装，比较稳妥，那篇翻译的教程仅供理解用

# 5.给MongoDB添加用户认证

```bash
mongo
use db
db.createUser({user:'',pwd:'',roles:[{role:'readWrite',db:'db'}]});  #添加db数据库下的用户，拥有读写权限
db.system.users.find().pretty()  #查看该数据库下所有的用户
```

记得在admin数据库下添加一个root用户用于以后关闭服务器
即 use admin，role: 'root'

有关新建用户的更多信息请见[这里][【Mongodb】3.0 配置身份验证db.createUser（）说明 - NoSQL论坛 -  51CTO技术论坛_中国领先的IT技术社区]

添加了用户后，在启动MongoDB时加上--auth参数即可开启用户认证

附上一条mongoose通过用户认证连接db的代码

```javascript
mongoose.connect('mongodb://username:password@host:port/database?options...');
```

# 6.复制你的应用到VPS服务器

如果你的代码还没复制到VPS服务器上，你可以使用git，svn，ftp等方式放上去
我用的是git

安装git，使用yum安装

```bash
yum -y install git
```

然后就可以愉快地使用git clone了

# 7.开启端口

CentOS 7采用了firewalld防火墙，如果没有开启端口，则外网无法通过ip来访问服务器上的Node应用。

比如Node.js默认用了3000端口，所以我们需要开启相应的端口

查询端口是否开启：

```bash
firewall-cmd --query-port=3000/tcp
```

如果显示no，则没有开启端口

开启端口：

```bash
firewall-cmd --add-port=3000/tcp
```

也可以修改Node监听的端口，比如修改为80端口，然后再开启80端口

# 8.后台运行

## 后台运行MongoDB

在启动MongoDB时，加上--fork参数，即可生成一个子进程，当子进程成功运行，则父进程就会被停止，这时候便实现了后台运行MongoDB，可以关闭当前的终端

```bash
mongod --dbpath /usr/local/mongodb/data --logpath /usr/local/mongodb/logs/log.log --logappend --auth --fork
```

如果要关闭后台MongoDB，则需要通过刚才添加的root用户调用shutdownServer()方法关闭
顺带一提，MongoDB没有正常关闭的话会很麻烦的，有时候还会造成一些严重后果

```bash
mongo
use admin
db.auth('usr', 'pwd')  #用root用户登录，usr、pwd为root用户的用户名和密码
db.shutdownServer()
```

## 后台运行Node.js

这边后台启动Node.js使用一个Node.js的模块forever，可以输出错误和日志
当然还有其他的方法，比如nohup啥的

全局安装forever

```bash
npm install -g forever
```

cd进入应用程序的目录

如果是普通的Node应用

```bash
forever start app.js
```

如果用的Express

```bash
forever start -c 'npm start' ./
```

有关如何停止forever以及其他操作请参考[forever][]官网及Google

我之前是看[CNode社区里的方法][如何 停止node进程？ - CNode技术社区]直接杀掉Node的进程 pkill node ，可以，很暴力

# 9.Hello world

至此，你应该可以从外网通过ip+端口访问到你的Node应用了，你也可以绑定个域名之类的

# 10.附加

## 开机启动

可以将MongoDB和Node通过编辑/etc/rc.local加入到开机启动中
这里我没有试验过，先挂上一篇[教程][Linux下设置MongoDB开机自启动_Linux教程_Linux公社-Linux系统门户网站]

## Vim

使用Linux免不了的一个问题就是 如何编辑文件
久闻Vim之大名，今日有幸相会

```bash
yum -y install vim
```

其实刚入门的小白用Vim，只要掌握一些基础的操作，用起来也是很爽的

附一个[Vim简明教程][Vim简明教程【CoolShell】 - 飘过的小牛 - 博客频道 - CSDN.NET]

# 参考

* [Centos 安装 NodeJS - Hamy - 博客园][]
* [如何在CentOS 7安装Node.js_Linux教程_Linux公社-Linux系统门户网站][]
* [Install MongoDB Community Edition on Red Hat Enterprise or CentOS Linux &mdash; MongoDB Manual 3.2][]
* [【Mongodb】3.0 配置身份验证db.createUser（）说明 - NoSQL论坛 -  51CTO技术论坛_中国领先的IT技术社区][]
* [当mongodb开启用户认证后(auth=true),如何使用mongoose链接数据库 - CNode技术社区][]
* [Centos 7 开启端口 - 鍒樻爧 - 博客园][]
* [mongodb设置后台运行的方法_MongoDB_脚本之家][]
* [Forever with \`npm start\` · Issue #540 · foreverjs/forever](https://github.com/foreverjs/forever/issues/540)
* [如何 停止node进程？ - CNode技术社区][]
* [Linux下设置MongoDB开机自启动_Linux教程_Linux公社-Linux系统门户网站][]
* [Vim简明教程【CoolShell】 - 飘过的小牛 - 博客频道 - CSDN.NET][]





[PuTTY]: http://www.putty.org/
[forever]: https://github.com/foreverjs/forever

[Centos 安装 NodeJS - Hamy - 博客园]: http://www.cnblogs.com/hamy/p/3632574.html
[如何在CentOS 7安装Node.js_Linux教程_Linux公社-Linux系统门户网站]: http://www.linuxidc.com/Linux/2015-02/113554.htm
[Install MongoDB Community Edition on Red Hat Enterprise or CentOS Linux &mdash; MongoDB Manual 3.2]: https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/
[【Mongodb】3.0 配置身份验证db.createUser（）说明 - NoSQL论坛 -  51CTO技术论坛_中国领先的IT技术社区]: http://bbs.51cto.com/thread-1146654-1.html
[当mongodb开启用户认证后(auth=true),如何使用mongoose链接数据库 - CNode技术社区]: https://cnodejs.org/topic/53e0a75abd3cc3e50b8179b3
[Centos 7 开启端口 - 鍒樻爧 - 博客园]: http://www.cnblogs.com/mliudong/p/4529612.html
[mongodb设置后台运行的方法_MongoDB_脚本之家]: http://www.jb51.net/article/54754.htm
[Forever with `npm start` · Issue #540 · foreverjs/forever]: https://github.com/foreverjs/forever/issues/540
[如何 停止node进程？ - CNode技术社区]: https://cnodejs.org/topic/510dd169df9e9fcc581cb97f
[Linux下设置MongoDB开机自启动_Linux教程_Linux公社-Linux系统门户网站]: http://www.linuxidc.com/Linux/2011-07/39149.htm
[Vim简明教程【CoolShell】 - 飘过的小牛 - 博客频道 - CSDN.NET]: http://blog.csdn.net/niushuai666/article/details/7275406