# 自建订阅转换（subconverter）

不同客户端支持的订阅格式不一样：有的要 Clash YAML，有的要 Surge 配置，有的只认 base64 链接。subconverter 就是干这个转换的——把面板输出的通用订阅，转换成各种客户端能直接导入的格式。自建一个，不用依赖公共转换服务，更稳定也更保护用户隐私。

## 准备工作

准备一台能访问外网的小服务器（1 核 1G 就够），和一个子域名如 sub.example.com。subconverter 是 C++ 写的单二进制，资源占用很小。

```bash
mkdir -p /opt/subconverter && cd /opt/subconverter
wget https://github.com/tindy2013/subconverter/releases/latest/download/subconverter_linux64.tar.gz
tar -xzf subconverter_linux64.tar.gz
```

解压后得到 `subconverter` 可执行文件和 `base/` 配置目录。

## 部署与配置

创建 systemd 服务：

```bash
cat > /etc/systemd/system/subconverter.service <<'EOF'
[Unit]
Description=subconverter
After=network.target
[Service]
WorkingDirectory=/opt/subconverter
ExecStart=/opt/subconverter/subconverter
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl enable --now subconverter
```

默认监听 25500 端口。用反代加上 HTTPS：

```nginx
server {
    listen 443 ssl;
    server_name sub.example.com;
    ssl_certificate /etc/letsencrypt/live/sub.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sub.example.com/privkey.pem;
    location / {
        proxy_pass http://127.0.0.1:25500;
    }
}
```

HTTPS 是必须的，订阅链接里含用户全部节点信息，明文传输等于裸奔。证书用 acme.sh 申请，参考反代与证书篇。

## 生成各客户端订阅

subconverter 的核心接口是 `/sub`，参数说明：

```
https://sub.example.com/sub?target=clash&url=<面板订阅链接urlencode后>&config=<规则配置url>
```

`target` 决定输出格式：clash、surge、quanx、loon、v2ray（base64）等；`url` 是 Xboard 给用户的原始订阅链接；`config` 是分流规则配置，不传则用默认。

在 Xboard 后台 → 系统设置 → 订阅设置里，把各客户端的订阅地址模板填成你自建的 subconverter 地址，用户点击「复制 Clash 订阅」时拿到的就是转换好的链接，一步到位。

## 验证

```bash
# 测试转换是否正常
curl -s "https://sub.example.com/sub?target=clash&url=$(echo -n 'https://panel.example.com/api/v1/client/subscribe?token=xxx' | python3 -c 'import sys,urllib.parse;print(urllib.parse.quote(sys.stdin.read()))')" | head -20
```

返回的应该是 YAML 格式的 Clash 配置，包含 proxies 列表。再分别测试 surge、v2ray 等 target，确认每种格式都能正常解析。最后用真实客户端导入，确认节点可用、分组正确。

## 常见坑

一是 `url` 参数没做 urlencode，`&` 和 `?` 把参数截断，转换返回空。二是 subconverter 版本太旧，不支持新的 Reality 参数，定期更新到最新 release。三是规则 config 用了失效的外链，转换超时，尽量把规则文件放自己服务器上。四是没加 HTTPS，某些客户端拒绝导入 http 订阅。

## 不想自己搭？

自建 subconverter 本身不难，但规则配置、分流策略的调优是个无底洞：规则一变就要重新测试，出问题用户先感知。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
