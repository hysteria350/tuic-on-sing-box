# 使用 sing-box 搭建 Tuic 节点

sing-box 是一个开源的、跨平台的通用代理与网络转发工具，常用于科学上网、流量分流、透明代理、流媒体解锁和多协议代理服务部署。它可以看作是新一代的高性能代理核心之一，设计目标是高性能、模块化、可扩展。它可以：

-   作为代理客户端使用

-   作为代理服务器部署

-   做透明代理网关

-   实现分流策略

-   解锁流媒体

-   构建家庭或服务器网络出口

本文主要介绍如何使用 sing-box 作为代理服务器，并部署一个 Tuic 节点。

## 1. 准备工作

-   海外 VPS 一台安装好主流的操作系统 （Almalinux 8+、Ubuntu 16+、Debian 8+）

-   域名一个

-   DNS 指向 VPS IP

-   80 端口开放，并且未被占用 （用于申请证书）

本教程使用的是一台[Hostvds VPS](https://hostvds.com/?affiliate_uuid=4dc97e32-b519-48b2-8e7e-c869cdf7948e) 和 一个[NameSilo DNS](https://www.namesilo.com/pricing?rid=5d23d49ag)

-   配置：2 vCore 4 GB RAM (3.99$)
-   域名： .top 域名 (1.88$)
-   操作系统： Almalinux 9

域名解析：

```bash

配置 A 记录:

tuic.example.top -> 1.2.3.4(服务器的 ip 地址)

```

## 2. 安装 sing-box

这里以 Redhat 系列发行版为例，更多平台安装方式，请参考 sing-box 官方文档，或之前讲 sing-box 作为代理客户端教程中 sing-box 安装部分。

对于 Redhat 系列发行版（Centos, Fedora, AlmaLinux, Rocky Linux 等）

```bash

# 向 DNF 添加 sing-box 官方软件仓库
# 这样系统就可以从该仓库获取 sing-box 软件包
sudo dnf config-manager addrepo --add-repo=https://sing-box.app/sing-box.repo

# 从指定网址下载并导入 sing-box 官方的 GPG 公钥
# 用于验证软件包签名，防止安装被篡改的软件
sudo rpm --import https://sing-box.app/gpg.key

# 清除 DNF 的所有缓存数据（软件包缓存、仓库元数据等）
# 常用于仓库更新异常或软件列表不同步时强制重新下载缓存
sudo dnf clean all

# 更新本地软件仓库缓存
# 让系统读取刚添加的仓库中的软件列表
sudo dnf makecache

# 安装 sing-box
# -y 表示自动确认安装过程中的提示
sudo dnf install -y sing-box


# 确认版本, 确保有 with_quic
sing-box version

# 设置为开机自启动
sudo systemctl enable --now sing-box.service

# 查看服务状态
systemctl status sing-box.service

# 如果启动过程中出现任何错误，查看日志并进行修复
journalctl -u sing-box

```

> 如何出现以下错误，需要在sing-box.service service区块添加 Environment="ENABLE_DEPRECATED_SPECIAL_OUTBOUNDS=true" 或者将 配置文件中 /etc/sing-box/config.json
> 中 outbounds 中dns-out部分删除
>
>```json
> {
>  "type": "dns",
>  "tag": "dns-out"
> }
>
> ```
>
> Jan 30 23:36:48 server-0hrxrx.novalocal sing-box[3807]: ERROR[0000] legacy special outbounds is deprecated in sing-box 1.11.0 and will be removed in sing-box 1.
> Jan 30 23:36:48 server-0hrxrx.novalocal sing-box[3807]: FATAL[0000] to continuing using this feature, set environment variable ENABLE_DEPRECATED_SPECIAL_OUTBOUNDS=true

## 3. 准备 TLS 证书

Tuic 强制要求 TLS

安装 Certbot

```bash

sudo dnf install -y certbot

```

申请证书：

```bash

# 如果开启, 需放行80端口
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --reload

# 申请证书：这里使用webroot方式申请，电子邮件地址（用于紧急续签及安全通知）
sudo certbot certonly --standalone --domain tuic.example.top -m your_email_address --agree-tos

```

> 证书通常保存在 /etc/letsencrypt/live/tuic.example.top/ 目录下, 目录内包括：fullchain.pem （证书）、 privkey.pem（私钥） 、 cert.pem、 chain.pem，通常只需要：fullchain.pem 和 privkey.pem

## 4. 配置 Tuic 代理服务器

参考以下配置替换其中的域名、端口、uuid、密码，然后放入/etc/sing-box/config.json，一个简单的tuic节点就配置完成了。

> sing-box提供了uuid生成工具 `sing-box generate uuid`

```json


{
  "log": {
    "level": "info"
  },

  "inbounds": [
    {
        "type": "tuic",
        "tag": "tuic-in",

        "listen": "::",
        "listen_port": 8443,

        "users": [
            {
            "name": "user1",
            "uuid": "bafbd521-3302-4925-8708-43513cc6ded5",
            "password": "644c31d7-0cf7-4a15-8007-80edb65ef3a0"
            }
        ],
        "congestion_control": "bbr",
        "auth_timeout": "3s",
        "zero_rtt_handshake": false,
        "heartbeat": "10s",
        "tls": {
            "enabled": true,
            "alpn": [
              "h3"
            ],
            "certificate_path": "/etc/letsencrypt/live/tuic.example.top/fullchain.pem",
            "key_path": "/etc/letsencrypt/live/tuic.example.top/privkey.pem"
        }
    }
  ],

  "outbounds": [
    {
      "type": "direct"
    }
  ]
}

```

## 5. 防火墙开放 UDP 端口

```bash
firewall-cmd --permanent --add-port=8443/udp
firewall-cmd --reload
```

## 6. 检测配置

```bash

sing-box check -c /etc/sing-box/config.json

```

## 7. 重启 sing-box

```bash

systemctl restart sing-box

```

当日志中出现 `inbound/tuic[tuic-in]: udp server started at [::]:8443` 表示 sing-box 启动成功，Tuic 节点监听在 8443 端口

```bash

sing-box[6819]: INFO[0000] network: updated default interface eth0, index 2
sing-box[6819]: INFO[0000] inbound/tuic[tuic-in]: udp server started at [::]:8443
sing-box[6819]: INFO[0000] sing-box started (0.22s)

```

## 8. 测试

V2rayN 测试，将以下连接字符串中域名替换成真实域名，粘贴到 v2rayN 节点分组，然后测试节点真连接延迟：

连接字符串： `tuic://bafbd521-3302-4925-8708-43513cc6ded5:644c31d7-0cf7-4a15-8007-80edb65ef3a0@tuic.example.top:8443?sni=tuic.example.top&alpn=h3&congestion_control=bbr#tuic_node`

如果使用 clash 系列客户端, 可以将以下配置中的 hy2.example.top 替换成真实域名， 导入到客户端进行测试：

```yaml

mixed-port: 7890
allow-lan: true
bind-address: '*'
mode: rule
log-level: info
external-controller: '127.0.0.1:9090'
dns:
    enable: true
    ipv6: false
    default-nameserver: [223.5.5.5, 119.29.29.29]
    enhanced-mode: fake-ip
    fake-ip-range: 198.18.0.1/16
    use-hosts: true
    nameserver: ['https://doh.pub/dns-query', 'https://dns.alidns.com/dns-query']
    fallback: ['https://doh-pure.onedns.net/dns-query', 'https://ada.openbld.net/dns-query', 'https://223.5.5.5/dns-query', 'https://223.6.6.6/dns-query']
    fallback-filter: { geoip: true, ipcidr: [240.0.0.0/4, 0.0.0.0/32] }
proxies:
    - { name: tuic-on-sing-box, type: tuic, server: tuic.example.top, port: 8443, udp: true, uuid: 71a636a6-974d-4454-8653-4cc332a9cd3e, password: 71a636a6-974d-4454-8653-4cc332a9cd3e, skip-cert-verify: false, sni: tuic.example.top, alpn: [h3], congestion-controller: bbr, udp-relay-mode: native }

proxy-groups:
    - { name: tuic-group, type: select, proxies: ['tuic-on-sing-box'], url: 'http://www.gstatic.com/generate_204', interval: 86400 }
    - { name: 自动选择, type: url-test, proxies: ['tuic-on-sing-box'], url: 'http://www.gstatic.com/generate_204', interval: 86400 }
    - { name: 故障转移, type: fallback, proxies: ['tuic-on-sing-box'], url: 'http://www.gstatic.com/generate_204', interval: 7200 }
rules:
    - 'DOMAIN-SUFFIX,cn,DIRECT'
    - 'DOMAIN-KEYWORD,-cn,DIRECT'
    - 'GEOIP,CN,DIRECT'
    - 'MATCH,tuic-group'

```
