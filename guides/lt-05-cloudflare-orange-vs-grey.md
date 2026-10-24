# Cloudflare 橙云和灰云对节点有什么影响

在 Cloudflare 的 DNS 管理页面，每个 A/AAAA 记录后面都有一朵小云：点成橙色叫"已代理"（Proxied，俗称橙云），点成灰色叫"仅 DNS"（DNS only，俗称灰云）。这个小开关对机场搭建的影响比看起来大得多——面板站和节点域名应该用哪种云，直接决定节点能不能用、面板安不安全。下面按实操顺序讲清楚。

> 前置：本文假设你已经在 Cloudflare 托管了域名。如果节点离线的问题跟云状态无关，先看本仓库的 [V2bX 对接失败：面板显示节点离线怎么办](v2bx-offline-troubleshoot.md) 排查面板与节点端的连通性。

## 橙云和灰云的本质区别

一句话：**橙云是流量先经过 Cloudflare 再到你的服务器；灰云是 DNS 只做解析、流量直连你的服务器 IP。**

具体到差异：

- **橙云**：用户解析到的是 Cloudflare 边缘节点的 IP，你的源站 IP 被隐藏；但 Cloudflare 免费/Pro 套餐的反向代理**只转发 HTTP/HTTPS 流量**（端口 80/443 等标准 Web 端口），非 HTTP 协议的 TCP/UDP 流量过不去。
- **灰云**：用户直接拿到你的 A/AAAA 记录指向的真实服务器 IP，流量直达；你的源站 IP 完全暴露。

这个区别决定了机场里两类域名必须区别对待：**面板站域名**和**节点域名**。

## 节点域名：为什么一般用灰云

机场节点的协议（VLESS、Hysteria2、Trojan 等）走的是非标端口上的自有协议，不是普通网页流量。橙云只代理 HTTP/HTTPS，所以：

1. **橙云会直接掐断节点流量**：比如你的 Hysteria2 监听在 UDP 443，橙云下 Cloudflare 边缘节点根本不转发 UDP 给源站，客户端永远连不上，表现为"节点全挂、面板显示在线但没人能连上"；
2. **VLESS+Reality 走橙云是自杀行为**：Reality 的核心是伪装成与某个真实站点（dest）的 TLS 握手，SNI 指向的是你的域名。如果中间插了一个 Cloudflare 代理，TLS 握手在 CF 边缘就被终结了，Reality 握手直接失败——这是开橙云后 Reality 节点必死的根本原因；
3. **灰云流量直连，延迟也最低**：少了一跳代理，晚高峰不经过 CF 边缘的拥塞点。

> 为什么只有 HTTP/HTTPS 能过橙云？因为 Cloudflare 的反向代理本质是 Web 应用防火墙+CDN，它理解 HTTP 语义才能做缓存、WAF、Bot 管理。UDP 和任意 TCP 端口的流量它不理解，免费套餐就直接不转发。企业版有 Spectrum 可以代理任意 TCP/UDP，但那是按量付费的贵东西，机场用它不划算。

**节点域名的正确姿势**：A 记录指向落地机 IP，云点成灰色。需要隐藏源站 IP 时，正确做法是**中转机架构**（落地机只接受中转机 IP 连接），而不是指望橙云——橙云保护的是 Web 站，不是节点。

## 面板站域名：橙云的正确用法

面板站（Xboard 前端+后端）是标准的 Web 应用，这正是橙云的舒适区：

```nginx
# 面板站用橙云时，源站只需要正常监听 80/443 即可
# CF 边缘到源站的回源走 443，源站证书用 CF 的 Origin Certificate 或 Let's Encrypt 都行
server {
    listen 443 ssl;
    server_name panel.example.com;   # 你的面板域名（占位符）
    # ... 正常面板配置
}
```

橙云给面板站带来的好处：

- **隐藏面板源站 IP**：面板机 IP 不暴露，被扫描、被打的面小很多；
- **免费的 WAF 和限流**：面板后台登录页是最容易被撞库的地方，CF 的 Bot Fight Mode 和限流规则能挡掉大部分；
- **HTTPS 自动续**：CF 边缘证书自动管理，源站证书即使过期，用户侧也不报错（但回源仍建议保持有效，见下文坑）。

**回源模式的选择**：SSL/TLS 设置里选 **Full (strict)**，要求源站证书有效。这是安全和可用性的平衡点——选 Flexible（CF 到源站走明文 HTTP）等于自废武功，面板 API 传输的用户数据在公网裸奔。

## 特殊场景：节点想"套 CF"的合法姿势

有些场景确实想让节点流量也经过 Cloudflare（比如落地机 IP 质量差、想蹭 CF 的 Anycast 入口）。有两种合法做法：

**做法一：VLESS + WebSocket + TLS，走 443 橙云**

这是唯一能在免费橙云下跑通的节点方案，因为 WebSocket over TLS 本质是 HTTPS 流量，CF 认得：

```json
// 节点端（以 Xray 为例）inbound 片段，监听本地端口给 CF 回源
{
  "port": 10000,
  "protocol": "vless",
  "settings": { "clients": [{ "id": "你的UUID（占位符）" }] },
  "streamSettings": {
    "network": "ws",
    "security": "none",
    "wsSettings": { "path": "/你的路径（占位符）" }
  }
}
```

```nginx
# 源站 Nginx：CF 回源 443 → 反代到本地 10000
server {
    listen 443 ssl;
    server_name node.example.com;   # 节点域名（占位符），DNS 开橙云
    location /你的路径（占位符） {
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

代价要想清楚：流量经过 CF 边缘，多一跳延迟；CF 对免费套餐的非网页大流量有 ToS 风险（官方不鼓励把 CF 当代理用）；而且**所有用户共享 CF 边缘出口特征**，一个域名被重点关照会影响整批用户。所以这是"备选线路"方案，不是主力方案——主力节点还是灰云直连。

**做法二：Cloudflare Tunnel（cloudflared）**

节点不出公网 IP，cloudflared 从内网主动连出到 CF 边缘建隧道。适合源站 IP 极度不想暴露、或机器没有公网 IP 的场景。配置复杂度高一个量级，新手不建议首选。

> 为什么不推荐新手用 Tunnel 跑主力节点？因为多了一个常驻进程要守护（挂了整台节点失联）、排障链路变长（分不清是隧道挂了还是节点挂了），而且免费 Tunnel 的带宽和稳定性不如直连。先用灰云把直连跑稳，再考虑这些花活。

## 验证：改完云状态怎么确认生效

改完 DNS 的云状态，DNS 生效需要几分钟到几十分钟。验证步骤：

```bash
# 1. 看解析结果：橙云应返回 CF 的 IP 段（如 104.x.x.x / 172.67.x.x），灰云应返回你的源站 IP
dig +short node.example.com

# 2. 节点域名（灰云）直连测试：TCP 端口通不通
nc -zv node.example.com 443

# 3. 面板站（橙云）回源测试：确认 CF 能连到源站
curl -sI https://panel.example.com | head -5
```

```bash
# 4. 看 HTTP 响应头里有没有 CF 痕迹，确认橙云真的在代理
curl -sI https://panel.example.com | grep -i "cf-ray"
# 有 cf-ray 响应头 = 流量确实经过了 Cloudflare
```

第 4 步最容易被忽略：DNS 显示 CF 的 IP 不代表代理链路健康，`cf-ray` 头存在才说明请求真的走了 CF 边缘。

## 常见坑

1. **全站一键开橙云，节点跟着挂**：最常见的事故。`*.example.com` 的通配符记录开了橙云，节点子域名也被代理。修复：给节点子域名单独建 A 记录并点成灰云，通配符只给面板/官网用。
2. **Reality 节点开了橙云**：TLS 握手在 CF 边缘终结，Reality 必死。Reality 节点的域名必须灰云，无例外。
3. **橙云 + Full (strict) 但源站证书过期**：CF 回源时校验源站证书，过期就 526 报错，面板全站打不开。源站 Let's Encrypt 自动续期要配好（见本仓库证书续期排查篇）。
4. **灰云暴露源站 IP 后被扫**：灰云是节点域名的正确选择，但代价是 IP 暴露。配套做：节点端只监听必要端口、SSH 密钥登录+改端口、中转架构隐藏真实落地 IP。
5. **DNS 没生效就下结论**：改了云状态立刻测试，命中的还是旧解析。`dig` 确认解析结果变了再测，TTL 设短一点（如 300 秒）方便调试。
6. **用橙云跑 Hysteria2/任何 UDP 协议**：CF 免费代理不转发 UDP，配了等于没配，客户端超时。UDP 协议节点一律灰云。
7. **面板 API 域名和节点域名混用**：面板订阅下发接口的域名开了橙云没问题，但别把同一个域名又拿去做节点。职责分离：`panel.example.com`（橙云）管面板，`node-*.example.com`（灰云）管节点。

记住这个分工口诀：**面板站要藏，用橙云；节点要通，用灰云；想套 CF 走 WebSocket 443 做备选，不做主力。**

## 不想自己搭？

从规划面板和节点两套域名、配 DNS 云状态、调回源 TLS 模式，到 WAF 规则、证书续期、中转架构——自己搭一套带 Cloudflare 优化的机场，顺利的话要折腾两三天，DNS 和证书的坑一个接一个，还不算之后持续的运维。如果你只是想有个稳定可用的服务、并不想研究这些，[漫画机场](https://vps.manhuajichang.fun) 提供了现成的订阅：标准套餐 ¥10/月起（含 100GB/月流量、支持 3 台设备），想要更好线路的还有家宽套餐 ¥30/月起，开箱即用，上面这些折腾一步都不用做。
