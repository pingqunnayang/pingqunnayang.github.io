---
layout: post
title: 安装shadowsocks访问谷歌
titlename: 安装shadowsocks访问谷歌
date: 2017-07-27 15:18:23
category: 其他
tags: 工具 娱乐
description:
---

<p style="text-indent: 2em">之前一直用chrome的插件『谷歌访问助手』来访问谷歌，一直都比较文档，最近想关注下国外的技术，本打算在网上买个VPN，发现greenVpn都关了，其他的感觉不靠谱，所以考虑在国外租用一个vps，正好向弄台服务器搭建77的个人相册网站，今天已经全部搞定，PC/手机访问速度相当OK，详细列下步骤。

##VPS选择

没有过多的考虑，直接阿里云国际版，毕竟以后搭建服务出问题都比较方便，网上评价香港服务器速度不错，于是开始注册[阿里云](https://account.alibabacloud.com) ，大陆注册需要走部分弯路，简单说下步骤：

1. 开始注册时请选择美国，不要选择香港等其他地方，因为账户注册完成后需要注册国的手机号进行验证，美国有网站支持在线注册手机，香港和其他国家没有测试。
2. 注册完成后如果选择购买云服务器会跳转到完善个人信息，需要完善手机号和支付方式，如下界面![Alt text](/public/img/other/shadowsocks_1.png)
3. 美国手机号可以跳转https://www.textnow.com/注册帐号，注册完帐号就会产生一个可用的手机号，阿里云下图左上角手机号，输入不需要带括号和中划线。
![Alt text](/public/img/other/shadowsocks_4.png)
4. 注册完手机号下一步是添加支付方式，国内银行卡不行，所以直接选PayPal，如果没有帐号可立马注册，不要用银联卡，使用visa卡亲测可以。
5. 接下来直接购买云服务器，请选择CentOS系统。

##服务器配置

1. 服务器可以直接用ssh root@公网IP登录到控制台，也可以web界面登录。
2. 登录以后开始安装shadowsocks，直接执行下面指令。

```java
yum install python-pip
pip install shadowsocks
```

3. 安装完成后开始配置shadowsocks，在/root/ss目录下创建ssserver.json文件，内容如下：

```java
{
"server":"0.0.0.0",  //此处不要写阿里云的外网IP，写0.0.0.0
"server_port":7777,  //阿里云限制了端口，在阿里云安全配置界面里面把7777端口加上
"local_address":"127.0.0.1",
"local_port":1080,
"password":"*******",
"method":"aes-256-cfb"
}
```

4. 构建完成后执行下面命令启动和关闭，启动后可以在/var/log/shadowsocks.log下面查看日志。

```java
ssserver -c /root/ss/ssserver.json -d stop    //关闭
ssserver -c /root/ss/ssserver.json -d start  //启动
netstat -antlp | grep 7777  //确定服务是否启动了
```

##客户端配置

###MAC

1. [下载](https://github.com/shadowsocks/ShadowsocksX-NG/releases)安装，界面如下配置
 ![Alt text](/public/img/other/shadowsocks_2.png)
2.点击确定后，在MAC右上栏有图标，务必选中服务器，有沟就行。
![Alt text](/public/img/other/shadowsocks_3.png)
3. 一切OK，试试google吧。

#####IOS
1. 没有太纠结，直接花18快钱购买shadowrocket，直接使用。
