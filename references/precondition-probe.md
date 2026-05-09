# +probe(precondition probe)

> **场景:** 任何调用 wx_video_download API 之前的健康检查。SKILL.md 主页第 1 节内联了一行 sentinel,失败 → 来这个文件按下面 4 分支排错。
>
> **前置:** `WX_SERVER` 环境变量(可缺省,缺省值 `http://127.0.0.1:2022`),系统装了 `curl` + `jq`。用户不需要执行这些命令,由智能体执行。

## 根因判断

`channels.available` 的真实含义不是"二进制进程已启动",而是上游 `ChannelsClient.Validate()` 中 `len(ws_clients) > 0`。也就是说,必须有经过该实例代理并被注入脚本的视频号页面,通过 WebSocket 连接到这个 API 实例。智能体可以自动安装 / 启动服务,但微信登录、系统证书 / 代理授权、打开或刷新视频号页面仍需要用户在 GUI 中完成。

## 完整 4 分支判定脚本

```bash
#!/usr/bin/env bash
set -e
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"

# 分支 1:网络层失败(连不上)
RESP=$(curl -fsS --max-time 5 "$SERVER/api/status" 2>&1) || {
  echo "❌ 服务不可达 ($SERVER)。智能体应先安装或启动 wx_video_download,不要让用户执行 CLI。"
  echo "   下一步由智能体执行: 安装缺失二进制或启动本地服务。"
  exit 1
}

# 分支 2:HTTP 200 但 code != 0
CODE=$(echo "$RESP" | jq -r '.code')
if [ "$CODE" != "0" ]; then
  MSG=$(echo "$RESP" | jq -r '.msg // "未知错误"')
  echo "❌ 服务异常: $MSG (code=$CODE)。可能要重启 wx_video_download。"
  exit 2
fi

# 分支 3:服务起来但视频号客户端没接通
AVAIL=$(echo "$RESP" | jq -r '.data.channels.available')
if [ "$AVAIL" != "true" ]; then
  echo "❌ 视频号客户端未连接。请确认:"
  echo "   1. 微信 PC 已登录"
  echo "   2. 已允许系统证书 / 代理授权弹窗"
  echo "   3. 已打开或刷新任意视频号页面"
  exit 3
fi

# 分支 4:就绪
VERSION=$(echo "$RESP" | jq -r '.data.version')
echo "✓ 就绪 (wx_video_download $VERSION)"
```

## 字段(响应 schema,源:`internal/api/handler.go:88-105`)

| 字段 | 类型 | 含义 |
|---|---|---|
| `.code` | int | `0` 表示服务自身 OK,非 0 → 服务异常 |
| `.msg` | string | 服务自身的错误描述(中文) |
| `.data.version` | string | wx_video_download 的版本号 |
| `.data.channels.available` | bool | 视频号子系统是否接通(`c.channels.Validate()` 返回 nil) |

## 出错时

| 分支 | 触发条件 | 用户该做的 |
|---|---|---|
| 1 | curl 失败 / connection refused / timeout | 智能体安装或启动服务:先走 [`install-binary.md`](install-binary.md),再走 [`run-binary.md`](run-binary.md) |
| 2 | `code != 0` | 看上游日志(终端输出),按 `.msg` 排查;一般要重启 |
| 3 | `available == false` | API 进程存在,但没有视频号前端 WebSocket 客户端。提示用户完成微信登录、证书 / 代理授权、打开或刷新视频号页面 |
| 4 | 全部通过 | 继续后续命令 |

> 用户不需要知道 curl、端口、jq、launchctl 等细节。智能体只在 GUI 权限点停下来等用户确认。

## 链接

- [SKILL.md §1](../SKILL.md) — 主页内联的快速 sentinel
- [`install-binary.md`](install-binary.md) — 下载并校验上游原始 release 包
- [`run-binary.md`](run-binary.md) — 启动 / 停止本地二进制服务
- 任何 reference 中"反复失败"都回到这个文件
