# 代理协议链接格式完全指南（2026）

把 16 种代理协议的分享链接格式整理成一张表，附常见解析坑和各客户端支持矩阵。

写这份文档的原因：做[代理链接转换工具](https://yuantou.one/tools/proxy/)时踩了不少格式上的坑，网上资料要么过时、要么各说各话，索性自己按实际抓包和客户端源码核对了一遍。

---

## 一、URI 格式速查表

| 协议 | 格式 |
|---|---|
| VMess | `vmess://base64(JSON)` |
| VLESS | `vless://uuid@host:port?params#name` |
| Shadowsocks | `ss://base64(method:pass)@host:port?plugin=...#name` |
| ShadowsocksR | `ssr://base64(host:port:protocol:method:obfs:base64pass/?params)` |
| Trojan | `trojan://password@host:port?params#name` |
| Hysteria | `hysteria://auth@host:port?params#name` |
| Hysteria2 | `hysteria2://auth@host:port?params#name` |
| TUIC | `tuic://uuid:password@host:port?params#name` |
| Juicity | `juicity://uuid:password@host:port?params#name` |
| AnyTLS | `anytls://password@host:port?params#name` |
| NaiveProxy | `naive+https://user:pass@host:port#name` |
| Brook | `brook://password@host:port#name` |
| SSH | `ssh://user:pass@host:port#name` |
| Mieru | `mierus://user:pass@host:port?params#name` |
| SOCKS5 | `socks5://user:pass@host:port#name` |
| HTTP | `http://user:pass@host:port#name` |
| WireGuard | 无标准 URI，用 `.conf` 文本 |

---

## 二、逐个说明

### VMess

唯一用整段 base64(JSON) 的协议，字段名短且不统一（历史原因）：

```json
{
  "v": "2", "ps": "节点名", "add": "1.2.3.4", "port": "443",
  "id": "uuid", "aid": "0", "scy": "auto",
  "net": "ws", "type": "none", "host": "cdn.example.com", "path": "/ws",
  "tls": "tls", "sni": "cdn.example.com", "alpn": "h2,http/1.1", "fp": "chrome"
}
```

坑：

- `port` 是字符串，不是数字
- `aid`（alterId）在 VMess AEAD 里必须为 0，但老客户端仍读这个字段
- `net` 里的 `http` 表示 HTTP/2 传输，在 v2rayN 里叫 `h2`
- `type` 是伪装类型（`none` / `http` / `srv`），跟传输层是两回事
- base64 可能带 URL-safe 字符（`-_`）和不带 padding，解码前要补齐

### VLESS

```text
vless://uuid@host:port?encryption=none&security=reality&sni=www.microsoft.com
  &fp=chrome&pbk=<公钥>&sid=<短ID>&type=tcp&flow=xtls-rprx-vision#节点名
```

坑：

- `encryption=none` 是必须的，某些客户端不写会报错
- `security` 取值 `none` / `tls` / `reality` / `xtls`
- reality 必须同时有 `pbk`（public key），`sid` 可选
- `flow=xtls-rprx-vision` 只在 TCP + TLS 下有效，配 ws 会失效
- `type=grpc` 时服务名放在 `serviceName`，不是 `path`

### Shadowsocks

两种格式并存：

```text
# SIP002（新）
ss://base64(method:password)@host:port?plugin=obfs-local%3Bobfs%3Dhttp#name

# 旧格式（整段 base64）
ss://base64(method:password@host:port)#name
```

坑：

- 判断顺序很重要：**先看有没有 `@`**，有就是 SIP002，没有才是旧格式
- userinfo 部分可能是明文 `method:pass`，也可能 base64，要两种都试
- 密码里可能有 `@` 或 `:`，必须用 `lastIndexOf('@')` 切分，不能用第一个
- `plugin` 参数里的分号要 URL 编码成 `%3B`，但很多客户端不做，解析时要兼容

### ShadowsocksR

```text
ssr://base64(
  host:port:protocol:method:obfs:base64url(password)
  /?obfsparam=base64url(...)&protoparam=base64url(...)&remarks=base64url(...)&group=base64url(...)
)
```

坑：

- 密码本身还要再 base64 一次（双重编码）
- 参数值全部 base64url 编码
- `remarks` 是节点名，编码后可能含 `+`，URL 传输时会被解成空格

### Trojan

```text
trojan://password@host:port?type=ws&sni=x.com&host=x.com&path=/tj&allowInsecure=1#name
```

坑：

- Trojan 强制 TLS，URI 里通常不写 `security=tls`，但必须有 `sni`
- `allowInsecure` 有的客户端写成 `skip-cert-verify`，解析要都认
- 密码里可能有 `@`，同样用 `lastIndexOf`

### Hysteria / Hysteria2

```text
hysteria2://password@host:port?sni=h.com&insecure=1&obfs=salamander&obfs-password=xxx#name
```

坑（这个我亲自踩过）：

- `"hysteria2://"` 长度是 **12**，不是 11。`hysteria://` 才是 11。差一个字节，密码前面就会多出一个 `/`
- Hysteria2 的 `obfs` 只有 `salamander` 一种，密码另给
- `upmbps` / `downmbps` 是可选带宽，不写就自适应
- 老版 Hysteria1 的格式完全不同（`auth` 可以是 base64 的 `user:pass`）

### TUIC

```text
tuic://uuid:password@host:port?sni=x.com&alpn=h3&congestion_control=bbr&udp_relay_mode=native#name
```

坑：

- v4 和 v5 的 `udp_relay_mode` 取值不同（v4 只有 `native`/`quic`）
- `congestion_control` 可选 `cubic` / `new_reno` / `bbr`
- 有些客户端把 token 放在 `password` 位置，实际是 `uuid:password` 拼起来的

### WireGuard

没有标准 URI，只能用 `.conf`：

```ini
[Interface]
PrivateKey = ...
Address = 10.0.0.2/32
DNS = 1.1.1.1
MTU = 1420

[Peer]
PublicKey = ...
PresharedKey = ...          # 可选
Endpoint = 1.2.3.4:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

坑：

- 一个 `.conf` 可能有多个 `[Peer]`，转成客户端配置时要合并成一个出站（或拆成多个）
- `Address` 可能有多个（IPv4 + IPv6），用逗号分隔
- `AllowedIPs` 决定分流，`0.0.0.0/0` 是全量

---

## 三、各客户端支持矩阵

| 协议 | Clash/Mihomo | sing-box | Surge | Quantumult X | Loon |
|---|---|---|---|---|---|
| VMess | ✅ | ✅ | ✅ | ✅ | ✅ |
| VLESS | ✅ | ✅ | ❌ | ✅ | ✅ |
| Shadowsocks | ✅ | ✅ | ✅ | ✅ | ✅ |
| SSR | ✅ | ✅ | ❌ | ✅ | ❌ |
| Trojan | ✅ | ✅ | ✅ | ✅ | ✅ |
| Hysteria2 | ✅ | ✅ | ✅ | ⚠️ 部分 | ✅ |
| TUIC | ✅ | ✅ | ✅ | ✅ | ✅ |
| Juicity | ✅ 1.19+ | ❌ | ❌ | ❌ | ❌ |
| AnyTLS | ✅ 1.19+ | ✅ 1.12+ | ❌ | ❌ | ❌ |
| NaiveProxy | ❌ | ✅ | ❌ | ❌ | ❌ |
| WireGuard | ✅ | ✅ | ✅ | ✅ | ✅ |
| SSH | ❌ | ✅ | ❌ | ❌ | ❌ |
| Mieru | ❌ | ✅ 1.12+ | ❌ | ❌ | ❌ |
| SOCKS5 | ✅ | ✅ | ✅ | ✅ | ✅ |
| HTTP | ✅ | ✅ | ✅ | ✅ | ✅ |

**转换到某个客户端时，不支持的协议必须明确提示，不能静默丢弃** —— 用户以为转换成功，导入后少一半节点，这个体验最差。

---

## 四、转换时容易出错的边界

1. **URL 里带 `#` 的节点名**：`#` 后面是名称，但名称本身可能含 `#`，要整体 decodeURIComponent
2. **IPv6 地址**：必须用 `[::1]:443` 的方括号形式，否则端口切分错误
3. **端口缺失**：很多分享链接省略端口，按协议默认值补（VMess 443、SS 8388、Trojan 443）
4. **base64 padding**：`=` 可能被省略，解码前按长度补 `=`
5. **URL-safe base64**：`+` 和 `/` 可能被换成 `-` 和 `_`，解码前要还原
6. **中文节点名**：必须 URL 编码，否则某些客户端解析失败
7. **空值参数**：`?sni=` 这种空值应该跳过，不要拼进配置

---

## 五、一个验证方法

判断解析器写得对不对，最直接的办法是**往返测试**：

```
链接 → 解析成对象 → 重新生成链接 → 再解析
两次得到的 server / port / uuid / password 必须完全一致
```

不一致的地方就是 bug。我自己的解析器就是这么测出 `hysteria2://` 前缀长度写错的。

---

## 相关

- 在线转换工具：https://yuantou.one/tools/proxy/ （16 协议 → 11 种输出格式）
- 开源代码：https://github.com/cointaboo/web-tools-collection

## 许可

MIT。有错误欢迎提 Issue。
