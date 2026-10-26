# V2bX 节点端报 websocket: bad handshake：反代与传输路径排查

用 WebSocket（WS）做传输的节点，明明端口通、证书没过期、面板也显示节点在线，客户端一连却握手失败，节点端日志里反复刷 `websocket: bad handshake`。这种报错很容易让人误以为是密钥错了或者节点挂了，其实问题往往不在节点核心本身，而在握手要走的那条路：反向代理没把 Upgrade 头透传过去、路径对不上、或者中间某一层把 WS 当成普通 HTTP 请求处理了。排查要按握手走过的路径一层一层剥开，不要一上来就重装节点端、换端口、换协议——路径没修好，换什么都还是握手失败。

## 先定性：确认报错确实来自握手层，而不是密钥或网络

`bad handshake` 的意思是握手请求发出去了，但对方返回的不是一次合法的 WebSocket 握手响应。先确认三件事：报错是不是持续出现、是不是只影响 WS 传输的节点、是不是从某次改动反代配置或换了 CDN 之后开始的。

```bash
# 看最近的报错是不是集中在 websocket 握手上，而不是认证或连接超时
journalctl -u v2bx -n 200 --no-pager | grep -iE 'websocket|bad handshake|upgrade'

# 确认服务本身在跑，排除服务没起来导致的连锁报错
systemctl status v2bx --no-pager
```

如果日志里同时有大量连接超时、连接被拒绝，那要先排除端口没放行、IP 被封这类网络问题，可以先按离线排查的思路走一遍：[V2bX 对接失败：面板显示节点离线怎么办](guides/v2bx-offline-troubleshoot.md)。如果服务正常、端口能通、报错清一色是 `bad handshake`，而且只有 WS 传输的节点出问题、直连 TCP 的节点正常，那就可以锁定在握手路径上，按本篇往下查。

还有一种情况要先排除：报错时间点和某次改动对得上——刚改过 Nginx 配置、刚上了 Cloudflare 橙云、刚换了反代路径。改动前正常、改动后握手失败，问题几乎一定在改动的那一层，先把那一层的配置翻出来看，不要去动节点核心配置。

## 第一步：画出握手要走的路径，再逐层检查

WS 握手本质上是一次特殊的 HTTP 请求：客户端先发一个带 `Upgrade: websocket` 请求头的 HTTP 请求，服务端同意升级后，这条连接才变成 WebSocket 连接。路径上任何一层——Nginx、Caddy、Cloudflare、面板前面的反代——只要有一层没把 Upgrade 头和 Connection 头原样透传，或者改写了请求路径，握手就会失败。

先把自己的路径画清楚，常见组合有三种：

1. 客户端直连节点端 WS 端口，中间没有反代；
2. 客户端 → Nginx/Caddy 反代 → 节点端 WS 端口；
3. 客户端 → Cloudflare → Nginx/Caddy → 节点端 WS 端口。

直连都失败的，先查节点端配置里的传输路径和监听地址；经过反代才失败的，重点查反代层；上了 Cloudflare 才失败的，重点查 Cloudflare 那一层。层数越多，越要一层一层隔离测试，不要同时改两层。

```bash
# 先确认节点端 WS 监听是否正常，端口是不是真的在监听
ss -tlnp | grep -E 'V2bX|xray|sing-box'
curl -v http://127.0.0.1:PORT/PATH 2>&1 | head -20
```

上面命令里的端口和路径换成你自己的节点端配置。本地 curl 能拿到响应，说明节点端本身没问题，问题在它前面的反代层；本地也握手失败，才回头查节点端配置。

## 第二步：Nginx 反代——Upgrade 头是最常见的漏网项

Nginx 默认不会透传 WebSocket 握手头，这是 WS 节点握手失败的第一大原因。反代配置里必须显式加上 Upgrade 相关头，缺一个都不行：

```nginx
location /ws-path {
    proxy_pass http://127.0.0.1:PORT;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

几个关键点，每个都解释一句为什么：

- `proxy_http_version 1.1`：WebSocket 握手需要 HTTP/1.1，默认的 HTTP/1.0 不支持 Upgrade 机制，不写这一行，Upgrade 头传过去也没用。
- `proxy_set_header Upgrade $http_upgrade`：把客户端发来的 Upgrade 头原样传给后端，Nginx 默认会丢掉这个头，后端收不到升级请求，自然握手失败。
- `proxy_set_header Connection "upgrade"`：告诉后端这条连接要升级成 WebSocket，没有这一行，后端会把请求当成普通 HTTP 请求处理，返回的就不是握手响应。
- `proxy_pass` 后面的路径：location 里的路径和节点端配置的 WS 路径必须一致，一边是 `/ws-path`、另一边是 `/ray`，握手请求根本到不了正确的处理函数。

改完配置不要只 reload 就以为好了，先验证语法、再 reload、再看日志：

```bash
nginx -t
systemctl reload nginx
journalctl -u nginx -n 50 --no-pager | grep -iE 'error|warn'
```

改完如果还是握手失败，用 curl 模拟一次握手请求，看返回状态码：返回 101 说明升级成功，返回 400、404、426 这类状态码，说明请求到了后端但被拒绝了，重点看路径和头；直接连接失败或超时，说明请求根本没到后端，重点看监听端口和防火墙。

## 第三步：路径和 Host 对不上——握手请求走错了门

路径和 Host 是第二大原因，而且报错常常不直观。节点端配置的 WS 路径、反代 location 的路径、面板上下发的节点信息里的路径，三处必须完全一致，差一个斜杠都不行。

```bash
# 对照三处路径：节点端配置、反代配置、面板上的节点传输设置
grep -A 5 -i 'websocket|wsSettings|path' /etc/V2bX/config.json
grep -A 3 'location' /etc/nginx/sites-enabled/*.conf 2>/dev/null
```

常见错法有三种：反代 location 写了 `/ws` 但节点端配的是 `/ws-path`；反代把路径改写了，比如 `proxy_pass` 后面带了 URI 导致路径被替换；面板上下发的路径还是旧的，节点端已经改了新的。改路径时三处一起改，改完立刻用订阅链接重新拉取一次，确认面板下发的确实是新路径。

Host 也一样：如果节点端或反代做了 Host 校验，握手请求里的 Host 和证书域名、反代 server_name 对不上，握手也会被拒绝。用了多域名、多 server_name 的配置，要确认握手请求实际命中的是配了 WS 转发的那个 server 块，而不是默认 server 块。默认 server 块常常没有 Upgrade 配置，请求落到它手里，握手必然失败。

## 第四步：Cloudflare 在中间——橙云、WebSocket 开关与端口

经过 Cloudflare 橙云的 WS 节点，还要多查一层。Cloudflare 默认支持 WebSocket，但有几个坑：免费套餐的 WebSocket 连接有并发和时长限制、回源端口必须在 Cloudflare 支持的端口范围内、回源链路上的任何一层（包括 Cloudflare 到源站的连接）同样需要 Upgrade 透传。

排查顺序是：先把域名切回灰云（DNS Only）直连源站测试，灰云下握手成功、橙云下失败，问题就在 Cloudflare 层；灰云下也失败，问题还在源站反代层，不要去动 Cloudflare 设置。橙云与灰云对节点的影响，本仓库另有专门一篇可以对照。

```bash
# 灰云直连测试：绕过 Cloudflare，直接对源站 IP 或源站域名做握手验证
curl -v http://源站IP:PORT/PATH -H "Connection: Upgrade" -H "Upgrade: websocket" 2>&1 | head -20
```

如果确认是 Cloudflare 层的问题，检查三项：域名确实是橙云且 WebSocket 没有被规则关掉；源站端口在 Cloudflare 允许的 HTTP/HTTPS 端口范围内；回源是 HTTPS 时源站证书有效，证书问题会导致 Cloudflare 回源失败，表现上也可能是握手失败。不要为了绕过 Cloudflare 问题而把源站 IP 直接写进面板下发——源站 IP 一旦暴露，换回橙云也藏不住了。

## 第五步：验证——握手成功、订阅能拉、流量能走

修完不要只看日志不刷报错就收工，握手成功只是第一步，按下面四项逐项验证：

```bash
# 1. 节点端日志里不再持续刷 bad handshake
journalctl -u v2bx -n 100 --no-pager | grep -iE 'websocket|bad handshake'

# 2. 用 curl 模拟握手，确认能拿到 101 Switching Protocols
curl -i -N -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==" \
  https://panel.example.com/ws-path 2>&1 | head -10

# 3. 重新拉取订阅，确认该节点出现在订阅里
curl -s https://panel.example.com/api/v1/client/subscribe?token=YOUR_TOKEN | head -c 500

# 4. 等一个流量上报周期，确认面板里该节点的流量在正常增长
```

上面命令里的域名、路径、token 都是占位符，换成你自己的。注意 curl 模拟握手只能验证反代层和握手层通不通，不能验证完整协议流程——最终还是要以真实客户端能连上、流量能走为准。握手层通了但客户端还是连不上，就要回到密钥、UUID、传输安全设置（TLS 与否）这些协议层配置上查，而不是继续在反代层打转。

## 常见坑

- **Nginx 漏了 Upgrade 头**：只配了 proxy_pass 没配 Upgrade 和 Connection，握手请求到后端就变成普通 HTTP 请求。
- **proxy_http_version 没写 1.1**：Upgrade 头传过去了，但 HTTP/1.0 不支持升级，照样失败。
- **三处路径不一致**：节点端、反代、面板下发各写各的路径，差一个斜杠握手就走错门。
- **请求落到默认 server 块**：多域名配置下握手请求命中了没配 WS 转发的默认 server，Upgrade 头被丢掉。
- **橙云灰云来回切着测**：DNS 缓存没过期，测到的还是旧解析结果，误以为改动没生效——切解析后等 TTL 过期再测。
- **只看日志不模拟握手**：日志不刷报错不等于握手成功，curl 模拟一次 101 才是硬证据。
- **同时改反代又改节点配置**：修好了也不知道是哪一层起的作用，出问题也分不清责任在谁。
- **源站 IP 写进面板下发**：为绕过 Cloudflare 问题暴露源站 IP，之后再藏也藏不住。

## 不想自己搭？

反代、握手头、路径、CDN 回源——搭机场的排障从来不是改一行配置就完事，每一层都可能藏一个让握手失败的小细节，半夜对着日志翻 Upgrade 头更是常事。如果你只是想稳定用起来，不想把时间花在对路径、查反代、逐层隔离上，可以直接用现成的订阅：漫画机场标准套餐 ¥10/月起、家宽套餐 ¥30/月起，100GB/月流量、支持 3 台设备，订阅开通即用，省去自己运维面板和节点的时间。详情见官网：https://vps.manhuajichang.fun
