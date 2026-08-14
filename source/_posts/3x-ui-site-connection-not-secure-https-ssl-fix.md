---
title: 3x-ui 提示“此站点的连接不安全”怎么办？HTTPS 与 SSL 证书完整解决教程
date: 2026-08-14 19:30:00
updated: 2026-08-14 19:30:00
tags: [3x-ui, HTTPS, SSL证书, Let's Encrypt, Cloudflare, Nginx]
categories: [技术教程, 服务器运维]
keywords: 3x-ui连接不安全,3x-ui HTTPS,3x-ui SSL证书,此站点的连接不安全,ERR_CERT_COMMON_NAME_INVALID,ERR_CERT_AUTHORITY_INVALID,3x-ui证书申请
description: 3x-ui 面板提示“此站点的连接不安全”的原因与解决方法，涵盖域名解析、Let's Encrypt 证书、裸 IP 证书、Cloudflare DNS 验证、反向代理及常见证书错误排查。
ai: 本文面向 3x-ui 新手，解释浏览器“不安全”提示的原因，并提供域名证书、IP 证书和反向代理三种修复方案及验证命令。
aside: true
---

## 📖 前言

搭建完 3x-ui 后，第一次打开管理面板，不少人会在浏览器地址栏看到：

```text
此站点的连接不安全
```

或者直接进入“您的连接不是私密连接”错误页，并显示以下代码之一：

```text
NET::ERR_CERT_AUTHORITY_INVALID
NET::ERR_CERT_COMMON_NAME_INVALID
NET::ERR_CERT_DATE_INVALID
ERR_SSL_PROTOCOL_ERROR
```

这通常不是 Xray 节点损坏，而是 **3x-ui 管理面板的 HTTPS 没有正确配置**。节点入站和管理面板是两个相对独立的部分：节点能正常连接，不代表面板已经受到 HTTPS 保护。

> 3x-ui 面板会传输管理员账号、密码、会话 Cookie 和配置数据。只要面板能够从公网访问，就不应该长期使用明文 HTTP，更不建议点击“继续访问”后直接输入管理员密码。

<!-- more -->

## 🔍 一、为什么会显示“连接不安全”

浏览器是否信任一个 HTTPS 网站，主要检查三件事：

1. 当前连接是否真正使用 HTTPS 加密。
2. 服务器证书是否由浏览器信任的证书机构签发。
3. 证书中的域名或 IP 是否与地址栏中的访问地址一致。

只要其中一项不满足，浏览器就可能发出警告。

### 原因 1：正在使用普通 HTTP

例如：

```text
http://1.2.3.4:2053/abc123/
```

地址以 `http://` 开头，数据没有经过 TLS 加密，因此浏览器显示“不安全”属于正常现象。它不是浏览器误报，也不能通过清理缓存解决。

### 原因 2：证书签发给域名，却通过 IP 访问

假设证书签发给：

```text
panel.example.com
```

正确访问方式是：

```text
https://panel.example.com:2053/abc123/
```

如果改用下面的 IP 地址访问：

```text
https://1.2.3.4:2053/abc123/
```

浏览器会发现地址栏里的 IP 不在证书有效名称中，于是报告：

```text
ERR_CERT_COMMON_NAME_INVALID
```

### 原因 3：使用了自签名证书

自签名证书同样可以加密连接，但它不是由浏览器默认信任的公共证书机构签发，因此经常出现：

```text
ERR_CERT_AUTHORITY_INVALID
```

在完全由自己管理的内网环境中，可以把自己的 CA 导入每台设备；面向公网使用时，更省事的做法是申请 Let's Encrypt 等公共证书。

### 原因 4：证书过期或访问设备时间错误

证书有明确的生效时间和失效时间。证书已经过期、尚未生效，或者正在访问面板的电脑、手机日期明显错误，都可能触发：

```text
ERR_CERT_DATE_INVALID
```

浏览器主要根据访问设备的时间检查证书有效期。VPS 时间错误通常会影响证书申请、自动续期和日志记录，但不是浏览器验证一张既有证书时的直接时间依据。

### 原因 5：3x-ui 的证书路径不完整或错误

3x-ui 面板自身启用 HTTPS，需要同时设置：

- `webCertFile`：完整证书链文件
- `webKeyFile`：与证书配对的私钥文件

只设置一项、文件不存在、权限不足，或把两个路径填反，都可能导致 HTTPS 无法启动。证书文件建议使用 `fullchain.pem`，不要只使用缺少中间证书的单证书文件。

### 原因 6：反向代理的协议配置不一致

如果前面还有 Nginx、Caddy 或 Cloudflare，常见错误包括：

- 浏览器到 Nginx 使用 HTTPS，但 Nginx 的证书配置错误
- 3x-ui 只提供 HTTP，反向代理却按 HTTPS 回源
- 3x-ui 已经启用 HTTPS，反向代理仍按 HTTP 回源
- 443 端口被多个程序争用

这类问题常表现为 `ERR_SSL_PROTOCOL_ERROR`、502 错误或页面完全打不开。

## ✅ 二、推荐方案：域名 + Let's Encrypt 证书

这是大多数用户最适合的方案。以下示例假设：

| 项目 | 示例值 |
| --- | --- |
| 面板域名 | `panel.example.com` |
| VPS 公网 IP | `1.2.3.4` |
| 面板端口 | `2053` |
| 面板路径 | `abc123` |

操作时请替换成自己的真实信息。

### 第 1 步：添加 DNS 解析

在域名 DNS 管理页面添加一条 `A` 记录：

```text
类型：A
主机记录：panel
记录值：1.2.3.4
```

如果服务器没有配置公网 IPv6，不要添加 `AAAA` 记录；已经存在但指向错误服务器的 `AAAA` 记录也应删除，否则部分设备可能优先连接错误的 IPv6 地址。

等待解析生效后检查：

```bash
getent ahosts panel.example.com
```

返回地址应包含当前 VPS 的公网 IP。

### 第 2 步：确认 80 端口可用于验证

3x-ui 内置的域名证书申请通常通过 ACME HTTP 验证完成，需要公网能够访问 TCP 80。

使用 UFW 时可以执行：

```bash
ufw allow 80/tcp
ufw allow from 你的固定公网IP to any port 2053 proto tcp
ufw status
```

如果没有固定的管理员公网 IP，可以临时使用 `ufw allow 2053/tcp` 完成测试，确认后再收紧访问来源。如果面板端口不是 `2053`，请换成实际端口。云服务器控制台中的安全组也必须同步设置。

这里不必默认开放 `443`：只有 Nginx、Caddy、3x-ui 或其他公网服务实际监听 443 时才需要开放。启用 UFW 之前，还必须先放行真实的 SSH 端口，避免把自己锁在服务器外面。

检查 80 端口是否已被其他程序占用：

```bash
ss -lntp | grep ':80 '
```

如果 Nginx、Apache 或 Caddy 正在占用 80 端口，可暂时停止对应服务后申请证书，或者改用后文的 Cloudflare DNS 验证方案。

### 第 3 步：在 3x-ui 菜单申请证书

SSH 登录 VPS，运行：

```bash
x-ui
```

进入 `SSL Certificate Management`。不同版本的菜单编号可能变化，新版通常显示为 `20`，操作时以屏幕中的文字为准。

依次选择：

```text
Get SSL (Domain)
```

输入：

```text
panel.example.com
```

申请成功后，证书一般位于：

```text
/root/cert/panel.example.com/fullchain.pem
/root/cert/panel.example.com/privkey.pem
```

### 第 4 步：把证书路径绑定到面板

继续在证书管理菜单中选择：

```text
Set Cert paths for the panel
```

确认对应关系：

```text
Certificate File Path: /root/cert/panel.example.com/fullchain.pem
Key File Path:         /root/cert/panel.example.com/privkey.pem
```

然后重启面板：

```bash
x-ui restart
```

3x-ui 只有在证书文件和私钥文件两项都正确设置后，面板才会切换到 HTTPS。

### 第 5 步：使用正确地址访问

现在应该使用：

```text
https://panel.example.com:2053/abc123/
```

注意四个细节：

- 必须是 `https://`
- 必须使用证书对应的域名
- 自定义面板端口不能漏掉
- 随机面板路径不能漏掉，结尾最好保留 `/`

不要继续通过服务器 IP 访问域名证书。

## 🌐 三、没有域名：申请裸 IP 证书

较新的 3x-ui 版本提供了：

```text
Get SSL for IP Address
```

它可以为公网 IP 申请短期证书。进入方式通常为：

```text
x-ui → SSL Certificate Management → Get SSL for IP Address
```

签发完成后，同样选择 `Set Cert paths for the panel`，再重启 3x-ui。之后通过证书绑定的公网 IP 访问。

官方文档说明，这类 IP 证书有效期约 6 天，需要依赖自动续期。建议定期检查续期状态；如果长期使用管理面板，域名证书仍然更直观。

> 如果旧版 3x-ui 菜单中没有 IP 证书选项，请先备份数据库和配置，再按照官方升级方法更新，不要随意运行来源不明的改版脚本。

## ☁️ 四、Cloudflare 用户：使用 DNS-01 验证

遇到以下情况时，适合使用 Cloudflare DNS 验证：

- 服务器 80 端口无法从公网访问
- 域名开启了 Cloudflare 橙色云代理
- 需要申请通配符证书
- 80 端口正被其他网站占用，不方便停止

前提是域名 DNS 已经托管到 Cloudflare。

### 创建最小权限 API Token

在 Cloudflare 控制台创建 API Token，仅授予目标域名区域：

```text
Zone → DNS → Edit
```

不要把 Global API Key 粘贴到不可信脚本中。最小权限 Token 泄露后的影响范围更小。

### 在 3x-ui 中申请

运行：

```bash
x-ui
```

选择 `Cloudflare SSL Certificate`，然后选择 API Token 模式，输入域名和 Token。签发成功后，再把证书路径设置给面板并重启。

DNS-01 验证通过临时 TXT 记录证明域名控制权，因此不依赖公网 80 端口。

如果域名开启 Cloudflare 橙色云，还要确认面板端口属于 Cloudflare 支持的 HTTPS 代理端口。常见支持端口包括 `443`、`2053`、`2083`、`2087`、`2096` 和 `8443`；任意自定义高位端口不一定能被代理。

## 🔁 五、进阶方案：Nginx 或 Caddy 反向代理

另一种思路是让 3x-ui 只在服务器本机提供 HTTP，由 Nginx 或 Caddy 统一处理公网 HTTPS：

```text
浏览器 ──HTTPS──> Nginx/Caddy ──HTTP──> 127.0.0.1:2053
```

这种结构适合服务器上已经运行网站、需要统一使用 443 端口的用户。

下面是一个简化的 Nginx 反向代理片段：

```nginx
location / {
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://127.0.0.1:2053;
}
```

完整的 HTTPS `server` 配置还需要证书路径、私钥路径和域名。配置完成后先检查语法：

```bash
nginx -t
```

然后重新加载：

```bash
systemctl reload nginx
```

使用反向代理终止 TLS 时，应注意：

- 3x-ui 和 Nginx 的回源协议必须对应
- 最好让 3x-ui 只监听 `127.0.0.1`，或用防火墙禁止公网直接访问面板端口
- 不要让 Nginx 和 3x-ui 同时监听公网 443
- 面板设置中的 URL 和路径要保持一致

如果只是第一次搭建，直接给 3x-ui 配置域名证书通常更简单；已经熟悉 Nginx/Caddy 再考虑反向代理。

## 🧪 六、如何检查证书是否真的正常

### 1. 检查 3x-ui 服务

```bash
x-ui status
systemctl status x-ui --no-pager
```

### 2. 查看最近日志

```bash
journalctl -u x-ui -n 100 --no-pager
```

重点搜索：

```text
certificate
private key
permission denied
no such file
bind
address already in use
```

### 3. 检查监听端口

```bash
ss -lntp
```

确认面板实际监听的是哪个端口，以及该端口有没有被其他程序占用。

### 4. 检查证书有效期和签发对象

```bash
openssl s_client \
  -connect panel.example.com:2053 \
  -servername panel.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

正常情况下会显示：

- `subject`：证书对应的域名
- `issuer`：证书签发机构
- `notBefore`：生效时间
- `notAfter`：到期时间

检查证书包含的域名：

```bash
openssl s_client \
  -connect panel.example.com:2053 \
  -servername panel.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName
```

输出中应包含 `DNS:panel.example.com`。

### 5. 使用 curl 检查握手

```bash
curl -Iv https://panel.example.com:2053/
```

如果面板使用随机路径，根目录返回 404 不一定代表 HTTPS 失败；只要 TLS 握手和证书验证成功，再访问完整面板路径即可。

## ❌ 七、根据错误代码快速定位

| 浏览器提示 | 常见原因 | 处理方法 |
| --- | --- | --- |
| 地址栏显示“不安全” | 使用普通 HTTP | 配置证书并改用 `https://` |
| `ERR_CERT_AUTHORITY_INVALID` | 自签名证书、证书链不完整、本机根证书异常或 HTTPS 检查软件替换证书 | 检查实际返回的签发者，使用公共证书并指向 `fullchain.pem` |
| `ERR_CERT_COMMON_NAME_INVALID` | 访问地址与证书名称不一致 | 使用证书对应域名，或申请匹配的 IP 证书 |
| `ERR_CERT_DATE_INVALID` | 证书过期、尚未生效或访问设备时间错误 | 续签证书并校准电脑或手机时间；同时检查 VPS 时间以免影响续期 |
| `ERR_SSL_PROTOCOL_ERROR` | HTTPS 访问了 HTTP 端口、证书路径错误 | 核对协议、端口、证书路径和日志 |
| 连接超时 | 防火墙、安全组或 DNS 错误 | 放行实际端口并检查 A/AAAA 记录 |
| 502 Bad Gateway | 反向代理无法连接 3x-ui | 核对回源地址、端口、HTTP/HTTPS 协议 |

## 🛠️ 八、几个最容易踩的坑

### 1. 申请成功，但没有设置给面板

“证书已经签发”和“面板正在使用证书”是两件事。申请完成后还要执行 `Set Cert paths for the panel` 并重启服务。

### 2. 用 `cert.pem` 代替 `fullchain.pem`

部分浏览器可能因缺少中间证书而不信任连接。给面板配置时优先使用：

```text
fullchain.pem
privkey.pem
```

### 3. Cloudflare 开了代理，却仍用错误方式验证

开启橙色云后，访问流量先到 Cloudflare。若 HTTP 验证反复失败，可以临时关闭代理，或者直接使用 Cloudflare DNS-01 验证。

### 4. 忘记云厂商安全组

UFW 显示允许，不代表公网一定能访问。AWS、Google Cloud、阿里云、腾讯云、Oracle Cloud 等平台通常还有独立安全组或网络防火墙。

### 5. 修好证书后仍用旧书签

旧书签可能保存的是 `http://IP:端口/路径`。请重新保存完整的 HTTPS 域名地址。

### 6. 面板路径直接暴露

HTTPS 只能保护传输过程，不能代替访问控制。建议同时做到：

- 使用非默认面板端口
- 使用足够长的随机面板路径
- 修改默认账号和密码
- 开启 2FA
- 只允许自己的固定 IP 访问，或通过 VPN/SSH 隧道管理
- 不公开面板地址、API Token 和证书私钥

## ❓ 九、常见问题

### 节点正常，但面板提示不安全，会影响节点吗？

不一定。节点入站和面板 Web 服务可以使用不同端口、不同 TLS 配置。面板证书错误通常不会直接导致 REALITY 节点失效，但管理员登录数据仍存在安全风险，应尽快修复。

### 可以一直点击“高级 → 继续访问”吗？

不建议。你无法仅凭页面外观判断是自己的自签名证书，还是连接遭到中间人替换。公网管理面板应该使用能够正常验证的证书。

### 证书续签后需要修改路径吗？

正常情况下不需要。让续签程序在原路径更新 `fullchain.pem` 和 `privkey.pem` 即可。续签后重启 3x-ui，让面板重新读取证书：

```bash
x-ui restart
```

### REALITY 入站需要使用这个面板证书吗？

不需要。REALITY 与传统 TLS 入站的工作方式不同，不应因为面板配置 HTTPS，就随意把面板证书填进 REALITY 入站。本文处理的是 **3x-ui 管理页面的 HTTPS**。

## 📝 总结

3x-ui 出现“此站点的连接不安全”，本质上通常只有三类问题：

1. 使用的是明文 HTTP。
2. 证书不受信任、已过期或链不完整。
3. 证书与当前访问的域名/IP 不匹配。

对大多数用户，最稳妥的处理顺序是：

```text
域名解析到 VPS
→ 申请 Let's Encrypt 证书
→ 设置 fullchain.pem 与 privkey.pem
→ 重启 3x-ui
→ 使用 HTTPS 域名访问
```

没有域名可以使用新版 3x-ui 的裸 IP 证书；已经使用 Nginx、Caddy 或 Cloudflare，则应明确由哪一层负责 TLS，避免重复配置和协议混乱。

## 📚 参考资料

- [3x-ui 官方 SSL 证书文档（中文）](https://github.com/MHSanaei/3x-ui/blob/main/docs/content/docs/zh/config/ssl-certificates.mdx)
- [3x-ui 面板设置与 TLS 说明](https://github.com/MHSanaei/3x-ui/blob/main/docs/content/docs/en/config/panel.mdx)
- [3x-ui 官方 Wiki：证书与反向代理配置](https://github.com/MHSanaei/3x-ui/wiki/Configuration)
- [Google Chrome：证书与连接错误说明](https://support.google.com/chrome/answer/95669?hl=zh-Hans)
- [Cloudflare：支持代理的网络端口](https://developers.cloudflare.com/fundamentals/reference/network-ports/)

---

如果这篇教程帮你解决了问题，欢迎收藏本站。配置过程中不要在评论区公开服务器 IP、面板路径、管理员密码、API Token 或证书私钥。
