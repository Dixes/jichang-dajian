# sing-box 做服务端：手动配置入门

V2bX 对接面板很方便，但想完全掌控每个细节，手动写 sing-box 配置是必经之路。sing-box 用一个 JSON 文件描述全部行为：入站怎么接、出站怎么走、路由怎么分。学会手写配置，任何冷门需求都能自己实现，不再被面板的功能边界卡住。

## 准备工作

一台落地服务器，一个 UUID（用作用户凭证）：

```bash
# 安装 sing-box
bash <(curl -fsSL https://sing-box.app/install.sh)
# 生成 UUID
sing-box generate uuid
```

`sing-box generate uuid` 生成标准 UUID，直接用，不要自己编，保证唯一性和格式正确。配置文件放在 `/etc/sing-box/config.json`。

## 写第一个服务端配置

最简可用的 VLESS + Reality 配置：

```json
{
  "log": { "level": "warn" },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        { "uuid": "你的UUID", "flow": "xtls-rprx-vision" }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.example.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.example.com",
            "server_port": 443
          },
          "private_key": "你的私钥",
          "short_id": ["abcd1234"]
        }
      }
    }
  ],
  "outbounds": [
    { "type": "direct", "tag": "direct" }
  ]
}
```

关键字段：`listen_port` 对外端口；`users[].uuid` 是用户凭证；`flow: xtls-rprx-vision` 开启 vision 流控，Reality 下必须开；`reality.private_key` 用 `sing-box generate reality-keypair` 生成；`short_id` 是连接标识，任意 8 位十六进制。

私钥生成：

```bash
sing-box generate reality-keypair
# 输出 PrivateKey 和 PublicKey，私钥填服务端，公钥给客户端
```

## 启动与检查

```bash
# 检查配置语法
sing-box check -c /etc/sing-box/config.json
# 启动
systemctl enable --now sing-box
systemctl status sing-box
```

`sing-box check` 是救命命令：JSON 少个逗号、多层括号，都会在这里报出具体行号。改完配置先 check 再重启，别直接重启猜错在哪。

查看运行日志：

```bash
journalctl -u sing-box -f
```

## 对接面板（可选）

手写配置的节点也可以进 Xboard：在面板新增节点时选「sing-box」或自定义类型，把 UUID、地址、端口、Reality 公钥等参数填进去，订阅下发给用户。流量统计需要节点定期上报，纯手写 sing-box 默认不上报，如需统计可用 V2bX 方案或在 sing-box 侧加统计脚本。

## 常见坑

一是 JSON 语法错误，90% 的启动失败都是这个原因，先 check。二是 Reality 密钥对搞混，私钥放服务端、公钥给客户端，反了连不上。三是 `server_name` 和 handshake 目标不一致，握手失败。四是防火墙没放行端口，`ss -tlnp` 确认监听后再查防火墙。

## 不想自己搭？

手写 sing-box 配置自由度最高，但 JSON 的每个字段都要自己负责，出问题没有面板兜底，排错全靠读日志，学习曲线陡峭。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
