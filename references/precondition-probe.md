# +probe(precondition probe)

> **场景:** 任何调用 wx_video_download API 之前的健康检查。SKILL.md 主页第 1 节内联了一行 sentinel,失败 → 来这个文件按下面 4 分支排错。
>
> **前置:** `WX_SERVER` 环境变量(可缺省,缺省值 `http://127.0.0.1:2022`),系统装了 `curl` + `jq`。

## 完整 4 分支判定脚本

```bash
#!/usr/bin/env bash
set -e
SERVER="${WX_SERVER:-http://127.0.0.1:2022}"

# 分支 1:网络层失败(连不上)
RESP=$(curl -fsS --max-time 5 "$SERVER/api/status" 2>&1) || {
  echo "❌ 服务不可达 ($SERVER)。请确认 wx_video_download 已以管理员/sudo 身份启动。"
  echo "   参考上游 README:https://github.com/ltaoo/wx_channels_download"
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
  echo "   1. 微信 PC 已登录(有时需要把视频号视频播放一次)"
  echo "   2. 系统代理生效(443 走 wx_video_download 拦截)"
  echo "   3. 必要时重启 wx_video_download"
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
| 1 | curl 失败 / connection refused / timeout | 起 wx_video_download(管理员/sudo);确认端口未被占用 |
| 2 | `code != 0` | 看上游日志(终端输出),按 `.msg` 排查;一般要重启 |
| 3 | `available == false` | 微信 PC 登录、播放一次视频号视频后再 probe;若仍 false → 检查证书与代理是否生效;重启 wx_video_download |
| 4 | 全部通过 | 继续后续命令 |

> **不要替用户做这些动作。** 报错给出指引即可,等用户操作完再让他重新触发请求。

## 链接

- [SKILL.md §1](../SKILL.md) — 主页内联的快速 sentinel
- 任何 reference 中"反复失败"都回到这个文件
