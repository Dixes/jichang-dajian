# VLESS + WS + TLS + CDN 传统方案

在 Reality 流行之前，VLESS + WebSocket + TLS + CDN 是机场的标配：流量伪装成正常的 HTTPS 网站访问，再套一层 CDN 隐藏源站 IP。这套方案至今仍有价值——CDN 覆盖地区的用户走 CDN 入口延迟更低，而且配置成熟、资料多。本文讲这套传统方案的完整搭建。

## 准备工作

需要：一个域名（如 cdn.example.com）、Cloudflare 账号（免费版即可）、一台落地服务器。把域名接入 Cloudflare，DNS 记录开启橙色云朵（代理模式），这是 CDN 生效的前提。

在落地机上安装 Xray 或 V2bX（参考 V2bX 对接篇），确认面板通信正常。

## 服务端配置

核心思路：Nginx/Caddy 监听 443 提供 TLS，`/ray` 路径反代给 V2bX 的 WebSocket 入站，其他路径返回一个伪装网站。

Nginx 配置示例：

```nginx
server {
    listen 443 ssl;
    server_name cdn.example.com;
    ssl_certificate /etc/letsencrypt/live/cdn.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cdn.example.com/privkey.pem;

    # 伪装网站
    location / {
        root /var/www/html;
        index index.html;
    }
    # WS 分流
    location /ray {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

`location /ray` 是关键：只有这个路径的流量是代理流量，其他都是普通网站访问。路径名用随机字符串更好，别用默认的 `/ray`，降低被扫描识别的概率。

V2bX 侧新增 VLESS + WS 入站，监听 127.0.0.1:10000，ws 路径填 `/ray`，与 Nginx 对应。

## CDN 配置要点

Cloudflare 侧：

```
SSL/TLS 加密模式：完全（严格）
# 严格模式要求源站证书有效，用 acme.sh 申请的 Let's Encrypt 证书即可
```

「完全（严格）」保证 CDN 到源站也是加密的，否则中间段明文传输。开启「始终使用 HTTPS」。注意 Cloudflare 免费版只代理常用端口，443 没问题。

在 Xboard 新增节点时：地址填 cdn.example.com，端口 443，传输协议 ws，ws 路径填你设置的随机路径，TLS 开启，SNI 填域名。

## 验证

```bash
# 1. 伪装站正常访问
curl -s -o /dev/null -w "%{http_code}" https://cdn.example.com/
# 应返回 200

# 2. WS 路径握手测试（应返回 400 或 101 相关，而非 404）
curl -s -o /dev/null -w "%{http_code}" https://cdn.example.com/你的随机路径

# 3. 确认源站 IP 隐藏
nslookup cdn.example.com
# 应返回 Cloudflare 的 IP，而非你的服务器 IP
```

再用测试账号实际连接，确认能正常使用。源站 IP 一旦泄露（历史 DNS 记录、邮件头等），CDN 隐藏就失效，定期检查。

## 常见坑

一是 Cloudflare SSL 模式选成「灵活」，CDN 到源站走明文，被中间识别。二是 WS 路径两端不一致，Nginx 转发出去了但 V2bX 没收到，检查路径拼写。三是伪装站太敷衍，一个空白页反而可疑，放个正常的博客模板。四是源站 IP 泄露后不换 IP，CDN 等于白套。

## 不想自己搭？

WS+TLS+CDN 的链路环节多：域名、CDN、证书、反代、路径分流，任何一环配错就全不通，排错要逐段抓包，耗时很长。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
