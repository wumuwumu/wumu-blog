---
title: let-us-encrypt证书
date: 2018-12-05 20:59:59
tags:
- web
---

# 基本知识

为了实现通配符证书，Let’s Encrypt 对 ACME 协议的实现进行了升级，只有 v2 协议才能支持通配符证书。

1. 客户在申请 Let’s Encrypt 证书的时候，需要校验域名的所有权，证明操作者有权利为该域名申请证书，目前支持三种验证方式：

- dns-01：给域名添加一个 DNS TXT 记录。

- http-01：在域名对应的 Web 服务器下放置一个 HTTP well-known URL 资源文件。

- tls-sni-01：在域名对应的 Web 服务器下放置一个 HTTPS well-known URL 资源文件。

  而申请通配符证书，只能使用 dns-01 的方式

2. ACME v2 和 v1 协议是互相不兼容的，为了使用 v2 版本，客户端需要创建另外一个账户（代表客户端操作者），以 Certbot 客户端为例，大家可以查看：
3. Enumerable Orders 和限制

# 安装

```bash
wget https://dl.eff.org/certbot-auto
chmod a+x ./certbot-auto
```

# 申请

```bash
./certbot-auto certonly  -d *.newyingyong.cn --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory
```

- certonly，表示安装模式，Certbot 有安装模式和验证模式两种类型的插件。
- –manual 表示手动安装插件，Certbot 有很多插件，不同的插件都可以申请证书，用户可以根据需要自行选择
- -d 为那些主机申请证书，如果是通配符，输入 *.newyingyong.cn（可以替换为你自己的域名）
- -preferred-challenges dns，使用 DNS 方式校验域名所有权
- –server，Let’s Encrypt ACME v2 版本使用的服务器不同于 v1 版本，需要显示指定。

# 添加记录

根据命令行提示，填写相关的内容，注意在添加记录的时候，要等到记录生效才确定。

```
-------------------------------------------------------------------------------
Please deploy a DNS TXT record under the name
_acme-challenge.newyingyong.cn with the following value:
2_8KBE_jXH8nYZ2unEViIbW52LhIqxkg6i9mcwsRvhQ
Before continuing, verify the record is deployed.
-------------------------------------------------------------------------------
Press Enter to Continue
Waiting for verification...
Cleaning up challenges
```

```
## 检测记录生效
$ dig  -t txt  _acme-challenge.newyingyong.cn @8.8.8.8
```

# 更新

查看当前服务器所配置的证书

```bash
certbot-auto certificates
```

1. 使用申请的普通证书，使用`certbot-auto renew`

2. 使用通配符证书。

   1. 添加DNS记录

   ```bash
   git clone https://github.com/ywdblog/certbot-letencrypt-wildcardcertificates-alydns-au.git
   ```

   ```bash
   ./certbot-auto renew --cert-name simplehttps.com  --manual-auth-hook /脚本目录/au.sh 
   ```

3. 自动更新

```
1 1 */1 * * root certbot-auto renew --manual --preferred-challenges dns  --manual-auth-hook /脚本目录/sslupdate.sh 
```

# 参考

> https://www.jianshu.com/p/c5c9d071e395
>
> https://www.jianshu.com/p/074e147b68b0
>
> [certbot工具](https://github.com/ywdblog/certbot-letencrypt-wildcardcertificates-alydns-au)