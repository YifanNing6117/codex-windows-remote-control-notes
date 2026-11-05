# Windows 上的 Codex App 提示“无法启用远程控制”怎么办？

2026 年 11 月 5 日，我在 Windows 版 Codex App（`26.1027.6604.0`）打开「设置 → 连接 → 控制此电脑」，点击「允许」后，应用一直提示“无法启用远程控制。请重试”。系统代理当时已经开启，更新应用后仍能复现。

**我最终看到了手机配对二维码。** 当时一起做了两项改动：设置 Windows 用户级代理环境变量，然后重启电脑。这次记录到二维码出现为止，手机扫码授权和从手机发起任务仍待验证。

## 从日志找到超时的位置

我先用浏览器以外的工具测试了普通 HTTPS 请求，确认通过本机代理能访问 ChatGPT。随后在 PowerShell 中临时设置代理变量，再从 Windows 应用入口打开 Codex，错误依旧。这个结果留下了一个问题：应用启动的后台执行器实际能否连到它使用的远程服务？

这个版本把文本日志放在应用包的 `LocalCache` 下。复现一次错误、记下时间，再用 PowerShell 搜索近期日志：

```powershell
$logs = Join-Path $env:LOCALAPPDATA 'Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs'
Get-ChildItem -LiteralPath $logs -Recurse -File -Filter '*.log' |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 |
    Select-String -Pattern 'rendezvous|websocket_error|timed_out'
```

在报错时段，我找到了 `Local Work executor rendezvous unavailable`、`websocket_error` 和 `io_error_kind="timed_out"`。这里的 rendezvous 指执行器尝试连接的协调服务。日志把排查范围缩小到了建立 WebSocket 连接的阶段。

## 对同一目标做两次请求

我从后台执行器的启动参数中找到了它访问的服务地址，然后对同一个地址发送两次 HTTPS HEAD 请求。下面保留当时的服务地址，本机代理地址由读者填写。

```powershell
$target = 'https://codex-cloud-environments.chatgpt.com/api'
$proxy = Read-Host '输入本机 HTTP 代理地址（含协议和端口）'
curl.exe --head --noproxy localhost --proxy $proxy --max-time 15 $target
curl.exe --head --noproxy "*" --connect-timeout 5 --max-time 10 $target
```

| 请求路径 | 当时的结果 | 可以确认的事实 |
| --- | --- | --- |
| 显式指定代理 | 建立隧道后，服务返回 HTTP 404 | 这条路径收到了目标服务的 HTTP 响应 |
| 显式直连 | 约 5 秒后连接超时 | 直连在测试时限内未能建立连接 |

这组结果让我优先检查执行器能否获得代理配置。HEAD 请求只能测试 HTTPS 可达性。原记录没有捕获执行器的实际流量，因此它先前是否直连、WebSocket 握手具体在哪一步超时，都只能留作待查。

## 我改了什么，以及怎样撤回

我将同一个本机 HTTP 代理地址写入 Windows 当前用户的 `HTTP_PROXY`、`HTTPS_PROXY` 和 `ALL_PROXY`，然后重启 Windows。确认本机代理运行后，我再次打开 Codex 并点击「允许」。这次开关成功启用，页面显示“在你的设备上批准以远程控制这台电脑”和配对二维码。三个变量一起设置，重启也在同一次尝试中完成，无法单独判断哪一步起了作用。

如果你也测得“代理有响应、直连超时”，并打算复试，请先在 Windows 的「编辑帐户的环境变量」中逐项记录这三个**用户变量**的原值，标明哪些原本不存在，并在本地保存。随后在同一窗口把三个变量设为你自己的代理地址。用户变量也可能影响同一账户下读取它们的其他程序。需要撤回时，把原有变量改回记录值，删除本次新增的变量，然后重新登录 Windows。

复测时可分别记录三个结果：电脑显示二维码、手机完成授权、手机发起的任务在电脑上执行并返回结果。若第一步仍失败，保留本次尝试的时间，对照新日志查看同一类超时是否再次出现。分享时只需摘录相关错误字段，配对二维码和完整日志请留在本机。

## 许可

本文以 [知识共享署名 4.0 国际许可协议（CC BY 4.0）](https://creativecommons.org/licenses/by/4.0/) 发布。
