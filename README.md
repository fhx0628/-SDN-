# 如果升级了SDN网关，可以这样科学上网
背景

SDN是什么

为什么SDN会屏蔽科学上网

我们的策略是什么

操作流程

## 背景

笔者位于魔都在一次机缘巧合下换了电信光猫升级了200MBPS宽带，然后VPN就连不上了。显然问题就出在光猫上，笔者在网上查阅了一下，发现目前并没有现成的解决方案，看来只能自己动手丰衣足食了。

## SDN是什么

SDN的全称是software defined network，简单来说原来的网络提速依靠的是物理手段，而SDN则是通过软件的方式给网络提速。它的方法很简单，就是在网关之上添加一个一个数据集散中心，网关的所有数据会先进入数据集散中心，由数据集散中心根据数据的目标节点分配一个快速通道，数据经由快速通道可以直达目标节点。

![image](https://upload-images.jianshu.io/upload_images/17233224-3ec63a64a4d6e2cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 为什么SDN会屏蔽科学上网

如果您看了上文会发现SDN明明就是一个好东西为啥就屏蔽了VPN呢？哼哼，答案就是电信的SDN方案不是纯粹的SDN方案，它是电信重新定义过的SDN方案，它在一个地方偷换了概念。请见下图：

![image](https://upload-images.jianshu.io/upload_images/17233224-da576532fce01b67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

显然在电信的SDN方案中多了一层东西，这层东西就是过滤器。当过滤器发现你的目标节点是指向海外，过滤器就会截断你的数据阻止其抵达目标节点。

那么问题又来了增加这一层过滤好处是什么呢？请看下图

![image](https://upload-images.jianshu.io/upload_images/17233224-9f27f7bbc321a700.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图截自电信SDN网关配套的网管管理app，目前用户可以在app中**免费**解锁对于海外网站的过滤但是一天只有1小时，显然这个过滤器可以为海外加速业务埋点。就我个人而言，如果电信推出海外加速包月业务且价格合理的话，我还是愿意买单的。原因有如下两点：1\. 电信提供的海外加速是直达专线，理论上也可以达到200MB，对于科学上网而言，你所需要面对的网速瓶颈仅仅在VPN的带宽上；2\. 如果你通过其他的方式规避电信过滤器的过滤，那你也将会面对不小的开销，那我不如选择电信。

## 我们的策略是什么

策略其实很简单，既然电信过滤了海外的访问请求，那么我们就顺其自然，索性不在SDN网关上直接访问海外，而是另外租一台国内的云主机，让云主机直接访问海外，而我们通过直连云主机间接访问海外。

![image](https://upload-images.jianshu.io/upload_images/17233224-f528e12eb0215e06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 操作流程

#### 第一步 启动vpn（购买海外云主机，启动shadowsocks server）。

在购买海外云主机后，ssh登录云主机，在控制台执行以下命令安装 shadowsocks：
在安装shadowsocks之前，先要安装pip：
```
$curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
$python get-pip.py
```
在控制台执行以下命令安装 shadowsocks：
```
$pip install --upgrade pip
$pip install shadowsocks
```
安装完成后，需要创建shadowsocks的配置文件/etc/shadowsocks.json，编辑内容如下：
```bash
{
    "server":"0.0.0.0",
    "server_port":1025,
    "local_address":"127.0.0.1",
    "local_port":1090,
    "password":"xxxx",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open":false
}
```
其中需要记住的有三点，在使用shadowsocks client时会用到。
- server_port：服务监听端口
- password：vpn密码（自定义）
- method：加密方式
最后启动shadowsocks server
```
$ssserver -c /etc/shadowsocks.json -d start
```
#### 第二步 购买国内云主机，部署端口转发应用rinted
源码安装rinted：
```
$cd /usr/local/src && wget https://boutell.com/rinetd/http/rinetd.tar.gz && tar xf rinetd.tar.gz && cd rinetd
$sed -i 's/65536/65535/g' rinetd.c
$mkdir -p /usr/man/man8 && make && make install
```
接着编辑配置文件:/etc/rinetd.conf：
```
#格式{localhost} {entry port} {remote} {out port}
0.0.0.0 1025 10.10.112.141 1025
```
- localhost: 填0.0.0.0即可
- entry port：填写云主机向外开放的入口端口，用于给shadowsocks client连接
- remote：填写你的海外云主机ip
- out port: 填写shadowsocks server中配置的server_port端口号

最后启动rinted
```
/usr/sbin/rinetd -c /etc/rinetd.conf
```
至此VPN+踏板机的组合方案已经完成了。
#### 最后一步 使用shadowsockets client连接VPN
作者使用的是MacOS，采用的shadowsocks client是ShadowsocksX-NG，所以在下图就展示一下ShadowsocksX-NG的配置。
![ShadowsocksX-NG](https://upload-images.jianshu.io/upload_images/17233224-b6b8a979dd34782f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)
 - 国内云主机IP：填写部署rinted服务的主机ip地址
 - rinted暴露的端口：填写rinted的entry port
 - shadowsocks server密码：填写shadowsocks server的vpn密码
