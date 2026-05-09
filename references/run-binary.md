# +run-binary(运行原始二进制)

> **场景:** 已安装 `wx_video_download`,需要启动 / 停止 / 重启本地 HTTP 服务,或 probe 报 connection refused。
>
> **前置:** 已按 [`install-binary.md`](install-binary.md) 安装;微信 PC 客户端已登录。macOS / Linux 默认通过 `nohup` 后台运行;Windows 默认用 PowerShell `Start-Process`。

## macOS / Linux 启动

必须在安装目录运行,因为 release 包内的 `config.yaml` 与二进制同级。默认安装目录是 `$HOME/.local/opt/wx_video_download`。

```bash
#!/usr/bin/env bash
set -euo pipefail

INSTALL_DIR="${WX_BINARY_DIR:-$HOME/.local/opt/wx_video_download}"
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"
LOG="${WX_LOG:-$INSTALL_DIR/wx_video_download.log}"
PID_FILE="${WX_PID_FILE:-$INSTALL_DIR/wx_video_download.pid}"

if curl -fsS --max-time 2 "$SERVER/api/status" >/dev/null 2>&1; then
  echo "already running: $SERVER"
  exit 0
fi

if [ ! -x "$INSTALL_DIR/wx_video_download" ]; then
  echo "missing binary: $INSTALL_DIR/wx_video_download" >&2
  echo "run +install-binary first" >&2
  exit 1
fi

cd "$INSTALL_DIR"
nohup "$INSTALL_DIR/wx_video_download" >"$LOG" 2>&1 &
echo $! > "$PID_FILE"

for _ in $(seq 1 20); do
  if curl -fsS --max-time 2 "$SERVER/api/status" >/dev/null 2>&1; then
    echo "started: $SERVER"
    exit 0
  fi
  sleep 1
done

echo "started process but API is not ready; log: $LOG" >&2
exit 2
```

启动后继续跑 [`precondition-probe.md`](precondition-probe.md)。如果 probe 返回 `channels.available == false`,服务已经起来,问题在微信登录、证书或代理链路,不是安装问题。若系统要求管理员权限才能安装证书或接管代理,保留当前安装目录,改用管理员终端在同一目录启动原始二进制。

## macOS / Linux 停止

```bash
INSTALL_DIR="${WX_BINARY_DIR:-$HOME/.local/opt/wx_video_download}"
PID_FILE="${WX_PID_FILE:-$INSTALL_DIR/wx_video_download.pid}"

if [ -f "$PID_FILE" ] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
  kill "$(cat "$PID_FILE")"
  rm -f "$PID_FILE"
  echo "stopped"
else
  pgrep -f "wx_video_download" >/dev/null && pkill -f "wx_video_download" || true
  echo "no pid file process"
fi
```

## Windows PowerShell 启动

Windows 上通常需要以管理员权限运行,否则证书 / 代理相关能力可能无法生效。

```powershell
$ErrorActionPreference = "Stop"

$InstallDir = if ($env:WX_BINARY_DIR) { $env:WX_BINARY_DIR } else { Join-Path $env:LOCALAPPDATA "Programs\wx_video_download" }
$Server = if ($env:WX_SERVER) { $env:WX_SERVER } else { "http://127.0.0.1:2022" }
$Exe = Join-Path $InstallDir "wx_video_download.exe"
$Log = Join-Path $InstallDir "wx_video_download.log"
$ErrLog = Join-Path $InstallDir "wx_video_download.err.log"

try {
  Invoke-RestMethod -Uri "$Server/api/status" -TimeoutSec 2 | Out-Null
  Write-Host "already running: $Server"
  exit 0
} catch {}

if (!(Test-Path $Exe)) { throw "missing binary: $Exe; run +install-binary first" }

Start-Process -FilePath $Exe -WorkingDirectory $InstallDir -RedirectStandardOutput $Log -RedirectStandardError $ErrLog

for ($i = 0; $i -lt 20; $i++) {
  try {
    Invoke-RestMethod -Uri "$Server/api/status" -TimeoutSec 2 | Out-Null
    Write-Host "started: $Server"
    exit 0
  } catch {
    Start-Sleep -Seconds 1
  }
}

throw "started process but API is not ready; log: $Log; err: $ErrLog"
```

## Windows PowerShell 停止

```powershell
Get-Process wx_video_download -ErrorAction SilentlyContinue | Stop-Process
```

## 运行后检查

```bash
curl -fsS "${WX_SERVER:-http://127.0.0.1:2022}/api/status" | jq .
```

`code == 0` 只表示服务自身可访问。下载前仍要通过 SKILL.md §1 的 `channels.available == true` 检查。
