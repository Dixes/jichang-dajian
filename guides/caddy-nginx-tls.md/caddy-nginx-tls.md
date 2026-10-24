# 反代与证书：Caddy/Nginx + 自动续期

机场的面板、订阅下发和支付回调都依赖 HTTPS：没有证书，浏览器会拦截登录页，支付回调会被拒，CF 的「完全（严格）」模式也过不了。本文给出两套反代方案——省心的 Caddy 和可控的 Nginx + Certbot——并把证书自动续期一次配好，避免 90 天后半夜证书过期全站宕机。

## 准备工作

准备一台已装好 Xboard 面板的服务器，域名已解析到本机，例如 panel.example.com；确认 80 和 443 端口未被占用。证书签发依赖 80 端口的 HTTP 验证或 DNS 验证，签发期间 80 端口必须可从公网访问。

先确认域名解析正确：

```bash
dig +short panel.example.com
curl -I http://panel.example.com
```

解析出的 IP 必须是本机公网 IP（或 CF 代理 IP，签发时记得对应处理），否则证书机构验证域名所有权会失败。

## 方案一：Caddy（推荐新手）

Caddy 会自动申请并续期 Let's Encrypt 证书，配置只有几行：

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install -y caddy
```

写 Caddyfile：

```
panel.example.com {
    reverse_proxy 127.0.0.1:7001
}
```

`reverse_proxy` 把请求转发给本机的 Xboard 容器端口，7001 换成你 compose 里映射的实际端口。Caddy 启动后自动完成证书申请和续期，你不需要再管 certbot。重载配置：

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile
systemctl reload caddy
journalctl -u caddy -n 30 --no-pager
```

日志里出现「certificate obtained successfully」即签发成功。

## 方案二：Nginx + Certbot

习惯 Nginx 的可以用这套，控制粒度更细：

```bash
apt install -y nginx certbot python3-certbot-nginx
```

先写一个 80 端口的 server 块用于验证：

```nginx
server {
    listen 80;
    server_name panel.example.com;
    location / {
        proxy_pass http://127.0.0.1:7001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

`X-Forwarded-Proto` 必须传递，否则面板可能判断不出用户用的是 HTTPS，导致登录态或回调地址生成错误。然后申请证书：

```bash
certbot --nginx -d panel.example.com
nginx -t && systemctl reload nginx
```

certbot 会自动改写 server 块加上 443 和证书路径，并写入续期定时任务。用 `systemctl list-timers | grep certbot` 确认续期任务存在。

## 验证与续期检查

验证三件事：浏览器打开 https://panel.example.com 没有证书警告；`curl -I https://panel.example.com` 返回 200；证书有效期正常：

```bash
echo | openssl s_client -connect panel.example.com:443 -servername panel.example.com 2>/dev/null | openssl x509 -noout -dates
certbot renew --dry-run
```

第一条看证书起止日期，第二条模拟续期流程，成功说明 90 天后能自动续上。建议再设一个日历提醒，在到期前一周人工检查一次。

## 常见坑

一是 80 端口被 CF 代理挡住导致 HTTP 验证失败：签发时先把域名切到 DNS only（灰色云朵），签完再开代理；或改用 DNS 验证。二是 Nginx 和 Caddy 同时装、同时抢 80/443，先停掉一个。三是证书签发成功但浏览器仍报不安全，多半是 CF 的 SSL 模式还是「灵活」，按域名篇改成「完全（严格）」。四是忘记开 443 防火墙，表现为本机 curl 正常、外网打不开。

## 不想自己搭？

反代和证书看着是几条命令，但签发失败、续期失效、重定向循环这些坑每个都够折腾一晚上。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
