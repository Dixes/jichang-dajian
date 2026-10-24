# Hysteria2 节点搭建与调优

Hysteria2 是基于 QUIC（UDP） 的代理协议，专为弱网和高丢包环境优化，在晚高峰和跨国链路抖动时往往比 TCP 系协议更稳。它也是很多机场的「备用高速通道」：TCP 被限速或拥塞时，UDP 通道常常还能跑满。本文在 V2bX 节点机上部署 Hysteria2 入站，并完成带宽调优。

## 准备工作

准备一台已对接 Xboard 的 V2bX 节点服务器；确认 UDP 端口可用，例如 8443；准备一个域名（可选），例如 hy2.example.com。Hysteria2 走 UDP，所以机房是否限 UDP 很关键：先用 iperf3 或简单的 UDP 连通性测试确认目标机房 UDP 可达，否则调优无从谈起。放行端口：

```bash
ufw allow 8443/udp
```

只开需要的 UDP 端口，多余的 UDP 端口保持关闭，避免被扫描利用。

## 生成证书与配置

Hysteria2 需要 TLS 证书，可以用自签证书（面板会把指纹下发给客户端），也可以用域名申请的正式证书。自签最快：

```bash
mkdir -p /etc/hysteria && cd /etc/hysteria
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout server.key -out server.crt -subj "/CN=hy2.example.com"
```

有效期给 10 年，省去频繁更换的麻烦，因为客户端校验的是证书指纹而非有效期链，过期前记得更换即可。然后在 V2bX 管理菜单新增 Hysteria2 入站，示例配置：

```yaml
listen: :8443
tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key
auth:
  type: password
  password: "换成强随机密码"
bandwidth:
  up: 500 mbps
  down: 500 mbps
```

`password` 是用户认证口令，强度直接决定节点是否会被扫爆；`bandwidth.up/down` 是 Hysteria2 拥塞控制的基准带宽，填服务器的真实出口带宽，填大了会导致拥塞误判、填小了跑不满速度，这是调优的第一步。

## 系统级调优

QUIC 对 UDP 缓冲区敏感，调大内核参数能明显提升吞吐：

```bash
cat >> /etc/sysctl.conf <<'EOF'
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.udp_mem = 102400 873800 16777216
EOF
sysctl -p
```

`rmem_max/wmem_max` 放大 UDP 收发缓冲区，避免高速传输时丢包；`udp_mem` 控制 UDP 协议栈的内存水位。三者配合，千兆口才能真正跑出千兆。如果服务器只有 100Mbps 小水管，调优效果有限，优先升级带宽。

## 面板侧配置与验证

在 Xboard 后台新增 Hysteria2 节点，端口填 8443，协议选 Hysteria2，把证书指纹（或 insecure 跳过校验的选项说明）和密码同步写入。保存后重启 V2bX：

```bash
systemctl restart v2bx
journalctl -u v2bx -n 30 --no-pager
```

验证分三步：面板显示节点在线；用测试账号连接后做一次持续 5 分钟的下载测速，观察速度曲线是否平稳；故意在弱网（比如手机热点）下对比 TCP 节点的卡顿情况，Hysteria2 应该明显更稳。

## 常见坑

一是机房限 UDP，表现为能握手但速度极慢或频繁断流，换机房是唯一解。二是带宽参数填错，填了 1gbps 实际只有 200Mbps，拥塞控制会不断误判，反而更卡。三是证书路径写错导致启动失败，看日志第一行报错即可定位。四是防火墙只开了 TCP 没开 UDP，这是新手最常见的疏漏。

## 不想自己搭？

Hysteria2 的调优涉及内核参数、带宽标定和长期观察，单个节点折腾下来往往要花几个晚上。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
