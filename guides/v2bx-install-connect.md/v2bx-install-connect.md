# V2bX 节点端安装与对接 Xboard

V2bX 是节点端的后端程序，装在每台负责转发流量的服务器上。它从 Xboard 面板拉取节点配置和用户数据，负责协议解析、流量统计和上报。面板是控制面，V2bX 是数据面：两者配对后，你在面板里建的每个节点才真正有服务器去承载用户连接。本文从一台全新的节点服务器开始，完成 V2bX 安装、与 Xboard 的对接，以及基础核验。

## 准备工作

准备一台独立于面板服务器的境外 VPS，建议 2 核 2GB 内存起步；确认面板地址可访问，例如 https://panel.example.com；拿到 Xboard 后台的「节点对接」参数，包括通信域名（或 IP）、API Key 和节点 ID。节点服务器用独立的机器，因为面板和节点混装会互相抢资源，面板故障时也不应影响在线用户。

先同步时间并放行端口：

```bash
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp true
ufw allow 22/tcp && ufw allow 443/tcp && ufw allow 8443/tcp
```

时间必须准确，VLESS、Trojan 等协议对时间偏差敏感，偏差过大会直接导致握手失败。端口按你要用的协议放行，暂时只开 443 和 8443，后续按需增删。

## 安装 V2bX

使用官方一键脚本安装，安装过程中不要中途断开 SSH，建议先开 tmux 会话：

```bash
tmux new -s v2bx
bash <(curl -Ls https://raw.githubusercontent.com/wyx2685/V2bX-script/master/install.sh)
```

脚本执行完后输入 `v2bx` 进入管理菜单。如果脚本拉取失败，多半是 GitHub 连接被墙或 DNS 污染，换 DNS（如 8.8.8.8）或用镜像地址重试。安装后检查服务状态：

```bash
systemctl status v2bx
systemctl enable --now v2bx
```

服务必须是 active（running），开机自启必须打开，否则服务器一重启所有节点就失联。

## 对接 Xboard 面板

在 V2bX 的管理菜单里选择「配置 Xboard 后端」，依次填入：面板地址（必须与后台设置的通讯域名一致，否则签名校验失败）、API Key、节点 ID。对应 Xboard 的配置文件片段如下（示意）：

```yaml
Nodes:
  - PanelType: "V2board"
    ApiHost: "https://panel.example.com"
    ApiKey: "YOUR_API_KEY_HERE"
    NodeID: 1
    NodeType: "Vless"
```

`ApiHost` 填面板的对外域名而不是内网 IP，因为对接走的是公网 HTTPS；`ApiKey` 泄露等于把整台面板的节点和用户数据拱手让人，填写后不要截图外传；`NodeID` 要与面板后台创建的节点一一对应，一个 ID 只给一台服务器用，重复对接会互相覆盖流量统计。

保存配置后重启 V2bX 让配置生效。

## 在面板侧补齐节点信息

回到 Xboard 后台，找到对应节点，把协议类型、监听端口、传输方式等信息补齐，让面板下发的节点信息与 V2bX 本机的真实监听一致。例如 V2bX 监听 443 端口的 VLESS，面板侧也要写 443，不能一头写 443 一头写 8443。节点备注写清机房和用途，方便以后分组和排查。

## 验证对接是否成功

在节点服务器上看 V2bX 日志，出现「节点信息获取成功」或类似的同步字样即为成功：

```bash
journalctl -u v2bx -f --no-pager
```

然后到面板后台看该节点是否显示在线；再在面板建一个测试套餐和测试账号，生成订阅链接，确认订阅里能拉到这个节点。订阅能出节点、节点显示在线、流量计数在面板里增长，三者都满足才算对接完成。

## 常见坑

一是面板地址填错，比如面板用了 Cloudflare 代理却填了源站 IP，导致 API 请求被拦截或证书校验失败，统一用对外的面板域名。二是 API Key 两边不一致，改了一边没改另一边，重启 V2bX 也无济于事。三是节点服务器时间不对，先校时再排查。四是防火墙只开了 SSH 没开业务端口，表现为面板在线但用户连不上。

## 不想自己搭？

节点端对接只是开始，之后每台节点的系统更新、证书续期、线路质量监控都要持续投入时间。如果你只是想稳定使用而不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
