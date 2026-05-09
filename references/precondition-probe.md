# +probe(precondition probe)

> **场景:** 任何调用 wx_video_download API 之前的健康检查。SKILL.md 主页第 1 节内联了一行 sentinel,失败 → 来这个文件按下面 4 分支排错。
>
> **前置:** `WX_SERVER` 环境变量(可缺省,缺省值 `http://127.0.0.1:2022`),系统装了 `curl` + `jq`。`WX_SERVER` 必须指向用户已经手动验证正常的那个实例。

## 根因判断

`channels.available` 的真实含义不是"二进制进程已启动",而是上游 `ChannelsClient.Validate()` 中 `len(ws_clients) > 0`。也就是说,必须有经过该实例代理并被注入脚本的视频号页面,通过 WebSocket 连接到这个 API 实例。另起一个新实例即使打印了 `API服务启动成功` 和 `代理服务启动成功`,在微信页面没有重新接入它之前也会是 `available == false`。

## 完整 4 分支判定脚本

```bash
#!/usr/bin/env bash
set -e
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"

# 分支 1:网络层失败(连不上)
RESP=$(curl -fsS --max-time 5 "$SERVER/api/status" 2>&1) || {
  echo "❌ 服务不可达 ($SERVER)。这表示 skill 没连到用户手动验证正常的运行实例。"
  echo "   先确认真实 API 地址: curl -fsS http://127.0.0.1:2022/api/status | jq ."
  echo "   若手动实例使用其他端口,设置 WX_SERVER 指向真实地址。若未安装再看 references/install-binary.md。"
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
  echo "   1. 当前 WX_SERVER 指向的是用户手动验证正常的那个实例"
  echo "   2. 微信 PC 已登录,并且视频号页面经过这个实例的代理重新打开过"
  echo "   3. 系统代理仍指向该实例的代理端口,没有被另一个实例覆盖"
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
| 1 | curl 失败 / connection refused / timeout | skill 没连到手动验证正常的实例。确认真实端口并设置 `WX_SERVER`;只有确认未安装时才走 [`install-binary.md`](install-binary.md) |
| 2 | `code != 0` | 看上游日志(终端输出),按 `.msg` 排查;一般要重启 |
| 3 | `available == false` | API 进程存在,但没有视频号前端 WebSocket 客户端。通常是连到了另一个实例 / 新实例,或微信页面没有经过当前代理重新打开 |
| 4 | 全部通过 | 继续后续命令 |

> 安装可按本 skill 的 `+install-binary` 执行。运行阶段优先使用用户手动验证正常的前台实例;不要让 agent 为一次 API 调用临时后台启动新实例。

## 链接

- [SKILL.md §1](../SKILL.md) — 主页内联的快速 sentinel
- [`install-binary.md`](install-binary.md) — 下载并校验上游原始 release 包
- [`run-binary.md`](run-binary.md) — 启动 / 停止本地二进制服务
- 任何 reference 中"反复失败"都回到这个文件
