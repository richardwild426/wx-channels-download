# +run-binary(自动启动 / 连接原始二进制)

> **场景:** 用户通过对话要求搜索作者、列作品、单条下载或批量下载,但本地 `wx_video_download` 服务未就绪。
>
> **原则:** 智能体负责安装、启动、探测、重连和调用 API。不要让用户执行 CLI。只有微信登录、系统证书 / 代理授权、打开或刷新视频号页面这些 GUI 动作需要用户处理。

## 会话入口

用户说"搜视频号作者"、"列作品"、"下载这个视频号"、"批量下载某作者"时,按这个顺序自动执行:

1. 跑 [`precondition-probe.md`](precondition-probe.md)。
2. 如果二进制缺失,先跑 [`install-binary.md`](install-binary.md)。
3. 如果 API 不可达,按本文启动原始二进制服务。
4. 如果 API 可达但 `channels.available == false`,提示用户完成 GUI 动作:登录微信 PC,同意证书 / 代理授权,打开或刷新任意视频号页面。
5. probe 通过后继续用户原始任务:搜索作者、列作品、下载或批量下载。

## macOS: 用 launchctl 托管

不要用 `nohup ... &`。在 Codex / agent 执行环境中,后台子进程可能在命令返回后被清理。macOS 使用用户级 LaunchAgent 托管进程。

```bash
#!/usr/bin/env bash
set -euo pipefail

INSTALL_DIR="${WX_BINARY_DIR:-$HOME/.local/opt/wx_video_download}"
BIN="$INSTALL_DIR/wx_video_download"
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"
LABEL="com.wx-channels-download.agent"
PLIST="$HOME/Library/LaunchAgents/$LABEL.plist"
LOG="$INSTALL_DIR/wx_video_download.launchd.log"
ERR="$INSTALL_DIR/wx_video_download.launchd.err.log"

if curl -fsS --max-time 2 "$SERVER/api/status" >/dev/null 2>&1; then
  echo "service already reachable: $SERVER"
  exit 0
fi

if [ ! -x "$BIN" ]; then
  echo "missing binary: $BIN" >&2
  echo "run +install-binary first" >&2
  exit 1
fi

mkdir -p "$HOME/Library/LaunchAgents" "$INSTALL_DIR"
cat > "$PLIST" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>$LABEL</string>
  <key>ProgramArguments</key>
  <array>
    <string>$BIN</string>
  </array>
  <key>WorkingDirectory</key>
  <string>$INSTALL_DIR</string>
  <key>StandardOutPath</key>
  <string>$LOG</string>
  <key>StandardErrorPath</key>
  <string>$ERR</string>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
EOF

UID_VALUE="$(id -u)"
launchctl bootout "gui/$UID_VALUE/$LABEL" >/dev/null 2>&1 || true
launchctl bootstrap "gui/$UID_VALUE" "$PLIST"
launchctl kickstart -k "gui/$UID_VALUE/$LABEL"

for _ in 1 2 3 4 5 6 7 8 9 10; do
  if curl -fsS --max-time 2 "$SERVER/api/status" >/dev/null 2>&1; then
    echo "service started: $SERVER"
    exit 0
  fi
  sleep 1
done

echo "service process started but API is not reachable yet; log: $LOG; err: $ERR" >&2
exit 2
```

停止服务时由智能体执行:

```bash
LABEL="com.wx-channels-download.agent"
UID_VALUE="$(id -u)"
launchctl bootout "gui/$UID_VALUE/$LABEL" >/dev/null 2>&1 || true
```

## Windows: 用 PowerShell 启动

Windows 上可能需要管理员权限弹窗。智能体可以发起启动;用户只处理权限弹窗。

```powershell
$ErrorActionPreference = "Stop"
$InstallDir = if ($env:WX_BINARY_DIR) { $env:WX_BINARY_DIR } else { Join-Path $env:LOCALAPPDATA "Programs\wx_video_download" }
$Server = if ($env:WX_SERVER) { $env:WX_SERVER } else { "http://127.0.0.1:2022" }
$Exe = Join-Path $InstallDir "wx_video_download.exe"

try {
  Invoke-RestMethod -Uri "$Server/api/status" -TimeoutSec 2 | Out-Null
  Write-Host "service already reachable: $Server"
  exit 0
} catch {}

if (!(Test-Path $Exe)) { throw "missing binary: $Exe; run +install-binary first" }

Start-Process -FilePath $Exe -WorkingDirectory $InstallDir

for ($i = 0; $i -lt 10; $i++) {
  try {
    Invoke-RestMethod -Uri "$Server/api/status" -TimeoutSec 2 | Out-Null
    Write-Host "service started: $Server"
    exit 0
  } catch {
    Start-Sleep -Seconds 1
  }
}

throw "service process started but API is not reachable yet"
```

## Linux: 用 systemd 用户服务

Linux 用户环境优先使用 systemd user service。无 systemd 的环境不能可靠托管长期进程;此时只提示用户需要保留程序窗口,不要输出 CLI 命令。

```bash
#!/usr/bin/env bash
set -euo pipefail

INSTALL_DIR="${WX_BINARY_DIR:-$HOME/.local/opt/wx_video_download}"
BIN="$INSTALL_DIR/wx_video_download"
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"
UNIT_DIR="$HOME/.config/systemd/user"
UNIT="$UNIT_DIR/wx-video-download.service"

if curl -fsS --max-time 2 "$SERVER/api/status" >/dev/null 2>&1; then
  echo "service already reachable: $SERVER"
  exit 0
fi

if [ ! -x "$BIN" ]; then
  echo "missing binary: $BIN" >&2
  echo "run +install-binary first" >&2
  exit 1
fi

mkdir -p "$UNIT_DIR"
cat > "$UNIT" <<EOF
[Unit]
Description=wx_video_download

[Service]
WorkingDirectory=$INSTALL_DIR
ExecStart=$BIN
Restart=no

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user restart wx-video-download.service

for _ in 1 2 3 4 5 6 7 8 9 10; do
  if curl -fsS --max-time 2 "$SERVER/api/status" >/dev/null 2>&1; then
    echo "service started: $SERVER"
    exit 0
  fi
  sleep 1
done

echo "service process started but API is not reachable yet" >&2
exit 2
```

## GUI 介入点

如果服务可达但 `channels.available == false`,不要把 curl 命令给用户。给用户一句明确动作:

> 我已经启动下载器服务。现在需要你在微信 PC 登录账号,按系统提示允许证书 / 代理授权,然后打开或刷新任意视频号页面。完成后告诉我"好了",我会继续搜索 / 下载。

在 macOS 上,智能体先主动打开微信和本地代理页,降低用户寻找入口的成本:

```bash
open -a WeChat >/dev/null 2>&1 || open -a 微信 >/dev/null 2>&1 || true
open "http://127.0.0.1:2023" >/dev/null 2>&1 || true
```

用户回复后由智能体轮询等待,不要让用户执行验证命令:

```bash
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"
for _ in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30; do
  if curl -fsS --max-time 2 "$SERVER/api/status" \
    | jq -e '.code == 0 and .data.channels.available == true' >/dev/null; then
    echo "channels ready"
    exit 0
  fi
  sleep 2
done
echo "channels still unavailable" >&2
exit 3
```

probe 通过后继续原始任务。

## 成功标准

```bash
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"
curl -fsS "$SERVER/api/status" | jq -e '.code == 0 and .data.channels.available == true'
```

只要这个检查通过,后续的作者搜索、作品列表、单条下载、批量下载都由智能体继续执行。
