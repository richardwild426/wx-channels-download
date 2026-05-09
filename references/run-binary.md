# +run-binary(连接 / 运行原始二进制)

> **场景:** 已安装 `wx_video_download`,需要让 skill 连接到用户手动验证正常的实例,或需要说明如何人工启动本地 HTTP + 代理服务。
>
> **前置:** 已按 [`install-binary.md`](install-binary.md) 安装;微信 PC 客户端已登录。

## 结论先行

不要让 agent 在非交互 shell 里用 `nohup ... &` 为一次操作临时启动下载器。原因有两个:

- Codex / agent 执行环境可能在命令返回后清理后台子进程,导致服务看起来启动过但随即消失。
- 上游 `/api/status` 的 `channels.available` 取决于是否有视频号页面通过 WebSocket 连到这个实例。新启动的实例通常没有这个连接,即使 API 和代理端口都启动成功也不能搜索 / 下载。

正确流程是:连接用户已经手动验证 `channels.available == true` 的实例。

```bash
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"
curl -fsS "$SERVER/api/status" | jq -e '.code == 0 and .data.channels.available == true'
```

如果用户手动正常的程序不在默认端口,设置真实地址:

```bash
PORT="2022"
export WX_SERVER="http://127.0.0.1:$PORT"
```

如果用户手动正常的程序在另一份目录,记录真实目录,后续排错用同一份配置:

```bash
export WX_BINARY_DIR=/path/to/working/wx_video_download_dir
```

## macOS / Linux 人工前台启动

必须在安装目录运行,因为 release 包内的 `config.yaml` 与二进制同级。默认安装目录是 `$HOME/.local/opt/wx_video_download`。该命令应由用户在自己的终端前台运行,保持窗口打开。

```bash
INSTALL_DIR="${WX_BINARY_DIR:-$HOME/.local/opt/wx_video_download}"
cd "$INSTALL_DIR"
./wx_video_download
```

启动后在另一个终端验证:

```bash
curl -fsS "${WX_SERVER:-http://127.0.0.1:2022}/api/status" | jq .
```

必须看到 `.data.channels.available == true` 后,skill 才能调用搜索 / 下载 API。

## macOS / Linux 停止

前台运行时用 `Ctrl+C` 停止。不要用 agent 随意 `pkill`,避免误杀用户正在验证的实例。

## Windows PowerShell 人工前台启动

Windows 上通常需要以管理员权限运行,否则证书 / 代理相关能力可能无法生效。

```powershell
$InstallDir = if ($env:WX_BINARY_DIR) { $env:WX_BINARY_DIR } else { Join-Path $env:LOCALAPPDATA "Programs\wx_video_download" }
Set-Location $InstallDir
.\wx_video_download.exe
```

## 实例不一致的典型现象

### 1. skill 看到 connection refused

```bash
curl -fsS http://127.0.0.1:2022/api/status
# curl: (7) Failed to connect
```

说明 skill 没连到用户手动启动的那个 API。确认手动实例端口,并设置 `WX_SERVER`。

### 2. skill 启动的实例 API 可达但 `available == false`

```json
{
  "code": 0,
  "data": {
    "channels": {
      "available": false
    }
  }
}
```

说明 API 进程存在,但没有视频号前端 WebSocket 客户端连接。常见原因是 skill 另起了一份安装目录 / 配置 / 代理实例,微信 PC 的视频号页面仍连着用户手动验证的旧实例,或没有重新经过当前代理打开。

## 运行后检查

```bash
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"
curl -fsS "$SERVER/api/status" | jq .
```

成功标准:

```json
{
  "code": 0,
  "data": {
    "channels": {
      "available": true
    }
  }
}
```

`code == 0` 只表示服务自身可访问。下载前仍要通过 `channels.available == true` 检查。
