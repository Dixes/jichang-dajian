# VLESS + Reality 节点搭建

VLESS + Reality 是当前机场节点的主力方案之一：它不需要域名和证书，服务端直接监听 443 端口，利用 Reality 技术伪装成访问某个知名网站的 TLS 流量，抗封锁能力强，部署也比传统 TLS 方案简单。本文在已对接 Xboard 的 V2bX 节点机上，完整配置一个 VLESS + Reality 入站。

## 准备工作

准备一台已安装并对接好 Xboard 的 V2bX 节点服务器（见 V2bX 对接篇）；确保 443 端口可用且未被其他程序占用。Reality 需要一个「目标网站」作为伪装对象，选一个与你服务器同地区、长期稳定、支持 TLS 1.3 的大站域名，例如 www.example-corp.com。注意这个域名只是用来「借指纹」，你不需要拥有它，但要保证它长期可访问。

先检查端口占用并生成密钥对：

```bash
ss -tlnp | grep 443
xray x25519
```

第一条确认 443 空闲；第二条生成 Reality 用的公私钥，记下输出的 PrivateKey 和 PublicKey。私钥放服务端，公钥写入面板节点配置给客户端用，两者必须配对，否则握手永远失败。

## 在 V2bX 中添加 Reality 入站

进入 V2bX 管理菜单，新增一个 VLESS 入站，关键参数如下：

- 监听端口：443（Reality 要求使用 443 才能与真实 HTTPS 流量混淆）
- 传输方式：tcp + reality
- 目标网站（dest）：www.example-corp.com:443，伪装流量的转发目标
- 私钥：上一步生成的 PrivateKey
- shortId：自定义 8 位十六进制字符串，用于区分同一私钥下的多个入站

用 Reality 不需要申请证书，因为它的 TLS 是服务端自己签发的「影子证书」，探测者看到的是目标网站的指纹。这就是它比传统方案省事的核心原因，但也要求 dest 选得靠谱。

## 在 Xboard 面板侧配置节点

回到 Xboard 后台的节点设置，把协议选为 VLESS，端口 443，传输 tcp，TLS 类型选 reality，并填入：

```
PublicKey: <上一步生成的公钥>
ShortId: <你设置的 shortId>
ServerName (SNI): www.example-corp.com
Fingerprint: chrome
```

`ServerName` 必须与服务端的 dest 域名一致，客户端握手时用它做 SNI，不一致直接连不上；`Fingerprint` 建议用 chrome，与主流浏览器指纹保持一致，降低被识别的概率。配置完保存，面板会自动把这些参数写进用户的订阅链接。

## 验证节点可用性

重启 V2bX 生效配置：

```bash
systemctl restart v2bx
journalctl -u v2bx -n 50 --no-pager
```

日志里没有报错即启动成功。然后在面板创建一个测试账号，取出订阅链接里的该节点信息，确认参数齐全（address、port、public-key、short-id、sni、fp）。用一台测试机实际连接，观察延迟和下载速度，连续跑几分钟没有断流即为可用。

## 常见坑

一是 443 端口被面板反代、旧的 Nginx 或其他入站占用，Reality 必须独占 443，冲突时先停掉占用的服务。二是公私钥不匹配，多为复制时混入空格或换行，重新生成一对最省事。三是 dest 选了不稳定或屏蔽境外 IP 的站点，导致回源失败，换一个全球可达的大站。四是 shortId 格式错误，必须是偶数位十六进制字符。

## 不想自己搭？

Reality 方案虽然省了证书，但调优参数、排查握手失败和长期维护依然需要投入时间。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
