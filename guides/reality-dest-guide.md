# Reality 的 dest 怎么选：借用站点与密钥生成

VLESS + Reality 是现在节点端最省事的组合：不用买域名、不用申请证书，服务端自己生成一对密钥就能把流量伪装成某个"借用站点"的 TLS 流量。但很多人第一次配 Reality 就卡在两个地方——`dest` 到底填什么、密钥怎么生成。填错 dest，轻则客户端连不上，重则伪装效果大打折扣。本文把这两件事一次讲透。

> 前置：本文面向节点端（V2bX / Xray / sing-box 均可，配置思路相通），假设你已经有一台能出网的 VPS。关于 V2bX 与面板对接失败的排查，可以先看本仓库的另一篇教程 [V2bX 对接失败：面板显示节点离线怎么办](v2bx-offline-troubleshoot.md)。

## dest 是干什么的：先理解原理再选

Reality 的核心思想是"借用"一个真实网站的 TLS 指纹。当客户端发起连接时，服务端先完成和**真实** TLS 握手一样的流程，把自己伪装成目标站点；只有带着正确密钥的客户端才能通过验证、进入代理流程，普通扫描器看到的就是一个正常的网站。

`dest`（destination）就是那个被借用的真实站点，格式为 `域名:端口`，例如 `www.microsoft.com:443`。服务端并不会真的去访问它——它只是在握手时"扮演"它。但扮演得像不像，取决于两个条件：

1. **这个站点真实存在且支持 TLS 1.3**，因为 Reality 完整复刻了 TLS 1.3 握手；
2. **这个站点的证书链和指纹特征稳定**，频繁变更的站点会让伪装前后不一致。

所以 dest 的选择标准是：大厂、长期稳定、TLS 1.3、最好和你的服务器在同一个"网络语境"里。

## 怎么选 dest：三条实操原则

**原则一：选大厂、选稳定，不要选小站。**

最常用的几个备选（任选其一即可，不用纠结）：

- `www.microsoft.com:443` —— 最经典的选择，全球 CDN，证书稳定；
- `www.apple.com:443` —— 同样稳定，TLS 1.3 支持完善；
- `www.amazon.com:443` —— 可用，但部分地区解析到的 IP 变化较频繁；
- `download.windowsupdate.com:443` —— 微软更新域名，流量特征非常"正常"。

不要选自己的域名、不要选不知名小站、不要选会频繁换证书的站点。小站的 TLS 指纹一旦变化，你的伪装就对不上，等于白搭。

**原则二：dest 的 IP 归属地最好和你的落地机接近。**

如果落地机在日本，就选在日本有 CDN 节点的站点（如微软、苹果）；落地机在美国，就选美国 CDN 覆盖好的。这不是硬性要求，但能让握手时的 SNI、证书链、乃至 TCP 行为更"合理"。可以用下面的命令看目标站点在你服务器上的解析结果是否正常：

```bash
# 看看从你的落地机解析 microsoft 是什么结果
dig +short www.microsoft.com
curl -sI --max-time 10 https://www.microsoft.com | head -5
```

只要能正常解析、443 端口能通，这个 dest 就是合格的。注意：**dest 不需要你拥有它，也不需要你能控制它**，能正常访问就行。

**原则三：一台落地机固定一个 dest，别频繁换。**

dest 写进服务端配置后，客户端的密钥对是和它绑定的（见下文 `serverNames`）。频繁更换 dest 会导致已下发的订阅全部失效，用户端要重新同步——对运营来说这是灾难。选定之后就不要动。

## 密钥生成：服务端一对、客户端一对

Reality 使用 X25519 密钥对做身份验证。服务端持有**私钥**，客户端持有**公钥**；公钥可以公开下发，私钥绝不能泄露。生成方法很简单，用 Xray 自带的命令即可：

```bash
# 如果装了 xray，直接生成
xray x25519
```

输出类似：

```
Private key: 4K7sX9mQ2vL8nP3wR6tY1uI5oP0aS9dF8gH7jK6l
Public key: 9Q2wE4rT6yU8iO0pA1sD3fG5hJ7kL9zX8cV2bN4m
```

> 为什么是 X25519 而不是 RSA/ECDSA？因为 Reality 的验证发生在 TLS 握手阶段，需要的是快速的密钥交换算法，X25519 正好合适——这也是 Reality 不需要证书的原因，验证靠的是密钥对而不是 CA 签发的证书。

把**私钥（Private key）**填进服务端的 `privateKey` 字段，把**公钥（Public key）**填进客户端配置（或面板后台节点配置）的 `publicKey` 字段。注意别填反：服务端用私钥签名、客户端用公钥验证，填反了握手直接失败，客户端日志里会看到 `reality verification failed` 之类的报错。

另外还需要一个 `shortId`（短 ID），服务端和客户端必须一致：

```bash
# 生成一个随机的 shortId（8 位十六进制）
openssl rand -hex 4
```

## 完整配置示例（Xray / V2bX 通用思路）

服务端 inbound 的关键字段如下（占位符请替换成你自己的值）：

```json
{
  "protocol": "vless",
  "settings": {
    "clients": [
      {
        "id": "你的UUID-用-uuidgen-生成",
        "flow": "xtls-rprx-vision"
      }
    ],
    "decryption": "none"
  },
  "streamSettings": {
    "network": "tcp",
    "security": "reality",
    "realitySettings": {
      "show": false,
      "dest": "www.microsoft.com:443",
      "xver": 0,
      "serverNames": ["www.microsoft.com"],
      "privateKey": "上面生成的Private key",
      "minClientVer": "",
      "maxClientVer": "",
      "maxTimeDiff": 0,
      "shortIds": ["上面生成的shortId"]
    }
  }
}
```

几个关键参数解释一句：

- `dest`：借用的真实站点，`域名:443` 格式，按上文三原则选；
- `serverNames`：允许客户端使用的 SNI 列表，**必须和 dest 的域名一致**，否则握手时 SNI 对不上会直接拒绝；
- `show: false`：不向未验证的连接展示任何信息，保持"看起来像个正常网站"；
- `flow: xtls-rprx-vision`：Reality 下推荐的流控，和 VLESS 搭配最成熟；
- `shortIds`：服务端接受的短 ID 列表，客户端配置里的 shortId 必须在这个列表里。

V2bX 用户注意：V2bX 的节点配置是由面板下发的，上述字段对应面板后台「节点配置」里的 Reality 相关项——`dest`、`serverNames`、`privateKey`、`shortIds` 在面板侧填好，V2bX 会自动生成 xray/sing-box 的实际配置，不需要你手写 JSON。

## 验证：三步确认 Reality 真的在工作

配完先别急着下发订阅，按这个顺序验证：

```bash
# 1. 服务端配置语法检查（xray 示例）
xray -test -config /usr/local/etc/xray/config.json
# 输出 "Configuration OK." 才算通过

# 2. 端口在监听
ss -tlnp | grep 443

# 3. 用一个"裸" TLS 客户端看伪装效果（不带 Reality 密钥去连）
curl -vk --max-time 10 https://你的服务器IP:443 -o /dev/null
```

第 3 步最关键：**不带密钥的连接应该看到和 dest 站点一致的证书**（比如微软的证书），而不是报错或暴露代理特征。如果看到的是 dest 站点的证书，说明伪装成功；如果直接连接被拒绝或返回异常，先检查 `serverNames` 和 `dest` 的域名是否一致。

然后再用带正确公钥和 shortId 的客户端连一次，能通就说明整条链路 OK。

## 常见坑

1. **serverNames 和 dest 域名不一致**：最常见的手滑。dest 写 `www.microsoft.com:443`，serverNames 却写了 `microsoft.com`（少了 www）——握手时 SNI 对不上，直接失败。两处域名必须字符级一致。
2. **私钥公钥填反**：服务端填公钥、客户端填私钥， symptom 是所有客户端都连不上，服务端日志出现 reality 验证失败。记住：服务端=私钥，客户端=公钥。
3. **shortId 对不上**：面板下发的 shortId 和服务端 `shortIds` 列表里的不一致。V2bX + 面板模式下，以面板后台填的值为准，改完要重启 V2bX 生效。
4. **dest 选了被墙的站点**：如果 dest 域名本身在你用户所在地区打不开，伪装就失去了意义——审查者看到的是一个"连正常用户都打不开的站点"，反而可疑。选全球可达的大厂域名。
5. **一台机器多个 Reality inbound 共用密钥**：技术上可行，但一旦其中一个 inbound 的配置泄露，所有 inbound 一起遭殃。每个 inbound 单独生成一对密钥，`openssl rand -hex` 和 `xray x25519` 都是秒级操作，别省这一步。

dest 选对、密钥配对，Reality 这部分就算彻底搞定了。剩下的就是把节点信息录入面板、下发订阅——这部分如果遇到面板显示节点离线，回头看 [V2bX 对接失败排查](v2bx-offline-troubleshoot.md) 那篇，排查思路是通的。

## 不想自己搭？

从选 dest、生成密钥、写 inbound 配置，到面板对接、下发订阅、半夜起来修离线节点——自己搭一套能稳定跑的机场，前前后后搭进去的时间往往以"天"为单位，还不算后续的运维。如果你只是想有个稳定可用的服务，而不是想学运维，[漫画机场](https://vps.manhuajichang.fun) 提供了现成的订阅：标准套餐 ¥10/月起（含 100GB/月流量、支持 3 台设备），需要更好线路的还有家宽套餐 ¥30/月起，开箱即用，不用折腾上面的任何一步。
