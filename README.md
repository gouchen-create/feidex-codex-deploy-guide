# Feidex 飞书远程 Codex 新电脑部署指南

目标：在一台新电脑上部署一套独立的 Feidex + 飞书机器人，让我可以在飞书里远程使用这台电脑上的 Codex。每台电脑对应一个飞书机器人。

本文是给“新电脑上的 Codex”读取并执行的操作文档。默认新电脑只有 Codex，可能没有 Git、Go、Chrome、飞书客户端、Chrome DevTools MCP、Windows MCP 或 Computer Use。Codex 必须先自检能力，能自动完成的步骤不要交给用户；只有登录、扫码、授权、管理后台确认这类 Codex 无法可靠完成的步骤，才让用户操作。

## 0. 执行原则

1. 先检测，再安装；不要假设任何工具已存在。
2. 能用命令行安装和配置的，Codex 自己完成。
3. 需要用户扫码、输入验证码、确认飞书后台权限、选择企业等交互时，暂停并明确告诉用户要做什么。
4. 不要把 Feidex 做成 WezTerm 终端镜像。Feidex 的逻辑是：飞书消息 -> Feidex -> 本机 Codex App Server -> 飞书回复。
5. Feidex 使用本机 `codex` 命令和本机 `C:\Users\<用户名>\.codex` 配置。
6. 每台电脑建议创建独立飞书应用和机器人，名称类似 `Codex-Office-PC`、`Codex-Home-PC`、`Codex-Laptop`。
7. 禁止把 `app_secret`、完整 `config.toml`、访问令牌、GitHub token 提交到仓库或发到公开聊天。需要展示配置时必须脱敏。
8. 如果执行过程中遇到本文未覆盖的问题，并且 Codex 已定位原因并验证了解法，部署完成后必须把问题、报错原文、解决路径和验证命令补回本文档，并提交 PR 或直接提交到本文档仓库。

## 0.1 文档仓库与自进化规则

本文档应托管在 GitHub 仓库中。新电脑部署时，Codex 优先从仓库拉取最新版文档再执行，不要依赖用户手里旧副本。

推荐对新电脑下达的指令：

```text
请从 <本文档仓库地址> 下载 Feidex 飞书远程 Codex 部署指南，并严格按文档完成部署。部署过程中如果遇到文档未覆盖的问题，解决后把经验补回文档并提交 PR。
```

执行回流规则：

1. 部署前先阅读仓库中的 `README.md` 或部署指南全文。
2. 部署中遇到新报错时，先按本文档排障；文档未覆盖时再自行定位。
3. 新问题解决后，在文档的对应章节补充：
   - 触发条件。
   - 报错原文或关键日志。
   - 根因判断。
   - 可复制执行的修复命令。
   - 验证命令和通过标准。
4. 提交前必须扫描文档，确认没有 `app_secret`、token、完整 `config.toml`、私钥等敏感信息。
5. 如果有仓库写权限，直接提交并推送；如果没有写权限，fork 后提交 PR。
6. PR 标题建议使用：

```text
docs: 补充 <电脑/环境/报错关键词> 部署排障经验
```

## 1. 新电脑自检

先运行：

```powershell
$ErrorActionPreference = "Continue"

Write-Host "USERPROFILE=$env:USERPROFILE"
Write-Host "USERNAME=$env:USERNAME"

function Test-Cmd($name) {
  $cmd = Get-Command $name -ErrorAction SilentlyContinue
  if ($cmd) {
    Write-Host "[OK] $name -> $($cmd.Source)"
  } else {
    Write-Host "[MISS] $name"
  }
}

Test-Cmd codex
Test-Cmd git
Test-Cmd go
Test-Cmd node
Test-Cmd npm
Test-Cmd winget
Test-Cmd chrome
Test-Cmd msedge

Test-Path "$env:USERPROFILE\.codex\config.toml"
```

判断结果：

- `codex` 缺失：先让用户安装并登录 Codex，本文后续暂停。
- `git` 缺失：Codex 优先用 `winget` 自动安装 Git。
- `go` 缺失：Codex 优先用 `winget` 自动安装 Go。
- `chrome` 缺失但 `msedge` 存在：可以用 Edge 打开飞书网页测试；如果需要 Chrome DevTools MCP，再安装 Chrome。
- `winget` 缺失：Codex 改用官网下载或提示用户安装 App Installer。

## 2. 检查 Codex 是否可用

执行：

```powershell
codex --version
```

如果失败，停止部署并告诉用户：

```text
当前电脑没有可用 Codex CLI。请先安装 Codex 并完成登录，然后重新让 Codex 读取本文档继续部署。
```

检查 Codex 配置：

```powershell
$CodexConfig = "$env:USERPROFILE\.codex\config.toml"
if (Test-Path $CodexConfig) {
  Get-Content $CodexConfig
} else {
  Write-Host "Codex config not found: $CodexConfig"
}
```

建议远程控制场景使用：

```toml
approval_policy = "never"
sandbox_mode = "danger-full-access"
```

如果用户更重视安全，可以用：

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

但远程飞书使用会更频繁要求确认。

## 3. 检查 Codex 是否具备浏览器或桌面控制能力

本步骤用于判断 Codex 能否自动操作飞书网页和飞书开发者后台。

Codex 先检查当前会话可用工具列表或 MCP 资源：

- 是否有 Chrome DevTools / Playwright / Browser Use 之类浏览器工具。
- 是否有 Windows MCP / Computer Use 之类桌面操作工具。
- 是否能通过命令行启动浏览器并打开 URL。

如果具备浏览器控制能力，Codex 可以尝试自动打开：

```text
https://open.feishu.cn/app
https://www.feishu.cn/messages
```

如果不具备浏览器或桌面控制能力，Codex 不要强行假装能点网页。改为：

1. 用命令行安装依赖和生成配置。
2. 在飞书后台步骤暂停，让用户手动创建应用、扫码登录、复制 `app_id` 和 `app_secret`。
3. 用户提供信息后继续配置和启动 Feidex。

如果需要让后续 Codex 具备网页操作能力，可提示用户安装或启用以下之一：

- Chrome DevTools MCP
- Playwright MCP
- Windows MCP
- Computer Use

这些不是 Feidex 必需项，只是为了让 Codex 能自动操作网页和桌面。缺少这些工具时，部署仍可继续，但飞书后台的创建、登录、发布通常需要用户手动完成。

## 4. 安装基础工具

如果缺少 Git：

```powershell
winget install --id Git.Git -e --source winget
```

如果缺少 Go：

```powershell
winget install --id GoLang.Go -e --source winget
```

如果需要 Chrome 且缺失：

```powershell
winget install --id Google.Chrome -e --source winget
```

安装后重新打开 PowerShell 或刷新 PATH：

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
git --version
go version
```

如果 `winget` 不可用，Codex 应改用官方安装包下载方式，或者告诉用户需要先安装 App Installer / winget。

### 4.1 winget 不可用时安装 Go 的备用方案

如果 `winget` 缺失，但网络可访问 Go 官方下载源，Codex 可以直接下载 Windows MSI 安装包并静默安装。示例：

```powershell
$GoJson = Invoke-RestMethod "https://go.dev/dl/?mode=json"
$GoMsi = $GoJson |
  ForEach-Object { $_.files } |
  Where-Object { $_.os -eq "windows" -and $_.arch -eq "amd64" -and $_.filename -like "*.msi" } |
  Select-Object -First 1

if (!$GoMsi) {
  throw "No Windows amd64 Go MSI found from go.dev"
}

$MsiPath = Join-Path $env:TEMP $GoMsi.filename
Invoke-WebRequest -Uri "https://go.dev/dl/$($GoMsi.filename)" -OutFile $MsiPath
Start-Process msiexec.exe -ArgumentList "/i `"$MsiPath`" /qn /norestart" -Wait

$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
go version
```

如果官方安装包下载失败，Codex 应把失败 URL、HTTP 状态或网络错误摘要告诉用户，让用户手动安装 Go 后继续。

## 5. 创建本机工作目录

```powershell
$BotRoot = "$env:USERPROFILE\feidex-codex-bot"
New-Item -ItemType Directory -Force $BotRoot | Out-Null
Set-Location $BotRoot
```

这个目录是飞书 Codex 的默认工作目录。Codex 生成的文件、下载的附件、日志、状态数据都优先放这里。

创建项目级 Codex 规则文件：

```powershell
$Agents = "$BotRoot\AGENTS.md"
if (!(Test-Path $Agents)) {
@'
# AGENTS.md

## 飞书远程 Codex 工作规则

- 默认在当前 workspace 内创建文件，不要随意写到用户桌面或下载目录。
- 需要生成临时文件时，优先放到当前目录下的 outputs/。
- 回复飞书用户时要简洁说明结果、文件路径、测试情况。
- 修改配置或安装软件前先说明影响。
- 能自动完成的部署步骤由 Codex 自己完成；只有扫码、登录、授权、验证码、后台发布确认这类步骤交给用户。
'@ | Set-Content -Encoding UTF8 $Agents
}
```

修改 `AGENTS.md` 后，飞书里可执行：

```text
/codex restart
```

## 6. 获取并构建 Feidex

```powershell
$SrcRoot = "$env:USERPROFILE\codex-feishu-eval\feidex"
New-Item -ItemType Directory -Force $SrcRoot | Out-Null
Set-Location $SrcRoot

if (!(Test-Path ".\feidex")) {
  git clone https://github.com/yuhuan417/feidex.git
}

Set-Location .\feidex
git pull

$InstallDir = "$env:LOCALAPPDATA\Programs\feidex"
New-Item -ItemType Directory -Force $InstallDir | Out-Null

go build -o "$InstallDir\feidex.exe" .\cmd\feidex
& "$InstallDir\feidex.exe" --help
```

如果普通 `git clone` 失败，且错误类似：

```text
curl 56 schannel: server closed abruptly
early EOF
invalid index-pack output
```

通常是 Windows TLS/HTTP2/大包传输不稳定。先清理未完成目录，再使用 HTTP/1.1 + 浅克隆：

```powershell
$SrcRoot = "$env:USERPROFILE\codex-feishu-eval"
Set-Location $SrcRoot
Remove-Item -Recurse -Force ".\feidex" -ErrorAction SilentlyContinue
git -c http.version=HTTP/1.1 clone --depth 1 --single-branch https://github.com/yuhuan417/feidex.git
```

如果 Go 构建失败，Codex 应先根据错误自动修复常见问题，例如 PATH、依赖下载、网络代理。不能修复时，把错误摘要发给用户。

## 7. 创建或获取飞书机器人凭据

需要一个飞书开放平台应用：

```text
https://open.feishu.cn/app
```

Codex 根据自身能力选择流程。

### 7.1 Codex 有浏览器/桌面控制能力

Codex 可以打开飞书后台，检查用户是否已登录：

```powershell
Start-Process "https://open.feishu.cn/app"
```

如果需要扫码、验证码、企业选择、授权确认，Codex 暂停并提示用户完成。用户完成后继续。

Codex 尝试创建应用并启用机器人能力。建议名称：

```text
Codex-<这台电脑名称>
```

可以用：

```powershell
$env:COMPUTERNAME
```

创建后记录：

```text
app_id
app_secret
```

### 7.2 Codex 没有浏览器/桌面控制能力

Codex 告诉用户手动执行：

1. 打开 `https://open.feishu.cn/app`。
2. 登录飞书开放平台。
3. 创建企业自建应用。
4. 应用名称使用 `Codex-<电脑名>`。
5. 启用机器人能力。
6. 找到并复制 `app_id` 和 `app_secret`。
7. 把 `app_id` 和 `app_secret` 发给 Codex。

Codex 收到凭据后继续。

## 8. 配置飞书后台

在飞书开放平台中配置以下内容。

如果 Codex 有网页控制能力，尽量由 Codex 自动点击配置；遇到扫码、验证码、发布确认时让用户处理。如果没有网页控制能力，用户手动配置。

### 8.1 事件订阅方式

选择：

```text
长连接 / WebSocket / Persistent connection
```

不要用公网 HTTP 回调。

### 8.2 订阅事件

至少订阅：

```text
im.message.receive_v1
card.action.trigger
```

### 8.3 权限 scopes

至少添加：

```text
im:message
im:message:send_as_bot
```

为了支持图片、文件和下载链接，继续添加：

```text
im:resource
drive:drive
```

实测 Feidex 的文件清理、文档检索或下载能力也可能触发以下任一权限要求：

```text
drive:drive
drive:drive:readonly
space:document:retrieve
```

如果飞书返回 `code=99991672` 且提示：

```text
Access denied. One of the following scopes is required: [drive:drive, drive:drive:readonly, space:document:retrieve]
```

打开错误日志中的授权链接，至少开通其中一个满足要求的权限。为了减少后续文件能力缺口，建议同时开通：

```text
drive:drive
drive:drive:readonly
space:document:retrieve
```

如果日志出现 `urgent_app failed`，并提示缺少：

```text
im:message.urgent
im:message.urgent:app_send
```

也在飞书开放平台开通这两个权限。每次改权限后都必须创建并发布新版本。

### 8.4 发布版本

每次改权限、事件、机器人能力、卡片回调后，都必须：

```text
创建版本并发布
```

这是最容易漏的步骤。没有发布版本时，本地 Feidex 可能能连上 WebSocket，但收不到消息或按钮不可用。

## 9. 安装或打开飞书聊天测试入口

可以使用飞书桌面客户端，也可以使用网页版飞书。

Codex 自检：

```powershell
Get-Command Feishu -ErrorAction SilentlyContinue
Get-Command Lark -ErrorAction SilentlyContinue
```

如果没有桌面客户端，可以用网页：

```powershell
Start-Process "https://www.feishu.cn/messages"
```

如果浏览器未登录，Codex 提示用户：

```text
请在打开的飞书网页中扫码或输入验证码登录。登录完成后告诉我继续。
```

如果 Codex 没有浏览器控制能力，用户需要自己打开飞书网页或客户端，后续测试时手动给机器人发消息。

## 10. 创建 Feidex 配置

在 `C:\Users\<用户名>\feidex-codex-bot\config.toml` 写入配置。把 `APP_ID_HERE` 和 `APP_SECRET_HERE` 换成真实值。

Codex 可以用以下 PowerShell 生成：

```powershell
$BotRoot = "$env:USERPROFILE\feidex-codex-bot"
$ConfigPath = "$BotRoot\config.toml"
$AppId = "APP_ID_HERE"
$AppSecret = "APP_SECRET_HERE"

$UserProfileEscaped = $env:USERPROFILE.Replace("\", "\\")
$BotRootEscaped = $BotRoot.Replace("\", "\\")

@"
data_dir = "$BotRootEscaped\\.feidex-data"

[log]
  level = "info"

[feishu]
  backend = "codex"
  codex_auto_retry = false
  app_id = "$AppId"
  app_secret = "$AppSecret"
  domain = "feishu"
  group_at_only = false
  respond_to_at_everyone = false
  card_enabled = true
  reply_in_thread = true
  quiet = "progress"

[codex]
  command = "codex"
  transport = "stdio"
  experimental_api = true
  service_name = "feidex"

[daemon]
  service_name = "feidex"

[[workspace]]
  id = "default"
  name = "Default"
  cwd = "$BotRootEscaped"
  model = ""
  approval_policy = "never"
  sandbox_mode = "danger-full-access"
  multi_agent_mode = "explicitRequestOnly"
"@ | Set-Content -Encoding UTF8 $ConfigPath

Get-Content $ConfigPath
```

如果用户不想全权限，把 workspace 改成：

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

## 11. 启动 Feidex

先停止旧进程。不要泛杀所有 `codex` 进程，因为当前正在执行部署的 Codex 也可能叫 `codex.exe`。如果只是第一次启动，可以停止 `feidex`：

```powershell
Get-Process feidex -ErrorAction SilentlyContinue | Stop-Process
```

启动：

```powershell
$BotRoot = "$env:USERPROFILE\feidex-codex-bot"
$Feidex = "$env:LOCALAPPDATA\Programs\feidex\feidex.exe"

Start-Process -FilePath $Feidex `
  -ArgumentList "--config `"$BotRoot\config.toml`"" `
  -WorkingDirectory $BotRoot `
  -RedirectStandardOutput "$BotRoot\feidex.out.log" `
  -RedirectStandardError "$BotRoot\feidex.err.log" `
  -WindowStyle Hidden
```

检查进程：

```powershell
Get-Process feidex
```

查看日志：

```powershell
Get-Content "$env:USERPROFILE\feidex-codex-bot\feidex.err.log" -Tail 100
```

正常日志应包含类似：

```text
service starting
codex app-server started
codex app-server ready
service started
feishu websocket connected
```

如果没有 `feishu websocket connected`，优先检查：

- `app_id` / `app_secret` 是否正确。
- 飞书后台是否使用长连接。
- 飞书应用是否已发布版本。
- 网络是否能访问飞书开放平台。

### 11.1 安装为 Windows 开机自启 daemon

手动 `Start-Process` 适合首次验证。确认能连上飞书后，应安装 Windows Task Scheduler daemon：

```powershell
$BotRoot = "$env:USERPROFILE\feidex-codex-bot"
$Feidex = "$env:LOCALAPPDATA\Programs\feidex\feidex.exe"

& $Feidex daemon install --config "$BotRoot\config.toml" --force
& $Feidex daemon status --config "$BotRoot\config.toml"
```

正常应显示：

```text
Status: Running
Platform: Task Scheduler (Windows)
Unit: Task Scheduler\feidex
```

查看 daemon 日志：

```powershell
& $Feidex daemon logs -n 120 --config "$BotRoot\config.toml"
```

Windows Task Scheduler 的 `LastTaskResult=267009` 通常表示任务正在运行，不是失败。

### 11.2 权限开通后必须干净重启

如果飞书后台新增了权限、事件或机器人能力，发布版本后必须重启 Feidex：

```powershell
$BotRoot = "$env:USERPROFILE\feidex-codex-bot"
$Feidex = "$env:LOCALAPPDATA\Programs\feidex\feidex.exe"

& $Feidex daemon restart --config "$BotRoot\config.toml"
Start-Sleep -Seconds 8
& $Feidex daemon status --config "$BotRoot\config.toml"
& $Feidex daemon logs -n 120 --config "$BotRoot\config.toml"
```

看“重启后的新日志”，不要只看旧日志。新日志应包含：

```text
service starting
codex app-server ready
service started
feishu websocket connected
```

并且不应再出现：

```text
99991672
Access denied
artifact gc failed
urgent_app failed
```

## 12. 飞书聊天测试

在飞书中找到机器人，发送：

```text
/status
```

重点检查：

```text
workspace: default
cwd: C:\Users\<用户名>\feidex-codex-bot
生效 sandbox: danger-full-access
生效 policy: never
```

然后发送：

```text
你好，请回复你当前运行在哪台电脑、当前工作目录是什么
```

如果能回复，说明链路打通。

如果 Codex 有浏览器控制能力，可以自动打开飞书网页版并观察回复；如果没有，用户手动发消息并把结果告诉 Codex。

## 13. 图片和文件测试

### 图片

用户给机器人发送一张图片，然后发送：

```text
请分析刚才那张图片
```

Feidex 会把图片下载到：

```text
C:\Users\<用户名>\feidex-codex-bot\.feidex-attachments
```

然后作为 `localImage` 传给 Codex。

### 文件

用户发送文件后，再发送：

```text
请读取刚才的文件，并总结内容
```

普通文本、Markdown、代码、CSV 通常没问题。PDF、Word、Excel 取决于本机是否有解析工具，必要时让 Codex 安装或调用工具处理。

视频和音频不是完整自动解析能力，通常只会作为文件路径传给 Codex，不会自动转写。

## 14. 常用指令

```text
/menu
```

打开按钮菜单。

```text
/status
```

查看当前状态。

```text
/new
```

新建 Codex thread。

```text
/threads
```

查看可恢复的 thread。

```text
/stop
```

中断当前任务，并清空排队输入。

```text
/history
```

查看当前 thread 历史。

```text
/download
```

从当前 workspace 选择文件并生成飞书下载链接。

```text
/quiet progress
```

把执行过程折叠成“工作中”卡。

```text
/quiet final
```

只显示最终结果。

```text
/plan on
```

开启计划模式。

```text
/goal <目标>
```

设置长期目标。

```text
/codex restart
```

重启 Codex runtime，适合修改 `AGENTS.md`、安装 skill、修改 Codex 配置后使用。

## 15. 故障排查

### 15.1 Feidex 连接成功，但飞书没回复

检查飞书后台：

- 是否启用机器人能力。
- 是否使用长连接。
- 是否订阅 `im.message.receive_v1`。
- 是否添加 `im:message`、`im:message:send_as_bot`。
- 是否创建并发布了新版本。

### 15.2 按钮提示卡片回调未配置

检查飞书后台是否订阅：

```text
card.action.trigger
```

然后重新发布版本。

### 15.3 图片或文件不工作

检查权限是否包含：

```text
im:resource
drive:drive
drive:drive:readonly
space:document:retrieve
```

然后重新发布版本。

如果 `/status` 能回复，但同时出现红色权限错误卡片，通常说明文本消息链路已通，只是文件/云空间权限不足。按错误中的授权链接开通对应 scope，发布版本，然后按 11.2 干净重启。

### 15.4 总是要求确认权限

在飞书发送：

```text
/status
```

确认生效值是不是：

```text
生效 policy: never
生效 sandbox: danger-full-access
```

如果不是，检查 `config.toml` 里的 workspace：

```toml
approval_policy = "never"
sandbox_mode = "danger-full-access"
```

保存后重启 Feidex。

### 15.5 修改 AGENTS.md 后没生效

执行：

```text
/codex restart
```

或者重启 Feidex。

### 15.6 关闭 Feidex

```powershell
Get-Process feidex -ErrorAction SilentlyContinue | Stop-Process
```

### 15.7 daemon 显示 Stopped 但 Feidex 进程还在

现象：

```text
feidex daemon status
Status: Stopped
LastTaskResult: 1
```

但进程表里仍能看到：

```text
feidex.exe serve --config C:\Users\<用户名>\feidex-codex-bot\config.toml
codex app-server ...
```

这说明 Windows Task Scheduler 包装层和实际服务进程状态不一致，可能残留了孤儿服务树。不要直接 `Stop-Process codex`，否则可能误杀正在执行部署的 Codex。按配置路径精确清理 Feidex 服务树：

```powershell
$BotRoot = "$env:USERPROFILE\feidex-codex-bot"
$Feidex = "$env:LOCALAPPDATA\Programs\feidex\feidex.exe"

& $Feidex daemon stop --config "$BotRoot\config.toml"
Start-Sleep -Seconds 3

$all = @(Get-CimInstance Win32_Process)
$roots = @($all | Where-Object {
  $_.Name -eq "feidex.exe" -and
  $_.CommandLine -like "*feidex-codex-bot\config.toml*"
})

$ids = @()
foreach ($p in $roots) {
  $ids += [int]$p.ProcessId
}

$changed = $true
while ($changed) {
  $changed = $false
  foreach ($p in $all) {
    if (($ids -contains [int]$p.ParentProcessId) -and -not ($ids -contains [int]$p.ProcessId)) {
      $ids += [int]$p.ProcessId
      $changed = $true
    }
  }
}

$ids = @($ids | Sort-Object -Unique)
if ($ids.Count -gt 0) {
  Write-Host "Stopping Feidex service tree: $($ids -join ', ')"
  foreach ($id in ($ids | Sort-Object -Descending)) {
    Stop-Process -Id $id -Force -ErrorAction SilentlyContinue
  }
}

Start-Sleep -Seconds 3
& $Feidex daemon start --config "$BotRoot\config.toml"
Start-Sleep -Seconds 8
& $Feidex daemon status --config "$BotRoot\config.toml"
Get-ScheduledTaskInfo -TaskName "feidex" | Format-List LastRunTime,LastTaskResult
```

注意：PowerShell 中不要依赖 `HashSet[int].ToArray()`，部分环境会报：

```text
Method invocation failed because [System.Int32] does not contain a method named 'ToArray'
```

使用上面的原生数组写法更稳。

## 16. 交付检查清单

部署完成后，必须确认：

- `codex --version` 正常。
- `git --version` 正常。
- `go version` 正常，或已有可用 `feidex.exe`。
- `feidex.exe --help` 正常。
- `config.toml` 中 app_id/app_secret 正确。
- 飞书后台使用长连接。
- 飞书后台已订阅 `im.message.receive_v1`。
- 飞书后台已订阅 `card.action.trigger`。
- 飞书后台已添加必要权限。
- 飞书后台已发布版本。
- 本机日志显示 `codex app-server ready`。
- 本机日志显示 `feishu websocket connected`。
- `feidex daemon status` 显示 `Running`，或手动进程明确在运行。
- 权限开通后的新日志中没有 `99991672`、`Access denied`、`artifact gc failed`、`urgent_app failed`。
- 飞书 `/status` 有回复。
- 飞书普通消息有 Codex 回复。
- 飞书图片测试通过。
- `/download` 能生成文件下载入口。
- 如果部署过程中补充了新排障经验，已更新本文档并提交到文档仓库。

部署时最容易漏的是“飞书后台改完后没有发布版本”。每改一次权限、事件、卡片回调，都要重新发布。
