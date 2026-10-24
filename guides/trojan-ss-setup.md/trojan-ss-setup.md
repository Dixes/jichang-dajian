# Trojan 与 Shadowsocks 节点速通

VLESS 和 Hysteria2 是主力，但有些场景需要备选协议：Trojan 伪装成标准 HTTPS 流量，适合对特征敏感的线路；Shadowsocks 轻量快速，适合低端设备和老旧客户端。多备一两个协议，节点池的兼容性会好很多。

## 准备工作

一台落地服务器、一个域名（Trojan 需要，SS 不需要）。先申请证书（Trojan 必须用有效证书，参考 Caddy/Nginx 篇）：

```bash
# Trojan 依赖有效证书，先确认证书已就绪
ls /etc/letsencrypt/live/tj.example.com/
```

证书路径记下来，后面配置要用。

## Trojan 节点部署

用 sing-box 一行入站搞定：

```json
{
  "type": "trojan",
  "tag": "trojan-in",
  "listen": "::",
  "listen_port": 443,
  "users": [{ "password": "设置的强密码" }],
  "tls": {
    "enabled": true,
    "server_name": "tj.example.com",
    "certificate_path": "/etc/letsencrypt/live/tj.example.com/fullchain.pem",
    "key_path": "/etc/letsencrypt/live/tj.example.com/privkey.pem",
    "fallback": {
      "server": "127.0.0.1",
      "server_port": 8080
    }
  }
}
```

关键点：`password` 是用户凭证，用强随机字符串；`fallback` 把非法访问转到本地伪装站，别人直接访问域名看到的是正常网站，这是 Trojan 的精髓。V2bX 也支持 Trojan 入站，在面板里选 Trojan 类型填同样参数即可对接。

在 Xboard 新增节点：协议选 Trojan，地址填域名，端口 443，密码填上面设置的，TLS 开启。

## Shadowsocks 节点部署

SS 配置最简单，不需要域名和证书：

```json
{
  "type": "shadowsocks",
  "tag": "ss-in",
  "listen": "::",
  "listen_port": 8388,
  "method": "2022-blake3-aes-128-gcm",
  "password": "设置的强密码"
}
```

`method` 建议用 2022-blake3 系列加密方式，性能和安全性都比老旧的 aes-256-gcm 好；`password` 长度按加密方式要求来，2022-blake3-aes-128-gcm 需要 16 字节。注意 SS 流量特征明显，不要作为主力协议暴露，适合做备用。

## 验证

```bash
# 端口监听检查
ss -tlnp | grep -E '443|8388'
# Trojan 证书检查
openssl s_client -connect tj.example.com:443 -servername tj.example.com < /dev/null 2>/dev/null | openssl x509 -noout -dates
```

证书有效期正常、端口监听正常后，用测试账号实际连接两种协议，确认都能正常使用。

## 常见坑

一是 Trojan 用了自签证书或过期证书，客户端校验失败。二是 fallback 没配，直接访问域名返回错误页面，等于告诉别人这里有代理。三是 SS 密码长度不对，2022 系列对密码长度有硬性要求，报错先查长度。四是把 SS 当主力大流量用，特征明显容易被针对。

## 不想自己搭？

多协议节点池听起来美好，但每种协议都要单独维护证书、端口和监控，节点一多运维量翻倍。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
