# 中转机端口转发：realm / iptables 实操

中转机的价值在于「借路」：用户先连到线路优质的中转机，再由中转机把流量转发到真正的落地节点。这样即使落地机线路一般，用户体验也由中转机决定；落地机 IP 被封时，只换落地不换入口，用户无感。本文用 realm（用户态转发，配置简单）和 iptables（内核态转发，性能更高）两种方式实现 TCP/UDP 端口转发。

## 准备工作

准备两台服务器：中转机（线路好、带宽足，例如 relay.example.com）和落地机（已跑好 V2bX 节点，监听 443 端口，IP 为 203.0.113.10 示例）。确认中转机到落地机的直连延迟和丢包率，先测速：

```bash
ping -c 20 203.0.113.10
mtr -rw 203.0.113.10
```

中转质量取决于中转→落地这一段，丢包超过 2% 就换中转机房，否则转发只是把卡顿搬了个地方。开放中转机的转发端口，例如 443 的 TCP 和 UDP。

## 方案一：realm 转发（推荐）

realm 是 Rust 写的转发工具，单文件、配置直观，适合大多数场景：

```bash
wget -O /usr/local/bin/realm https://github.com/zhboner/realm/releases/latest/download/realm-x86_64-unknown-linux-gnu.tar.gz
# 解压后
chmod +x /usr/local/bin/realm
```

写配置文件 `/etc/realm/config.toml`：

```toml
[log]
level = "warn"

[[endpoints]]
listen = "0.0.0.0:443"
remote = "203.0.113.10:443"
```

`listen` 是中转机对外监听的地址端口，用户连这里；`remote` 是落地机的真实地址端口。转发是透明的，落地机看到的源 IP 是中转机 IP，这对大多数协议没影响。创建 systemd 服务：

```bash
cat > /etc/systemd/system/realm.service <<'EOF'
[Unit]
Description=realm port forward
After=network.target
[Service]
ExecStart=/usr/local/bin/realm -c /etc/realm/config.toml
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl enable --now realm
systemctl status realm
```

用 systemd 托管，转发进程崩了自动拉起，服务器重启自动恢复。

## 方案二：iptables 转发

对性能要求极高、转发量大的场景用内核态转发，延迟更低：

```bash
# 开启转发
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

# TCP 转发
iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination 203.0.113.10:443
iptables -t nat -A POSTROUTING -p tcp -d 203.0.113.10 --dport 443 -j MASQUERADE

# UDP 转发
iptables -t nat -A PREROUTING -p udp --dport 443 -j DNAT --to-destination 203.0.113.10:443
iptables -t nat -A POSTROUTING -p udp -d 203.0.113.10 --dport 443 -j MASQUERADE

# 保存规则（Debian/Ubuntu）
apt install -y iptables-persistent
netfilter-persistent save
```

`DNAT` 把目标地址改写成落地机，`MASQUERADE` 把源地址改写成中转机，保证回包能原路返回。不保存规则的话重启就丢，这是最常见的「重启后转发失效」原因。

## 面板侧配合与验证

在 Xboard 后台新增节点时，地址填中转机的域名或 IP（relay.example.com），端口填转发端口，其他协议参数与落地机保持一致。用户感知不到落地机的存在。

验证：

```bash
# 中转机上看端口监听
ss -tlnp | grep 443
# 从本地测试连通性
curl -vk --connect-timeout 5 https://relay.example.com:443
```

再用测试账号实际连接，对比直连落地和走中转的延迟、速度。中转正常时延迟应接近「本地→中转→落地」两段之和，速度不低于直连的 80%，否则说明中转机房选得不好。

## 常见坑

一是落地机防火墙只放行了中转机段之外的 IP，转发过来的包被拦，落地机要放行中转机 IP。二是只转了 TCP 没转 UDP，Hysteria2 用户全部不通，UDP 规则别漏。三是 realm 配置文件路径写错导致服务起不来，看 `journalctl -u realm` 定位。四是中转机带宽跑满，表现为晚高峰集体卡顿，加带宽或加中转机分流。

## 不想自己搭？

中转架构涉及两台以上服务器的配合，任何一段出问题都要跨机器排查，运维成本是单节点的数倍。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
