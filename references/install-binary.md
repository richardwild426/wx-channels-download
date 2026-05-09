# +install-binary(安装原始二进制)

> **场景:** 本机没有 `wx_video_download` 可执行文件,或用户要求安装 / 更新原始 `wx_channels_download` release 包。
>
> **前置:** 已安装 `gh`;macOS / Linux 需要 `tar` 或 `unzip`;Windows 走 PowerShell `Expand-Archive`。只从上游 release 下载。

## macOS / Linux

默认安装到 `$HOME/.local/opt/wx_video_download`,并把可执行文件链接到 `$HOME/.local/bin/wx_video_download`。用 `WX_VERSION` 固定版本;不设则安装 GitHub latest。

```bash
#!/usr/bin/env bash
set -euo pipefail

INSTALL_DIR="${WX_BINARY_DIR:-$HOME/.local/opt/wx_video_download}"
BIN_DIR="${WX_BIN_DIR:-$HOME/.local/bin}"
TAG="${WX_VERSION:-$(gh release view --repo ltaoo/wx_channels_download --json tagName --jq .tagName)}"

case "$INSTALL_DIR" in
  ""|"/"|"$HOME"|"$HOME/") echo "unsafe install dir: $INSTALL_DIR" >&2; exit 1 ;;
esac

case "$(uname -s)" in
  Darwin) OS="darwin"; EXT="zip" ;;
  Linux) OS="linux"; EXT="tar.gz" ;;
  *) echo "unsupported OS: $(uname -s)" >&2; exit 1 ;;
esac

case "$(uname -m)" in
  arm64|aarch64) ARCH="arm64" ;;
  x86_64|amd64) ARCH="x86_64" ;;
  i386|i686) ARCH="x86" ;;
  *) echo "unsupported arch: $(uname -m)" >&2; exit 1 ;;
esac

ASSET="wx_video_download_${TAG}_${OS}_${ARCH}.${EXT}"
TMP="$(mktemp -d)"
trap 'rm -rf "$TMP"' EXIT

gh release download "$TAG" --repo ltaoo/wx_channels_download \
  --pattern "$ASSET" \
  --pattern "wx_video_download_${TAG}_checksums.txt" \
  --dir "$TMP"

cd "$TMP"
if command -v sha256sum >/dev/null 2>&1; then
  grep "  $ASSET$" "wx_video_download_${TAG}_checksums.txt" | sha256sum -c -
else
  grep "  $ASSET$" "wx_video_download_${TAG}_checksums.txt" | shasum -a 256 -c -
fi

rm -rf "$INSTALL_DIR"
mkdir -p "$INSTALL_DIR" "$BIN_DIR"
case "$EXT" in
  zip) unzip -q "$ASSET" -d "$INSTALL_DIR" ;;
  tar.gz) tar -xzf "$ASSET" -C "$INSTALL_DIR" ;;
esac

chmod +x "$INSTALL_DIR/wx_video_download"
ln -sfn "$INSTALL_DIR/wx_video_download" "$BIN_DIR/wx_video_download"
test -x "$BIN_DIR/wx_video_download"
echo "installed: $INSTALL_DIR"
```

## Windows PowerShell

默认安装到 `%LOCALAPPDATA%\Programs\wx_video_download`。`WX_VERSION` 可固定版本;不设则安装 latest。普通包和 `safe` 包都来自同一 release;默认用普通包,只有用户明确要求安全包时才把 `$Safe = $true`。

```powershell
$ErrorActionPreference = "Stop"

$InstallDir = if ($env:WX_BINARY_DIR) { $env:WX_BINARY_DIR } else { Join-Path $env:LOCALAPPDATA "Programs\wx_video_download" }
$Tag = if ($env:WX_VERSION) { $env:WX_VERSION } else { gh release view --repo ltaoo/wx_channels_download --json tagName --jq .tagName }
if ([string]::IsNullOrWhiteSpace($InstallDir) -or $InstallDir -eq $env:USERPROFILE -or $InstallDir -eq "$env:SystemDrive\") {
  throw "unsafe install dir: $InstallDir"
}
$Arch = switch ($env:PROCESSOR_ARCHITECTURE) {
  "ARM64" { "arm64" }
  "AMD64" { "x86_64" }
  default { throw "unsupported arch: $env:PROCESSOR_ARCHITECTURE" }
}
$Safe = $false
$Prefix = if ($Safe) { "wx_video_download_safe" } else { "wx_video_download" }
$Asset = "${Prefix}_${Tag}_windows_${Arch}.zip"
$Checksums = "wx_video_download_${Tag}_checksums.txt"
$Tmp = New-Item -ItemType Directory -Path ([System.IO.Path]::GetTempPath()) -Name ("wx-video-download-" + [guid]::NewGuid())

gh release download $Tag --repo ltaoo/wx_channels_download --pattern $Asset --pattern $Checksums --dir $Tmp.FullName
$Expected = (Select-String -Path (Join-Path $Tmp.FullName $Checksums) -Pattern "  $([regex]::Escape($Asset))$").Line.Split(" ")[0]
$Actual = (Get-FileHash -Algorithm SHA256 (Join-Path $Tmp.FullName $Asset)).Hash.ToLower()
if ($Expected -ne $Actual) { throw "checksum mismatch: $Asset" }

Remove-Item -Recurse -Force $InstallDir -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Force -Path $InstallDir | Out-Null
Expand-Archive -Force (Join-Path $Tmp.FullName $Asset) $InstallDir
Remove-Item -Recurse -Force $Tmp.FullName

if (!(Test-Path (Join-Path $InstallDir "wx_video_download.exe"))) { throw "missing installed executable" }
Write-Host "installed: $InstallDir"
```

## 验证

```bash
command -v wx_video_download
test -x "$(command -v wx_video_download)"
```

安装只负责落地原始 release 包。服务是否已监听、微信 PC 是否已接通,继续走 [`run-binary.md`](run-binary.md) 和 [`precondition-probe.md`](precondition-probe.md)。
