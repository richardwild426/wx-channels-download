# +update-skill(从 GitHub 更新本 skill)

> **场景:** 用户要求"更新 wx-channels-download skill"、"刷新技能"、"同步最新技能版本"。
>
> **硬规则:** 始终从 GitHub 仓库 `richardwild426/wx-channels-download` 更新。不要从当前工作区、临时目录、软链接目标或本地拷贝同步,因为这些位置可能是开发态、脏工作区或旧拷贝。

## macOS / Linux

把 GitHub 最新 `main` 克隆到临时目录,再交给当前智能体自己的 skill 安装机制决定目标目录。不要把安装位置写死到统一目录,也不要同步多个运行时目录。更新时直接替换目标目录,不保留旧副本。

如果当前智能体没有可调用的 skill 安装命令,先由智能体解析本轮运行时实际使用的 `wx-channels-download` skill 目录,再把该目录作为 `WX_SKILL_INSTALL_DIR` 执行下面的替换片段。

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO="${WX_SKILL_REPO:-richardwild426/wx-channels-download}"
SKILL_NAME="wx-channels-download"
TARGET="${WX_SKILL_INSTALL_DIR:?set WX_SKILL_INSTALL_DIR to the skill directory resolved by this agent runtime}"
TMP="$(mktemp -d)"
trap 'rm -rf "$TMP"' EXIT

gh repo clone "$REPO" "$TMP/src" -- --depth 1
rm -rf "$TMP/src/.git"

case "$TARGET" in
  ""|"/"|"$HOME"|"$HOME/"|"/Users"|"/Users/"|"/tmp"|"/tmp/") echo "unsafe target: $TARGET" >&2; exit 1 ;;
esac
[ "$(basename "$TARGET")" = "$SKILL_NAME" ] || { echo "target basename must be $SKILL_NAME: $TARGET" >&2; exit 1; }

parent="$(dirname "$TARGET")"
staging="$parent/.$SKILL_NAME.tmp"
rm -rf "$staging"
mkdir -p "$staging"
(cd "$TMP/src" && tar -cf - .) | (cd "$staging" && tar -xf -)
rm -rf "$TARGET"
mv "$staging" "$TARGET"
echo "updated: $TARGET"
```

更新完成后,当前会话可能仍持有旧 skill 上下文。用户再次使用前,让用户开启新会话或刷新技能上下文;不要把本地仓库路径当作更新结果。

## Windows PowerShell

```powershell
$ErrorActionPreference = "Stop"

$Repo = if ($env:WX_SKILL_REPO) { $env:WX_SKILL_REPO } else { "richardwild426/wx-channels-download" }
$SkillName = "wx-channels-download"
if (!$env:WX_SKILL_INSTALL_DIR) { throw "set WX_SKILL_INSTALL_DIR to the skill directory resolved by this agent runtime" }
$Target = $env:WX_SKILL_INSTALL_DIR
$Tmp = Join-Path ([System.IO.Path]::GetTempPath()) ("wx-skill-" + [guid]::NewGuid())
New-Item -ItemType Directory -Force -Path $Tmp | Out-Null

gh repo clone $Repo (Join-Path $Tmp "src") -- --depth 1
Remove-Item -Recurse -Force (Join-Path $Tmp "src\.git") -ErrorAction SilentlyContinue

if ([System.IO.Path]::GetFileName($Target) -ne $SkillName) { throw "target basename must be $SkillName: $Target" }
$Parent = Split-Path -Parent $Target
$Staging = Join-Path $Parent ".$SkillName.tmp"
Remove-Item -Recurse -Force $Staging -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Force -Path $Staging | Out-Null
Copy-Item -Recurse -Force (Join-Path $Tmp "src\*") $Staging
Remove-Item -Recurse -Force $Target -ErrorAction SilentlyContinue
Move-Item -Force $Staging $Target
Write-Host "updated: $Target"
Remove-Item -Recurse -Force $Tmp
```

## 更新后验证

```bash
grep -n 'skill_repository: https://github.com/richardwild426/wx-channels-download' \
  "$WX_SKILL_INSTALL_DIR/SKILL.md"
```

若当前对话已经加载旧 skill,更新文件不会 retroactively 改变本轮上下文。完成更新后,开启新会话再使用最新 skill。
