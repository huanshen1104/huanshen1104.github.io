---
title: 使用acme.sh申请免费的HTTPS证书
date: 2020-8-15 15:48:36
categories: [网站建设]
tags: [HTTPS]
cover: false
---

最近把自己的网站从HTTP升级为HTTPS，在这里把过程及踩过的坑记录一下。

[`acme.sh`](https://github.com/Neilpang/acme.sh)是一个纯粹用Shell语言编写的脚本，本质上是一个实现了ACME协议的客户端，它足够简单，功能强大且易于使用，可以实现自动颁发、续订和安装证书，目前支持ZeroSSL、BuyPass、Let`s Encrypt等多种不同证书，本文使用的是ZeroSSL证书。

<!--more-->

### 安装acme.sh

```
curl https://get.acme.sh | sh
```
该安装脚本做了几件事：

- 把 acme.sh 安装到了 home 目录下：~/.acme.sh/
- 创建了一个 bash 的 alias, 方便使用: alias acme.sh=~/.acme.sh/acme.sh
- 创建了cronjob，每天 0:00点自动检测所有的证书，如果快过期了，则会自动更新证书。

可能会因为网络问题无法自动安装，则需要手动更改安装脚本内容：

```
# 下载并保存脚本
curl https://get.acme.sh -o acme_install.sh

# 更改脚本内域名，使用代理加速
sed -i 's/raw/ghproxy.com\/https:\/\/raw/g' acme_install.sh

# 安装
chmod +x acme_install.sh
./acme_install.sh
```
或者直接手动安装：
```
git clone https://github.com/acmesh-official/acme.sh.git
cd ./acme.sh
./acme.sh --install
```

### 注册 ZeroSSL 账号
申请证书之前建议先去ZeroSSL官网注册账号，ZeroSSL官网地址：https://zerossl.com

也可以直接使用以下命令来注册，如果已经在官网注册，该命令会自动关联账户，邮箱改为你自己的ZeroSSL邮箱，一定不要随便填。
```
acme.sh  --register-account  -m myemail@example.com --server zerossl
```


### 将 acme.sh 的注册服务器改为 ZeroSSL

如果你的`acme.sh` 是2.x版本，默认使用Let`s Encrypt作为服务提供商，我们可以通过以下命令，将其更换为ZeroSSL：

```
acme.sh --set-default-ca --server zerossl
```

### 配置DNS API
本文使用 DNS 验证的方法来验证域名，`acme.sh` 可以通过 DNS 提供商的 API 自动设置验证记录，具体用法详见文档：https://github.com/acmesh-official/acme.sh/wiki/dnsapi
常用的 CloudFlare 、 DNSPod 、 CloudXNS 、阿里云 等DNS服务都支持。

以阿里云为例，登录后台找到账号的秘钥(建议创建子账号来专门管理此功能)：
![](AccessKey.png)
如果你使用的是子账号，一定要给子账号开通以下权限：
![](阿里云dns子账号授权.png)
接下来执行下面命令即可：

```
export Ali_Key="****************************"
export Ali_Secret="**************************" 
```

### 签发证书
以zmtupian.com为例，该域名使用阿里云解析，在上面的步骤中已经配置好阿里云DNS API
```
acme.sh --dns dns_ali --issue -d zmtupian.com -d *.zmtupian.com
```
其中：
- --dns 指定 DNS 服务商，dns_ali 代表阿里云，还有 dns_cf 代表 CloudFlare，更多的字段见 https://github.com/acmesh-official/acme.sh/tree/master/dnsapi

  如果不使用 API 自动添加验证，则不用添加后续参数，如：acme.sh --issue --dns -d example.com ....
- -d \*.example.com 表示签发泛域名证书


执行命令之后看输出的内容可以确定证书所在位置，这时就签发成功了。

### 安装证书

```
acme.sh --install-cert -d zmtupian.com \
        --key-file       /var/www/ssl/zmtupian.com.key  \
        --fullchain-file /var/www/ssl/zmtupian.com.cert \
        --reloadcmd     "service nginx force-reload"
```
其中：
- --key-file 和 -fullchain-file 后接想要安装到的目录及证书名
- --reloadcmd 后接服务的重启命令，程序自动续期证书后会运行该命令使其生效，一定不能漏

默认情况下，证书将每60天更新一次（可配置），证书更新后，服务将通过reloadcmd设置的命令自动重新加载。

### 启用https服务
修改服务器的配置文件启用HTTPS，以nginx为例：

```
server {
        listen 443 ssl;
        server_name  www.zmtupian.com zmtupian.com;

        ssl_certificate      /var/www/ssl/zmtupian.com.cert;
        ssl_certificate_key  /var/www/ssl/zmtupian.com.key;
        ...
    }

server {
    listen       80;
    server_name  www.zmtupian.com zmtupian.com;

    return 301 https://$host$request_uri;
}

```
配置好后，检查是否有错误，然后重启服务器即可：

```
nginx -t
service nginx force-reload
```

最后，记得去服务器管理后台打开服务器443端口。


大功告成！




