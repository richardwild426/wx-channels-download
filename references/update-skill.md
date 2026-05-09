# +update-skill(从 GitHub 更新本 skill)

> **场景:** 用户要求"更新 wx-channels-download skill"、"刷新技能"、"同步最新技能版本"。
>
> **硬规则:** 始终从 GitHub 仓库 `richardwild426/wx-channels-download` 更新。不要从当前工作区、临时目录、软链接目标或本地拷贝同步,因为这些位置可能是开发态、脏工作区或旧拷贝。

## macOS / Linux

把 GitHub 最新 `main` 克隆到临时目录,再原子替换 skill 目录。默认只安装到 `$AGENTS_HOME/skills`(`AGENTS_HOME` 未设时为 `$HOME/.agents`)。不要默认写入 `.codex/skills` 或 `.claude/skills`;如果这些专用目录已有旧副本,默认移到同级备份目录,否则 Codex / Claude 会继续把专用目录当成安装目标。

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO="${WX_SKILL_REPO:-richardwild426/wx-channels-download}"
SKILL_NAME="wx-channels-download"
AGENTS_HOME="${AGENTS_HOME:-$HOME/.agents}"
TMP="$(mktemp -d)"
trap 'rm -rf "$TMP"' EXIT

gh repo clone "$REPO" "$TMP/src" -- --depth 1
rm -rf "$TMP/src/.git"

install_one() {
  parent="$1"
  mode="${2:-optional}"
  if [ "$mode" = "required" ]; then
    mkdir -p "$parent"
  elif [ ! -d "$parent" ]; then
    return 0
  fi
  target="$parent/$SKILL_NAME"
  staging="$parent/.$SKILL_NAME.tmp"
  backup="$parent/.$SKILL_NAME.backup.$(date +%Y%m%d%H%M%S)"

  rm -rf "$staging"
  mkdir -p "$staging"
  (cd "$TMP/src" && tar -cf - .) | (cd "$staging" && tar -xf -)

  if [ -e "$target" ] || [ -L "$target" ]; then
    mv "$target" "$backup"
  fi
  mv "$staging" "$target"
  echo "updated: $target"
}

retire_legacy_one() {
  parent="$1"
  [ -d "$parent" ] || return 0
  target="$parent/$SKILL_NAME"
  [ -e "$target" ] || [ -L "$target" ] || return 0
  backup="$parent/.$SKILL_NAME.legacy.$(date +%Y%m%d%H%M%S)"
  mv "$target" "$backup"
  echo "retired legacy: $target -> $backup"
}

install_one "$AGENTS_HOME/skills" required
if [ "${WX_SYNC_RUNTIME_SKILLS:-0}" = "1" ]; then
  install_one "$HOME/.codex/skills"
  install_one "$HOME/.claude/skills"
else
  retire_legacy_one "$HOME/.codex/skills"
  retire_legacy_one "$HOME/.claude/skills"
fi
```

更新完成后,当前会话可能仍持有旧 skill 上下文。用户再次使用前,让用户开启新会话或刷新技能上下文;不要把本地仓库路径当作更新结果。

## Windows PowerShell

```powershell
$ErrorActionPreference = "Stop"

$Repo = if ($env:WX_SKILL_REPO) { $env:WX_SKILL_REPO } else { "richardwild426/wx-channels-download" }
$SkillName = "wx-channels-download"
$AgentsHome = if ($env:AGENTS_HOME) { $env:AGENTS_HOME } else { Join-Path $env:USERPROFILE ".agents" }
$Tmp = Join-Path ([System.IO.Path]::GetTempPath()) ("wx-skill-" + [guid]::NewGuid())
New-Item -ItemType Directory -Force -Path $Tmp | Out-Null

gh repo clone $Repo (Join-Path $Tmp "src") -- --depth 1
Remove-Item -Recurse -Force (Join-Path $Tmp "src\.git") -ErrorAction SilentlyContinue

function Install-One($Parent, [bool]$Required = $false) {
  if ($Required) {
    New-Item -ItemType Directory -Force -Path $Parent | Out-Null
  } elseif (!(Test-Path $Parent)) {
    return
  }
  $Target = Join-Path $Parent $SkillName
  $Staging = Join-Path $Parent ".$SkillName.tmp"
  $Backup = Join-Path $Parent ".$SkillName.backup.$(Get-Date -Format yyyyMMddHHmmss)"

  Remove-Item -Recurse -Force $Staging -ErrorAction SilentlyContinue
  New-Item -ItemType Directory -Force -Path $Staging | Out-Null
  Copy-Item -Recurse -Force (Join-Path $Tmp "src\*") $Staging

  if (Test-Path $Target) {
    Move-Item -Force $Target $Backup
  }
  Move-Item -Force $Staging $Target
  Write-Host "updated: $Target"
}

function Retire-LegacyOne($Parent) {
  if (!(Test-Path $Parent)) { return }
  $Target = Join-Path $Parent $SkillName
  if (!(Test-Path $Target)) { return }
  $Backup = Join-Path $Parent ".$SkillName.legacy.$(Get-Date -Format yyyyMMddHHmmss)"
  Move-Item -Force $Target $Backup
  Write-Host "retired legacy: $Target -> $Backup"
}

Install-One (Join-Path $AgentsHome "skills") $true
if ($env:WX_SYNC_RUNTIME_SKILLS -eq "1") {
  Install-One (Join-Path $env:USERPROFILE ".codex\skills")
  Install-One (Join-Path $env:USERPROFILE ".claude\skills")
} else {
  Retire-LegacyOne (Join-Path $env:USERPROFILE ".codex\skills")
  Retire-LegacyOne (Join-Path $env:USERPROFILE ".claude\skills")
}
Remove-Item -Recurse -Force $Tmp
```

## 更新后验证

```bash
AGENTS_HOME="${AGENTS_HOME:-$HOME/.agents}"
grep -n 'skill_repository: https://github.com/richardwild426/wx-channels-download' \
  "$AGENTS_HOME/skills/wx-channels-download/SKILL.md"
```

若当前对话已经加载旧 skill,更新文件不会 retroactively 改变本轮上下文。完成更新后,开启新会话再使用最新 skill。
