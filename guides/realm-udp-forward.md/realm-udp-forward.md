# realm 转发 UDP 的配置要点

很多节点用 realm 做中转时只测了 TCP：网页能打开、测速能跑，就以为转发通了，结果一上需要 UDP 的业务就出问题——游戏语音断断续续、QUIC 打不开、基于 UDP 的协议握手失败。原因往往不在协议本身，而在中转链路上的 UDP 没打通。realm 本身是同时处理 TCP 和 UDP 的转发工具，问题多半出在某一跳只放行了 TCP、端口没对齐，或者 UDP 会话在中途被 NAT 换了源地址。本篇讲 realm 转发 UDP 时真正要抠的几个点：单跳怎么写、多跳怎么串、以及 UDP 不通时按什么顺序排查。

## 先说清一个前提：realm 不需要单独开 UDP 开关

新手最容易走的弯路，是到处找 realm 的“开启 UDP”选项。realm 的 endpoint 配置里，`listen` 和 `remote` 写的是同一个端口，TCP 和 UDP 走同一个端口转发，不需要、也不应该再为 UDP 单开一个端口。配置写对之后，TCP 和 UDP 都会从这个端口进出。如果你发现 TCP 通、UDP 不通，第一反应不应该是改 realm 配置，而应该是查防火墙、安全组和链路上的每一跳——十有八九是某一层只放行了 TCP。

> 为什么 TCP 通不能证明 UDP 通？TCP 和 UDP 在防火墙、安全组、运营商 NAT 上是两套独立的放行规则。机内 `ss` 看到端口在监听，也只说明进程绑了端口，不等于 UDP 包真的能从公网进来。TCP 通、UDP 不通，是中转排障里最典型的“半通”状态，必须用 UDP 报文实测，不能用 TCP 结果推断。

## 单跳转发：最简配置怎么写

假设中转机是 `203.0.113.10`（占位 IP，下同），落地机是 `198.51.100.20`，要转发的端口是 `20000`，realm 2.x 的 TOML 配置可以这样写：

```toml
# /etc/realm/config.toml
[[endpoints]]
listen = "0.0.0.0:20000"
remote = "198.51.100.20:20000"
```

> 为什么 `listen` 要写 `0.0.0.0` 而不是中转机的公网 IP？写 `0.0.0.0` 表示监听本机所有网卡地址，公网 IP、内网 IP 都能进来；写死某个 IP 后，一旦机器换 IP 或走内网地址访问，转发就直接拒绝连接。中转配置里除非有明确的多地址隔离需求，否则一律用 `0.0.0.0`。

配成 systemd 服务跑起来，并设置开机自启：

```bash
# 把 realm 二进制放到固定路径（按你实际下载的版本调整）
mkdir -p /etc/realm
cp realm /usr/local/bin/realm
chmod +x /usr/local/bin/realm

# systemd 单元：/etc/systemd/system/realm.service
cat > /etc/systemd/system/realm.service <<'EOF'
[Unit]
Description=realm TCP/UDP relay
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/realm -c /etc/realm/config.toml
Restart=on-failure
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now realm
systemctl status realm --no-pager
```

> 为什么 `LimitNOFILE` 要调大？UDP 会话是按源地址加源端口来区分的，一个用户背后可能有大量并发 UDP 会话，默认的文件描述符上限很容易在高并发时先撞满，表现就是新连接偶发失败。转发机这项调大，成本很低，收益很实在。

启动之后先确认端口在 TCP 和 UDP 上都处于监听状态：

```bash
# -t 看 TCP，-u 看 UDP，-l 只看监听态，-n 不解析域名，-p 显示进程
ss -tlnp | grep 20000
ss -ulnp | grep 20000
```

两条命令都要有输出才算正常。只看到 TCP 那条、UDP 那条是空的，说明 realm 没起来或者端口被别的进程占了，先查 `systemctl status realm` 和日志，不要急着改配置。

## 多跳串联：每一跳的 UDP 都要对齐

单跳通了，多跳不一定通。常见的组网是“入口中转 → 二级中转 → 落地”，realm 串联时每一跳都要同时满足三个条件：本跳的 `listen` 端口放行了 UDP、本跳的 `remote` 指向下一跳的 `listen` 端口、下一跳的防火墙也放行了 UDP。任何一跳缺了 UDP，整条链的 UDP 就断在那一跳，而 TCP 可能全程都是通的，排查时很容易误判成“落地协议有问题”。

```toml
# 入口中转机（203.0.113.10）上的配置
[[endpoints]]
listen = "0.0.0.0:20000"
remote = "203.0.113.11:20000"

# 二级中转机（203.0.113.11）上的配置
[[endpoints]]
listen = "0.0.0.0:20000"
remote = "198.51.100.20:20000"
```

> 为什么多跳要强调“端口对齐”？UDP 没有 TCP 那样的三次握手来帮你纠错，端口写错一跳，包就被悄悄丢掉，不会返回一个明确的错误给你。串联配置写完后，要把每台机器的 `listen`、`remote`、防火墙规则三张表并排对一遍：入口的 remote 等于二级的 listen，二级的 remote 等于落地的实际服务端口，中间不能有一跳偷换端口还不改防火墙。

还有一个多跳特有的坑：中间某一跳如果还套了别的转发或 NAT，源地址可能在中途被改写。UDP 会话是靠源地址加源端口识别的，源地址一变，realm 和落地服务会把它当成一个全新的会话，旧会话的回包就再也对不上。表现是连接一开始能通，跑一会儿就断，或者换网络就断。遇到这种“时通时断”，先查链路上有没有多余的 NAT 层，能直连就直连，不要为了“多一层保险”无意义地叠转发。

## 防火墙与安全组：两层都要放 UDP

节点端 API 防护那篇讲过“机内防火墙和云厂商安全组是两层”，UDP 转发同样适用，而且更容易漏——很多人配安全组时只勾了 TCP，机内防火墙也只放了 TCP，realm 本身一点问题都没有，包在入口就被丢了。

```bash
# 机内防火墙：TCP 和 UDP 必须各放行一次
# ufw 的例子
ufw allow 20000/tcp
ufw allow 20000/udp
ufw status | grep 20000

# firewalld 的例子
firewall-cmd --permanent --add-port=20000/tcp
firewall-cmd --permanent --add-port=20000/udp
firewall-cmd --reload
firewall-cmd --list-ports
```

> 为什么命令要敲两遍？`ufw allow 20000` 不带协议时默认只放 TCP，很多人以为一条命令就全放了，结果 UDP 一直被拦。凡是放行端口的地方，都要亲眼确认 TCP 和 UDP 各有一条规则，云厂商安全组后台也一样：入方向规则里应当同时出现 TCP 20000 和 UDP 20000，缺任何一条都不算配完。

核对顺序是：先看机内防火墙，再看云厂商安全组，最后看运营商或机房层面有没有 UDP 限速或屏蔽。有些廉价 VPS 会对 UDP 做限速，大流量 UDP 一上来就丢包，表现是小流量测试全通、真实业务一跑就卡，这种情况换一台不限制 UDP 的中转机比调配置管用。

## 验证：不要用 TCP 工具测 UDP

realm 起来了、防火墙放行了，最后一步是真正发一个 UDP 包走完整条链路验证。不要用 `curl` 或 `telnet` 测 UDP，它们走的是 TCP，测通了也证明不了 UDP。

```bash
# 在落地机上起一个 UDP 回显监听（测试用，测完就关掉）
# 监听 20000 的 UDP，收到什么就原样打出来
socat -v UDP-LISTEN:20000,fork -

# 在入口中转机之外、能访问中转机的另一台机器上发 UDP 测试包
echo "realm-udp-test" | socat - UDP:203.0.113.10:20000

# 回到落地机：socat 应当原样输出 realm-udp-test
# 如果落地机没收到，按“入口防火墙 → 入口 realm → 二级防火墙 → 二级 realm → 落地防火墙”的顺序逐跳查
```

> 为什么测试包要从中转机之外的机器发？从中转机本机发包走的是回环或内网路径，绕过了公网防火墙和安全组，测通了也不能证明公网用户能进来。UDP 验证必须站在真实用户的视角，从链路外发包，落地机收到才算数。

如果测试包在某一跳丢了，定位方法是逐跳收包：在每一跳的机器上用抓包确认包有没有到达该跳、realm 有没有转发出去，不要凭感觉猜。包在哪一跳消失，问题就在那一跳的防火墙、端口或配置里，realm 的日志和系统日志要配合着看。

## 常见坑汇总

- **只放行 TCP，没放行 UDP**：最常见，症状是 TCP 业务全通、UDP 业务全断。机内防火墙和安全组各查一遍，两层都要有 UDP 规则。
- **多跳端口没对齐**：上一跳的 remote 和下一跳的 listen 不一致，UDP 包被静默丢弃，不报错。把所有跳的配置并排对一遍。
- **用 TCP 测试结论代替 UDP 验证**：`curl` 通了不等于 UDP 通了，必须用 `socat` 这类 UDP 工具从链路外实测。
- **中途多余的 NAT 改写源地址**：UDP 会话靠源地址识别，源一变会话就断，表现是时通时断。能直连就直连，少叠一层是一层。
- **UDP 限速的廉价中转机**：小流量全通、大流量丢包，调配置没用，换不限制 UDP 的机器才是正解。
- **测试用的 UDP 监听忘关**：上面 `socat` 监听只是临时测试，验证完要立刻关掉，长期敞着一个回显口等于给扫描器留了个现成的靶子。
- **只测了单跳就以为多跳没问题**：入口通不代表全链通，每加一跳就要重新从链路外测一次，跳数越多越不能省这一步。

realm 转发 UDP 本身并不复杂，复杂的是链路：防火墙两层、串联多跳、NAT 改写、UDP 限速，任何一处漏了都会表现为“UDP 不通”，而 realm 的配置往往是无辜的。按“配置对齐、两层放行、链路外实测、逐跳定位”的顺序走，大多数 UDP 转发问题都能在半小时内锁定到某一跳；怕的是只测 TCP 就宣布完工，等真实业务一上才发现 UDP 从来就没通过。

内链参考：防火墙两层放行的完整思路，见本仓库的[节点端 API 防护：只允许面板 IP 访问](node-api-firewall.md)。

## 不想自己搭？

按上面这套走下来，从写多跳配置、配 systemd 服务、两层防火墙各放一遍 UDP、再到从链路外发 UDP 包逐跳验证，顺利的话也要花上一两个小时，中间任何一跳端口或防火墙填错都得回来重查。如果你只是想有个现成可用的服务、并不想研究这些，[漫画机场](https://vps.manhuajichang.fun) 提供了现成的订阅：标准套餐 ¥10/月起（含 100GB/月流量、支持 3 台设备），想要更好线路的还有家宽套餐 ¥30/月起，开箱即用，上面这些折腾一步都不用做。
