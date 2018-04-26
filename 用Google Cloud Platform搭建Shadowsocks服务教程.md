用Google Cloud Platform搭建Shadowsocks服务教程

# 用Google Cloud Platform搭建Shadowsocks服务教程

** 发表于 2017-06-14 | ** 分类于 [教程 ](http://godjose.com/categories/%E6%95%99%E7%A8%8B/)| ** | ** 阅读次数

经过一天的努力和摸索，终于完成了 Shadowsocks 的搭建并优化提速，遗憾的是没有找到突破 netflix 封锁的办法，希望大神指点迷津。

以下内容分四步
一、Google Cloud Platform虚拟机部署
二、升级VPS内核开启BBR
三、搭建Shadowsocks server
四、设置Shadowsocks server开机启动

本人不是码农，基本算是零基础，相信你照着我下面的步骤也会成功的。
操作平台：PC-win10-64bit
需要工具：能访问Google的网络、VISA信用卡

# 一、Google Cloud Platform虚拟机部署

## 1.申请试用GCP

谷歌云平台可让您构建和主机应用程序和网站，存储数据，并分析对谷歌的可扩展基础架构的数据。
申请地址：[请点击](https://cloud.google.com/free/)

[![申请页面](http://godjose.com/2017/06/14/new-article/GCP1.jpg)](http://godjose.com/2017/06/14/new-article/GCP1.jpg)

登陆你的谷歌账户，必须使用信用卡，而且不能使用虚拟卡，招商银行、中信银行的全币种卡、浦发银行的 VISA 卡均可以通过验证。理论上 VISA 卡均可获得通过，由于我绑过美区的 Google Wallet 所以我选的是美国，选择中国后绑定信用卡会填写个人信息可以如实填写信用卡账单地址，添加信用卡和购物网站一样，不赘述。成功后会于扣款1刀，验证卡片后会返还。
GCD 现在免费赠送300刀期限是一年，也就是300刀和一年时间谁先用完就以谁为准，结束之后你不点继续使用时不会扣费的。

## 2.修改防火墙

这一步很多教程是最后再设置，但是我选择提前设置，这样后面部署好SS服务就可以直接使用了。

直接访问：[请点击](https://console.cloud.google.com/networking/firewalls/list)

或者在菜单中依次点击 【网络】 –> 【防火墙规则】 –> 【创建防火墙规则】
如图选择相关项目。

[![修改防火墙](http://godjose.com/2017/06/14/new-article/GCP2.jpg)](http://godjose.com/2017/06/14/new-article/GCP2.jpg)

设置按照上图来设置，名称自己取，IP 地址范围：`0.0.0.0/0`
保存后会生成规则，请耐心等待。

## 3.获取静态IP

这一步很重要，只有有了静态IP，你后面部署的SS服务才能用。
直接访问 [请点击](https://console.cloud.google.com/networking/addresses/list)

或者在菜单中依次点击 【网络】–> 【外部 IP 地址】 –> 【保留静态 IP】

[![静态IP](http://godjose.com/2017/06/14/new-article/GCP3.jpg)](http://godjose.com/2017/06/14/new-article/GCP3.jpg)

**名称自定义即可**
PS：静态 IP 只能申请一个！！！
大陆速度最佳的机房是台湾彰化的机房了，asia-east1-c对大陆最友好
还有东京亚洲东区，也就是东京机房了asia-northeast1-a

## 4.创建计算引擎

直接访问 ： [请点击](https://console.cloud.google.com/compute/instances)
或者在菜单中依次点击 【计算引擎】–> 【创建实例】

[![创建计算引擎](http://godjose.com/2017/06/14/new-article/GCP4.jpg)](http://godjose.com/2017/06/14/new-article/GCP4.jpg)

机器类型里面选最便宜的那个微型就够用，启动磁盘选Ubuntu16.04LTS，其他系统也可以只要是你会部署的，我只能参照其他教程搞定Ubuntu1604所以我选择这个

[![给vps配置静态](http://godjose.com/2017/06/14/new-article/GCP5.jpg)](http://godjose.com/2017/06/14/new-article/GCP5.jpg)

这里内部ip选择你刚刚得到的那个静态IP，这样虚拟机就完成了设置

[![配置后](http://godjose.com/2017/06/14/new-article/GCP6.jpg)](http://godjose.com/2017/06/14/new-article/GCP6.jpg)

Google Cloud 自带的浏览器 SSH 挺好用的，推荐使用！另外如果你是chrome浏览器的话推荐SSH for Google Cloud Platform 这个插件给你，chrome内扩展程序安装地址是：

[请点击](https://chrome.google.com/webstore/detail/ssh-for-google-cloud-plat/ojilllmhjhibplnppnamldakhpmdnibd)

点击上图的ssh后就直接弹出来

[![SSH](http://godjose.com/2017/06/14/new-article/GCP7.jpg)](http://godjose.com/2017/06/14/new-article/GCP7.jpg)

至此，第一部分GCD上的准备工作和部署全部完成，

# 二、升级vps内核开启BBR

由于众所周知的原因，单纯部署完shadowsocks服务之后速度都不会太理想，即使你选择的是台湾、日本这种很好的线路，依然会存在丢包和不稳定的情况。因为这一步会需要重新启动，所以放在部署SS服务之前。

这下面的东西我都不懂，扒自[这个教程](http://0x00.online/2017/04/29/ubuntu1604upgradekernel/) 你照着一步一步来绝对没问题。

> BBR 目的是要尽量跑满带宽, 并且尽量不要有排队的情况, 效果并不比速锐差。
> Linux kernel 4.9+ 已支持 tcp_bbr 下面简单讲述基于 KVM 架构 VPS 如何开启。

## 准备工作

进入ssh后不是root权限，先获取root权限

```
sudo –i

```

更新系统（两行命令分开执行，第二步等待时间较长，会出现####和进度百分百，耐心等）

```
apt update
apt upgrade

```

查看当前内核版本

```
uname –a

```

然后你会发现发现版本低于 4.9
安装新内核

```
apt install linux-image-4.10.0-20

```

卸载旧内核

```
apt autoremove

```

启用新内核

```
update-grub

```

重启

```
Reboot

```

验证内核版本

```
uname –r

```

看到如下类似如下回显，版本号为4.10.0-20-generic

## 启用BBR

写入配置

```
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf

echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf

```

配置生效

```
sysctl -p

```

检验

```
lsmod | grep bbr

```

看到回显`tcp_bbr 20480 0`说明已经成功开启 BBR
不需要重新启动，我们接下来直接开始在虚拟机部署SS

# 三、搭建Shadowsocks server

首先更新一下 apt-get 软件包

```
sudo apt-get update

```

然后通过 apt-get 安装 python-pip

```
sudo apt-get install python-pip

```

完成之后使用 pip 安装 shadowsocks 服务

```
sudo pip install shadowsocks

```

[![安装成功](http://godjose.com/2017/06/14/new-article/GCP8.jpg)](http://godjose.com/2017/06/14/new-article/GCP8.jpg)

说明安装成功

然后我们需要创建一个 shadowsocks server 的配置文件，可以直接建在当前用户目录下

```
sudo vim /etc/ss-conf.json

```

回车之后会进入这个创建的文件，按键盘上的 insert键会进入编辑，然后把下面的内容输入进去。按ESC键会发现左下角的insert消失，shift+：这个组合键左下角出现：输入wq回车就保存退出文件。红色字体分边是端口和密码，设置成你想要的就行了

> {
> “server”:”0.0.0.0”,
> “server_port”:8838,
> “local_address”:”127.0.0.1”,
> “local_port”:1080,
> “password”:”123456”,
> “timeout”:600,
> “method”:”aes-256-cfb”
> }

7.31日更新：上面的内容有网友反映直接复制会报错，请注意以下两点：1代码的全部内容必须为英文半角输入，2server_port与password后面的数字内容请自定义，也就是你之后在shadowsocks客户端上配置使用的端口和密码。

最后用这个配置文件启动 shadowsocks 服务

```
sudo ssserver -c /etc/ss-conf.json -d start

```

但是服务器可能会自动重启，这样的话就需要每次手动开启SS服务，很麻烦而且还会遇到要用梯子但是梯子在墙外的这种困境，怎么办呢？那么我们进入第四步，写脚本让系统开机后自动启动ss服务。

# 四、设置Shadowsocks server开机启动

这一点让我尝试了很久，搜了很多教程说什么在系统的rc.local里面添加启动命令就行等等，试过很多次，各种折腾改路径都不行，最后还是参考[这个教程](https://my.oschina.net/oncereply/blog/467349)把ss的配置文件修改成自己的，就成功了。

创建脚本 /etc/init.d/shadowsocks

```
sudo vim /etc/init.d/shadowsocks

```

进入文件后添加以下内容,方法与前面创建ss-conf.json这个文件一样，使用insert键、shif+：、wq回车保存等等

> *#*!/bin/sh
>
> start(){
> 　　　ssserver -c /etc/shadowsocks.json -d start
> }
> stop(){
> 　　　ssserver -c /etc/shadowsocks.json -d stop
> }
> case “$1” in
> start)
> 　　　start
> 　　　;;
> stop)
> 　　　stop
> 　　　;;
> reload)
> 　　　stop
> 　　　start
> 　　　;;
> *)
> 　　　echo “Usage: $0 {start|reload|stop}”
> 　　　exit 1
> 　　　;;
> esac

然后增加这个文件的可执行权限

sudo chmod +x /etc/init.d/shadowsocks

创建文件 /etc/init/shadowsocks.conf

sudo vim /etc/init/shadowsocks.conf

内容直接复制如下

> start on (runlevel [2345])stop on (runlevel [016])pre-start script
> /etc/init.d/shadowsocks start
> end script
>
> post-stop script
> /etc/init.d/shadowsocks stop
> end script

执行

```
sudo update-rc.d shadowsocks defaults

```

然后就添加到开机启动中了
最后你可以reboot测试一下看是否成功，若未成功就check一下第四步哪里有问题。
台湾这个线路经过优化之后，测试Youtube1080P是不会卡顿的，能达到你带宽的上限速度。



# 文档链接
http://godjose.com/2017/06/14/new-article/

