# Codex 反复 Reconnecting：代理环境变量修复方法

当 Codex 每次新建对话都连续出现 `Reconnecting... X/5`，等待一段时间后才恢复通信时，可以尝试让 Codex 明确使用本机代理软件的 HTTP 或 mixed 端口。

这个方法适合以下情况：

- 已经运行代理软件，但不想开启 TUN；
- 浏览网页正常，Codex 却反复重连；
- 代理软件可能是 Mihomo/Clash、V2Ray、sing-box 或其他实现；
- 希望只为 Codex 指定代理，不修改 Windows 系统代理。

> [!IMPORTANT]
> 不要直接复制别人的端口。网上出现的任何端口号都只是示例，必须使用当前电脑上代理软件实际提供的 **HTTP 或 mixed 端口**。

## 原理与边界

OpenAI 官方说明：Codex 除了标准 HTTPS 请求，还会使用 `wss://chatgpt.com/` 建立安全 WebSocket 连接。代理、防火墙或网络过滤设备如果阻止、改写或过早关闭 WebSocket 握手，可能造成停顿、断连或流式输出异常。参见 [OpenAI：Network recommendations for ChatGPT errors on web and apps](https://help.openai.com/en/articles/9247338-network-recommendations-for-chatgpt-errors-on-web-and-apps)。

网络社区中常见的解释是：客户端优先尝试 WebSocket，失败数次后再回退到其他传输方式，因此表现为固定次数的 `Reconnecting`。目前公开官方资料确认了 WebSocket 要求，但没有确认“必定重试 5 次后回退 HTTP”这一完整机制，所以本文不把该细节视为官方结论。

本文的核心是通过 `~/.codex/.env` 为 Codex 指定本机代理，从而避免依赖 TUN 或全局系统代理。

## 第一步：确认正确的代理端口

在代理软件中查找以下任一名称：

- HTTP Port / HTTP 端口
- Mixed Port / 混合端口
- HTTP Proxy Port

不要误用：

- SOCKS-only 端口；
- 控制面板、External Controller 或 REST API 端口；
- DNS 端口；
- TUN 端口。

在 Windows PowerShell 中可以先查看本机监听端口：

```powershell
netstat -ano -p tcp | Select-String "LISTENING"
```

端口必须由代理软件界面、配置文件或本地控制接口进一步确认，不能只凭端口号猜测。

## 第二步：创建 Codex 专用 `.env`

Windows 文件位置：

```text
%USERPROFILE%\.codex\.env
```

macOS/Linux 文件位置：

```text
~/.codex/.env
```

把 `<端口>` 替换成已经确认的 HTTP 或 mixed 端口：

```dotenv
HTTP_PROXY=http://127.0.0.1:<端口>
HTTPS_PROXY=http://127.0.0.1:<端口>
http_proxy=http://127.0.0.1:<端口>
https_proxy=http://127.0.0.1:<端口>
NO_PROXY=localhost,127.0.0.1,::1
```

示例仅用于说明格式：

```dotenv
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
http_proxy=http://127.0.0.1:7890
https_proxy=http://127.0.0.1:7890
NO_PROXY=localhost,127.0.0.1,::1
```

如果 `.env` 已经存在，请先备份，并只更新上述变量，不要覆盖其他配置。

## 第三步：重启并验证

1. 保持代理软件运行。
2. 完全退出 Codex，包括托盘或后台进程。
3. 重新启动 Codex。
4. 新建对话，观察是否还会固定出现多次 `Reconnecting`。

如果仍然重连，请依次检查：

- 端口是否确实是 HTTP/mixed 端口；
- 该端口当前是否仍在监听；
- 当前代理节点是否支持稳定的 WebSocket 长连接；
- `chatgpt.com` 及相关域名是否被代理规则误设为直连；
- `.env` 是否保存到了正确的用户目录；
- Codex 是否已经彻底退出并重启。

## 风险与注意事项

- 这不是 Windows 全局代理设置，主要影响读取该 `.env` 的 Codex 进程。
- 代理软件未运行、端口改变或配置失效时，Codex 可能完全无法连接。
- 更换代理软件或电脑后必须重新确认端口，不能沿用旧值。
- 不要在公开 Issue、截图或仓库中上传订阅地址、节点信息、访问令牌、日志、用户名或真实个人路径。
- Codex 后续版本可能改善相关网络行为；问题消失后可以测试撤销此配置。

## 撤销方法

从 `~/.codex/.env` 中删除以下变量：

```text
HTTP_PROXY
HTTPS_PROXY
http_proxy
https_proxy
NO_PROXY
```

如果文件只包含这些代理变量，可以删除整个 `.env`。随后完全退出并重新启动 Codex。

## 可直接交给新电脑 Codex 的提示词

换电脑或更换代理软件后，可以把下面内容复制到新的 Codex 对话：

```text
我的 Codex 每次新建对话时可能会连续出现多次“Reconnecting... X/5”。

请先只读检查当前正在运行的代理软件、代理内核，以及 127.0.0.1 上的监听端口。找出实际的 HTTP 或 mixed 代理端口，不要把 SOCKS-only、控制面板/API、DNS 或 TUN 端口误认为 HTTP 代理端口。

请检查 ~/.codex/.env 是否存在。如果存在，先显示其中与代理有关的变量并保留其他内容。确认端口后，先告诉我检测结果并等待确认，再添加或更新：

HTTP_PROXY=http://127.0.0.1:<确认后的端口>
HTTPS_PROXY=http://127.0.0.1:<确认后的端口>
http_proxy=http://127.0.0.1:<确认后的端口>
https_proxy=http://127.0.0.1:<确认后的端口>
NO_PROXY=localhost,127.0.0.1,::1

限制：不要启用 TUN，不要修改系统代理或全局环境变量，不要修改其他 Codex 配置，不要猜测或沿用旧电脑端口。修改前备份已有文件，完成后显示最终内容、确认端口仍在监听，并提供重启和撤销方法。如果当前最新版 Codex 已不再复现问题，则不要修改任何配置。
```

## 隐私说明

本文只包含通用回环地址、示例端口和公开文档链接，不包含作者的代理软件配置、节点、订阅、日志、访问凭据、用户名或设备路径。
