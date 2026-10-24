# V2bX 对接失败：面板显示节点离线怎么办

节点机上的 V2bX 明明跑起来了，日志里也没有报错，可面板后台的节点列表里它就是灰的、显示「离线」。这种情况十有八九不是协议服务端本身的问题，而是面板和节点机之间的"握手"没通——面板定期向 V2bX 的 API 端口拉取状态，拉不到就判离线。顺着"面板 → 网络 → V2bX 配置"这条链路查一遍，基本都能定位。

> 前置：本文假设面板侧（Xboard/SSPanel 类）已经建好节点、拿到了 API 地址、API 密钥（Token）和节点 ID。V2bX 的配置文件通常在 `/etc/V2bX/config.json`。

## 第一步：确认 V2bX 进程和 API 端口真的在监听

先在节点机上确认 V2bX 在运行，并且它的 API 端口（默认 `Config` 里 `ApiConfig` 指定的端口，常见 6060）处于监听状态：

```bash
# V2bX 进程在不在
ps aux | grep -v grep | grep V2bX

# API 端口有没有监听（把 6060 换成你 config.json 里配的端口）
ss -tlnp | grep 6060
```

如果进程没在跑，先看 V2bX 自己的日志找崩溃原因：

```bash
journalctl -u V2bX --no-pager -n 100
# 或直接看 V2bX 输出日志
cat /etc/V2bX/V2bX.log
```

进程在但端口没监听，大概率是 `config.json` 格式错误导致 V2bX 启动时回退或部分失败。用 `python3 -m json.tool /etc/V2bX/config.json` 校验一下 JSON 合法性——多一个逗号、少一个引号，V2bX 都会静默失败。

## 排查 1：API 地址、密钥、节点 ID 配错（最高发）

面板判离线的第一个嫌疑人永远是这三个参数对不上。打开节点机的 `config.json`，找到 `ApiConfig` 段：

```json
{
  "ApiConfig": {
    "ApiHost": "https://panel.example.com",
    "ApiKey": "面板里生成的API密钥",
    "NodeID": 3,
    "NodeType": "vless",
    "Timeout": 30,
    "EnableVless": true,
    "EnableXTLS": false,
    "SpeedLimit": 0,
    "DeviceLimit": 0
  }
}
```

逐项核对：

1. **ApiHost**：必须是面板的**公网可访问地址**，带 `https://` 前缀、结尾不带斜杠。如果面板用了 Cloudflare 橙云或反代，填浏览器能打开的那个域名。填内网 IP 或 `127.0.0.1` 的节点机，离面板十万八千里，根本连不上。
2. **ApiKey**：面板后台「节点」里复制的 API 密钥，复制时别带空格。换过密钥后记得同步更新节点机配置。
3. **NodeID**：面板里这个节点的数字 ID，不是节点名称。面板改过节点、删过重建后 ID 会变，旧配置里的 ID 就失效了。

改完配置重启 V2bX 生效：`systemctl restart V2bX`。环境变量方式注入的配置同理，改完必须重启。

## 排查 2：网络不通——面板够不着节点机的 API 端口

面板是**主动**去连节点机的 API 端口拉状态的，所以节点机的 API 端口必须对面板所在机器可达。在面板机（或任意一台能代表面板网络位置的机器）上测试：

```bash
# 把 node.example.com 换成节点机 IP/域名，6060 换成实际 API 端口
curl -v --connect-timeout 10 http://node.example.com:6060/
```

连不上就顺着这三层查：

```bash
# 1. 节点机防火墙放行了没
firewall-cmd --list-ports          # firewalld
ufw status                         # ufw

# 放行 API 端口（示例 6060）
firewall-cmd --permanent --add-port=6060/tcp && firewall-cmd --reload
ufw allow 6060/tcp

# 2. 云厂商安全组：阿里云/腾讯云等要在控制台给 6060/tcp 放行，
#    来源建议只填面板机的公网 IP，而不是 0.0.0.0/0

# 3. V2bX 只监听了 127.0.0.1？
ss -tlnp | grep 6060
# 如果显示 127.0.0.1:6060，说明只绑了本地回环，面板从外部连不进来。
# 检查 config.json 里有没有把监听地址写死，V2bX 默认监听 0.0.0.0，一般不用动
```

安全建议：API 端口不要对全网开放。正确做法是在节点机防火墙和云安全组里**只放行面板机的 IP**（见仓库里的《节点端 API 防护：只允许面板 IP 访问》），既保证对接，又不把管理端口暴露给扫描器。

## 排查 3：HTTPS 与证书问题

如果 `ApiHost` 填的是 `https://` 域名，而面板的证书有问题（过期、自签、域名不匹配），V2bX 拉取 API 会直接 TLS 握手失败，面板侧表现为节点离线。先确认面板域名本身正常：

```bash
# 在节点机上测面板域名的证书
curl -vI https://panel.example.com 2>&1 | grep -E "subject|issuer|expire"
```

证书过期了就去续（参考《Let's Encrypt 证书自动续期失败排查》）。应急测试时可以临时把 `ApiHost` 改成 `http://` 看离线是否消失——如果消失了，问题就锁定在证书链上，修好证书再切回 `https`。注意这只是诊断手段，生产环境 API 通信务必走 HTTPS。

## 排查 4：节点机时间偏差太大

V2bX 和面板的 API 交互带时间戳校验，节点机系统时间偏差超过几分钟，部分面板会拒绝其上报，节点显示离线。这在刚开好的 VPS 上很常见（NTP 没同步）：

```bash
# 看当前时间和偏差
timedatectl status

# 没同步就打开 NTP
timedatectl set-ntp true

# 强制同步一次
chronyc -a makestep   # 有 chrony 的系统
# 或
ntpdate -u pool.ntp.org
```

时间对上后重启 V2bX，再回面板看状态。

## 排查 5：V2bX 与面板版本/协议类型不匹配

- **NodeType 写错**：面板里建的是 VLESS 节点，`config.json` 里却写了 `"NodeType": "shadowsocks"`，面板拉到的数据对不上协议类型，会判异常/离线。两边必须一致。
- **V2bX 版本太旧**：面板升级后 API 字段变了，老版本 V2bX 解析失败。去 V2bX 的 GitHub releases 看一眼有没有新版，升级命令一般是重新跑一键脚本或替换二进制后 `systemctl restart V2bX`。
- **多节点共用一个 NodeID**：两台节点机配了同一个 NodeID，面板收到互相覆盖的上报，表现为节点一会儿在线一会儿离线。每个节点机必须用自己独立的 NodeID。

## 验证：对接成功的三个信号

改完配置重启 V2bX 后，按顺序确认：

```bash
# 1. V2bX 日志里出现成功拉取面板数据的记录，没有 API 报错
journalctl -u V2bX --no-pager -n 50 | grep -i -E "api|error|success"

# 2. 面板机能通节点 API 端口
curl -s --connect-timeout 10 http://node.example.com:6060/ -o /dev/null -w "%{http_code}\n"
# 只要不是连接超时/拒绝，就算网络层通了

# 3. 面板后台节点状态变绿，且「在线人数/流量」开始有数字变化
```

面板状态有缓存，刚修好可能要等 1～2 个上报周期（通常 60 秒左右）才变绿，别急着反复改配置。等两分钟再看。

## 常见坑

1. **面板和节点机互相"觉得对方有问题"**：记住方向——是面板主动连节点机的 API 端口。面板机出方向、节点机入方向，两边的防火墙都要看，只查一边会漏。
2. **改了配置没重启 V2bX**：`config.json` 是启动时一次性加载的，改完必须 `systemctl restart V2bX`，`reload` 不一定生效。
3. **把用户连接端口和 API 端口搞混**：用户连的是 443/2096 这类协议端口，面板连的是 6060 这类 API 端口。用户能连上不代表面板能连上，这是两条完全独立的链路。
4. **面板机 IP 变了**：面板迁移或换了出口 IP 后，节点机防火墙里只放行旧 IP 的规则就把面板挡在外面了，节点集体变灰。换 IP 后第一时间更新所有节点机的放行规则。

对接问题的排查就一句话：先确认 V2bX 本地正常，再确认面板"够得着"节点，最后核对三要素（地址、密钥、节点 ID）。九成离线都是这三步里的问题。

## 不想自己搭？

说实话，上面这一套对着日志调参数的活儿，第一次搭机场的人很容易卡住半天——面板和节点各在一台机器上，出问题还得两边来回查，对新手不太友好。如果你只是想有个稳定可用的订阅、不想把时间花在排查"节点离线"上，可以直接用现成的：[漫画机场](https://vps.manhuajichang.fun) 标准套餐 ¥10/月起、家宽套餐 ¥30/月起，每月 100GB 流量、支持 3 台设备，注册就有订阅链接可用，把折腾服务器的时间省下来。
