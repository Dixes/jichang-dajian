# 域名与 Cloudflare 配置要点

域名是机场的门面：面板地址、订阅链接、支付回调全挂在域名上；Cloudflare 则提供免费的 CDN 隐藏源站、抗 DDoS 和一键 HTTPS。两者配错是机场上线后故障最多的环节——订阅打不开、支付回调失败，十有八九是这里的问题。本文把机场常用的域名规划和 Cloudflare 配置一次讲清。

## 准备工作

准备一个在境外注册商购买的域名，例如 example.com（国内注册商的域名做解析跳转容易被要求备案）；注册一个 Cloudflare 账号。域名规划建议一次想好：

- `panel.example.com` → 面板后台（开启 CF 代理，隐藏源站）
- `sub.example.com` → 订阅下发（开启 CF 代理）
- `node.example.com` → 直连类节点（DNS only，不代理）

面板和订阅走代理能隐藏源站 IP，被攻击时只打到 CF；节点域名保持直连，因为代理会干扰 Reality、Hysteria2 等协议的握手，UDP 更是完全走不了 CF 代理。

## 把域名接入 Cloudflare

在 CF 添加站点，按提示把域名的 NS 记录改成 CF 分配的两个 nameserver：

```
xxx.ns.cloudflare.com
xxx.ns.cloudflare.com
```

生效一般要几分钟到几小时，期间用 `dig NS example.com` 确认。NS 生效前不要反复改解析，频繁变更会延长生效时间。接入后在 DNS 页面按上面的规划添加 A 记录，面板和订阅子域打开橙色云朵（代理），节点子域保持灰色云朵（DNS only）。

## SSL/TLS 模式选择

这是最关键的一步。CF 的 SSL/TLS 加密模式建议选「完全（严格）」：

```
SSL/TLS → 加密模式 → 完全（严格）
```

「灵活」模式下 CF 到源站是明文 HTTP，看似省事，但面板登录、支付回调走明文极不安全；「完全（严格）」要求源站也有有效证书，配合后面证书篇的自动续期，一劳永逸。模式选错的典型症状：无限重定向循环（ERR_TOO_MANY_REDIRECTS），八成是源站强制 HTTPS 而 CF 用了灵活模式。

## 常用规则配置

在 CF 后台做三处加固：第一，给 `panel.example.com` 开启「始终使用 HTTPS」和 HSTS，防止后台登录被降级劫持；第二，在 WAF 里对 `/admin` 路径加一条速率限制规则，挡掉爆破面板密码的扫描器；第三，关闭不必要的「电子邮件地址混淆」之外的性能选项时注意：Rocket Loader、Auto Minify 有时会打乱 Xboard 前端渲染，出样式错乱先关这两个再排查。

```
规则示例：
(路径 包含 /admin) 且 (5 分钟内请求 > 20 次) → 质询
```

速率限制的阈值按自己登录频率设，别把自己锁在外面，锁了就去 CF 后台放行自己的 IP。

## 验证配置是否生效

依次验证：`dig panel.example.com` 返回的是 CF 的 Anycast IP 而不是源站 IP，说明代理生效、源站已隐藏；浏览器访问面板域名，证书颁发者显示 Cloudflare 且没有警告；`dig node.example.com` 返回源站真实 IP，说明节点直连未被代理；用手机网络打开订阅链接能正常拉取。

```bash
dig +short panel.example.com
dig +short node.example.com
curl -I https://panel.example.com | head -5
```

三条命令分别验证代理、直连和 HTTPS 状态，全部正常才算配完。

## 常见坑

一是节点域名误开橙色云朵，导致 Reality 握手失败、Hysteria2 完全不通，记住节点一律灰色云朵。二是 SSL 模式用了「灵活」又开了源站强制跳转，造成重定向死循环。三是改了 NS 后旧解析商的记录还在生效期，表现为时好时坏，等 TTL 过期即可。四是把 API Token 或 Zone ID 贴到公开地方，CF 账号被接管后域名解析任人摆布。

## 不想自己搭？

域名和 Cloudflare 的组合配置看着简单，但 SSL 模式、代理开关、WAF 规则任何一个配错都会导致整站故障，排查起来很耗时间。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
