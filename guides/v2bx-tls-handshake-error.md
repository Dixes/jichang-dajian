# V2bX 节点端报 TLS handshake error：证书、SNI 与时钟三路排查

节点端日志里刷 `TLS handshake error`，面板却可能还显示节点在线，客户端一连就失败——这是 TLS 传输节点最让人头疼的一类报错。握手是在协议真正跑起来之前完成的，握手一失败，后面什么 UUID、流量统计、订阅配置都还没轮到，所以报错信息本身往往很笼统：只告诉你握手错了，不告诉你哪一环错了。排查这条报错不要上来就换证书、换端口、重启整个节点，TLS 握手失败的原因其实就集中在三条路上：证书本身有问题、SNI 对不上、系统时间差太多。按这三条路一条一条排除，比漫无目的地重装快得多。

## 先定性：确认报错是 TLS 握手层，而不是网络不通

先把日志里的完整报错看全，不要只看 `TLS handshake error` 这一行，后面跟的细节才是分流依据：`certificate has expired`、`unknown authority`、`bad certificate`、`first record does not look like a TLS handshake`、超时类报错，各自指向完全不同的方向。

```bash
# 看最近的 TLS 握手报错，把后面的具体原因一起打出来
journalctl -u v2bx -n 300 --no-pager | grep -iE 'TLS handshake|certificate|tls' | tail -40

# 确认服务在跑、端口在监听，先排除服务没起来的连锁报错
systemctl status v2bx --no-pager
ss -tlnp | grep -E 'V2bX|xray|sing-box'
```

再做一个关键区分：如果报错是 `first record does not look like a TLS handshake`，说明对端发来的根本不是 TLS 握手包——大概率是有人（或某个健康检查、端口扫描、反代）用纯 HTTP 去撞了一个 TLS 端口，或者客户端的 TLS 开关和节点端不一致，一边开了 TLS、一边没开。这种情况不要按证书问题去查，先核对客户端和节点端的 TLS 开关是否一致。如果报错里明确带证书字样，就按下面第一路查；如果带超时、连接重置，就先看端口放行和中间层，证书先往后放。

另外要确认影响范围：是所有 TLS 节点都握手失败，还是只有一个域名、一个节点出问题。全部都失败，优先查节点端共用的配置或时间；只有一个失败，优先查这个节点自己的证书和域名解析，不要动共用配置。

## 第一步：证书这条路——过期、自签、链不全、域名不匹配

证书问题是 TLS 握手失败的第一大原因，而且每一种的表现都不一样，要逐项核对。

```bash
# 看证书的有效期、签发者和域名覆盖范围
openssl x509 -in /path/to/cert.pem -noout -dates -issuer -subject -ext subjectAltName 2>/dev/null

# 从公网视角验证一次：握手能不能完成、链全不全、域名对不对
openssl s_client -connect node.example.com:443 -servername node.example.com </dev/null 2>/dev/null | head -30
```

逐项对：

- **过期了**：`notAfter` 的时间已经过去，客户端直接拒绝握手。证书续期失败的排查本仓库另有专门一篇，先确认续期任务还在不在、续期日志报了什么，不要只手动续一次就收工，自动续期没修好，下次还会过期。
- **自签证书**：自签证书在浏览器和客户端默认不被信任，节点端用了自签证书、客户端又没导入这张自签根证书，握手必然失败。生产节点不要用自签证书凑合，除非你能确保每个客户端都正确导入了根证书。
- **链不全**：只配了服务器证书、没配中间证书，某些客户端能靠缓存补全链、某些客户端直接拒绝，表现就是“有的能连、有的不能连”。`openssl s_client` 的输出里 `Verify return code` 不是 0（ok）时，先看链有没有问题。
- **域名不匹配**：证书是给 `a.example.com` 签的，节点实际跑在 `b.example.com` 上，或者证书的 SAN 里根本没有这个域名，握手时域名校验直接失败。换域名后证书也要跟着换，面板上下发的域名、证书域名、反代 server_name 三处要对得上。

证书路径也要核对：节点端配置里写的证书文件路径是不是真实存在、权限是不是节点端进程能读。路径写错或权限不足，服务启动时可能不报错、握手时才失败，这种情况看配置文件比看证书本身更重要。

```bash
# 对照节点端配置里的证书路径和实际文件
grep -A 5 -i 'cert\|tls' /etc/V2bX/config.json
ls -l /path/to/cert.pem /path/to/key.pem
```

证书文件改过之后一定要重启节点端服务，只 reload 面板不管用；重启后立刻用 `openssl s_client` 再验证一次，确认公网视角看到的已经是新证书，而不是缓存里的旧证书。

## 第二步：SNI 这条路——客户端发来的名字和服务端期望的名字对不上

SNI（Server Name Indication）是 TLS 握手第一阶段客户端告诉服务端“我要访问哪个域名”的字段。服务端靠 SNI 决定拿哪张证书、走哪个入站；SNI 对不上，服务端要么拿错证书、要么直接拒绝，握手就失败。这一路的问题常常被误当成证书问题——证书本身没问题，只是因为 SNI 错了，服务端拿出来的根本不是你想的那张证书。

先核对三处：客户端配置里的 SNI（或 serverName）、节点端入站配置里的域名、证书覆盖的域名，三处必须是一致的。常见错法有三种：客户端留空 SNI、靠 IP 直连，服务端按默认证书响应，域名校验失败；多域名共用一个端口，SNI 指向的域名没有对应的入站配置，请求落到默认入站；面板下发的节点信息里的 SNI 还是旧域名，节点端已经换了新域名。

```bash
# 用指定的 SNI 做一次握手测试，看服务端返回的是哪张证书
openssl s_client -connect node.example.com:443 -servername node.example.com </dev/null 2>/dev/null | grep -E 'subject|issuer|Verify return'

# 换一个 SNI 再测一次，对比返回证书有没有变化
openssl s_client -connect node.example.com:443 -servername other.example.com </dev/null 2>/dev/null | grep -E 'subject|issuer'
```

两次返回的证书如果一样，说明服务端根本没按 SNI 区分证书——多域名配置下这就是问题所在；如果第二次直接握手失败，说明该 SNI 没有对应的入站或证书。SNI 相关的配置改动后，同样要重启节点端并重新拉取订阅确认面板下发的是新 SNI，不要只改一头。

还有一个容易忽略的点：经过 Cloudflare 或其他 CDN 回源时，SNI 是回源域名还是源站域名，取决于回源配置。橙云回源时源站看到的 SNI 可能和客户端原始 SNI 不一致，源站证书要覆盖回源域名，或者回源配置要透传原始 SNI。这一层没对齐，灰云直连正常、一上橙云就握手失败，就是典型的 SNI 问题。

## 第三步：时钟这条路——节点端时间差太多，证书直接被判无效

证书有效期是按时间判定的，节点端系统时间如果差了几天甚至几个月，证书在客户端看来就是“还没生效”或“已经过期”，握手直接失败。更隐蔽的是：时间差得不多时，有的客户端宽容、有的客户端严格，表现就是部分客户端能连、部分不能连，很容易误判成证书或网络问题。

```bash
# 看节点端当前时间和与标准时间的偏差
date
timedatectl status
chronyc tracking 2>/dev/null || ntpq -p 2>/dev/null
```

如果 `System clock synchronized` 是 no，或者偏差已经按分钟甚至小时计，先校时再测握手，不要先动证书。面板机和节点端的时间也要一起对：面板签发、节点端校验、证书有效期判定都依赖时间，两边时间差太多，节点认证也会连带出问题。面板机与节点端校时的完整排查，本仓库另有一篇可以对照：[面板机与节点端系统时间不同步导致节点认证失败：NTP 校时排查](guides/panel-node-time-sync-ntp.md)。

```bash
# Debian/Ubuntu 上启用并立即同步一次
timedatectl set-ntp true
systemctl enable --now chrony 2>/dev/null || systemctl enable --now systemd-timesyncd
chronyc makestep 2>/dev/null; date
```

校时后重启节点端服务，再看日志里 TLS 握手报错是不是停了。虚拟机从快照恢复、宿主机时间漂移、容器时间没同步宿主机，都是时钟问题的常见来源——这类机器上要把 NTP 服务设为开机自启，只手动同步一次，下次重启还会漂回去。

## 第四步：中间层——反代和 CDN 也可能在握手阶段插一手

前面三路都排除了还握手失败，就要看握手要经过的中间层。反代配置里如果把 TLS 终结在反代层（反代持有证书、后端走明文），那客户端握手的对手其实是反代，不是节点端，证书、SNI、时间都要按反代那一层来查；如果反代只是 TCP 透传（stream），那反代不参与握手，问题还在节点端本身。先搞清楚自己是哪一种架构，再决定查哪一层，不要两层一起改。

```bash
# 确认反代是 TLS 终结还是 TCP 透传
grep -A 10 'ssl_certificate\|proxy_pass\|stream' /etc/nginx/sites-enabled/*.conf /etc/nginx/nginx.conf 2>/dev/null | head -40
```

反代层 TLS 终结时，常见坑是反代证书过期了、反代的 SNI 路由配错了、反代到后端的连接用了旧的 TLS 版本或密码套件。CDN 在中间时也一样：回源证书过期、回源协议版本不匹配，表现上都是握手失败，但根因在回源链路。隔离方法是绕过中间层直连节点端测试：直连握手成功、经中间层失败，问题在中间层；直连也失败，问题还在节点端。

## 验证：握手成功只是第一步

修完按下面四项逐项验证，不要只看日志不刷报错就收工：

```bash
# 1. 节点端日志里不再持续刷 TLS handshake error
journalctl -u v2bx -n 100 --no-pager | grep -i 'TLS handshake'

# 2. 公网视角握手成功，证书链校验通过
openssl s_client -connect node.example.com:443 -servername node.example.com </dev/null 2>/dev/null | grep -E 'Verify return|subject'

# 3. 重新拉取订阅，确认该节点出现在订阅里
curl -s https://panel.example.com/api/v1/client/subscribe?token=YOUR_TOKEN | head -c 500

# 4. 等一个流量上报周期，确认面板里该节点的流量在正常增长
```

上面命令里的域名、路径、token 都是占位符，换成你自己的。注意 `openssl s_client` 只能验证握手层和证书层，最终还是要以真实客户端能连上、流量能走为准。握手通了但客户端还是连不上，就要回到 UUID、传输设置、订阅配置这些协议层上查，而不是继续在 TLS 层打转。

## 常见坑

- **只看 `TLS handshake error` 这一行**：后面的具体原因（过期、自签、域名不匹配、非 TLS 包）才是分流依据，笼统报错没法定位。
- **证书续了但服务没重启**：公网视角看到的还是旧证书，误以为续期没生效。
- **链不全**：有的客户端能连、有的不能连，误判成网络问题。
- **SNI 留空或写错**：服务端拿错证书或落到默认入站，证书本身没问题也被判失败。
- **只改节点端不改面板下发**：节点端换了新域名、新 SNI，面板下发的还是旧的，客户端永远握手失败。
- **时间漂移没设开机自启**：手动同步一次就收工，重启后时间又漂回去，报错复发。
- **反代和节点端两层一起改**：修好了也不知道是哪一层起的作用，出问题也分不清责任在谁。
- **TLS 开关不一致**：一边开了 TLS、一边没开，握手包根本不是 TLS 包，按证书查是查不出原因的。

## 不想自己搭？

证书、SNI、时钟、反代——搭机场的排障从来不是改一行配置就完事，每一层都可能藏一个让握手失败的小细节，对着日志逐项核对更是常事。如果你只是想稳定用起来，不想把时间花在查证书、核对 SNI、校对时间上，可以直接用现成的订阅：漫画机场标准套餐 ¥10/月起、家宽套餐 ¥30/月起，100GB/月流量、支持 3 台设备，订阅开通即用，省去自己运维面板和节点的时间。详情见官网：https://vps.manhuajichang.fun
