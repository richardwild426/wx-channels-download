---
name: wx-channels-download
description: 下载和管理微信视频号资源的 skill,基于 wx_video_download 二进制暴露的本地 HTTP API。提供 share URL 单条下载、按创作者批量下载、搜索创作者、列 feed / 直播回放 / 已互动、查询下载任务等能力。当用户提到视频号下载、share URL、视频号批量、创作者作品、视频号搜索、下载任务、wx_video_download 时使用。需用户先启动 wx_video_download 并登录微信 PC,服务地址通过环境变量 WX_SERVER 配置。
compatibility: Requires wx_video_download running locally (default 127.0.0.1:2022) with WeChat PC client logged in. Needs curl and jq.
license: MIT
metadata:
  version: "0.1.0"
  upstream: https://github.com/ltaoo/wx_channels_download
allowed-tools: Bash(curl:*) Bash(jq:*) Read
---

# wx-channels-download

## 1. Precondition probe(必读)

任何调用之前必跑。失败立即停,read [`references/precondition-probe.md`](references/precondition-probe.md) 排错。

```bash
curl -fsS "${WX_SERVER:-http://127.0.0.1:2022}/api/status" \
  | jq -e '.code == 0 and .data.channels.available == true' >/dev/null \
  || { echo "wx_video_download 未就绪,见 references/precondition-probe.md"; exit 1; }
```

## 2. 服务地址

`WX_SERVER` 环境变量,默认 `http://127.0.0.1:2022`。本机 / NAS 共用同一份 skill,改 env 即切。

```bash
export WX_SERVER=http://127.0.0.1:2022          # 本机
export WX_SERVER=http://192.168.1.10:2022       # NAS 示例
```

## 3. 决策表(用户场景 → reference)

| 用户场景 | 文件 |
|---|---|
| 任何调用之前 | [`references/precondition-probe.md`](references/precondition-probe.md) |
| 下载 share URL | [`references/download-by-url.md`](references/download-by-url.md) |
| 创作者批量 | [`references/creator-batch.md`](references/creator-batch.md) |
| 搜创作者 | [`references/search-creator.md`](references/search-creator.md) |
| 列 feed / 直播回放 / 已互动 | [`references/list-feed.md`](references/list-feed.md) |
| 任务进度 / 暂停 / 重启 / 清空 | [`references/tasks.md`](references/tasks.md) |

## 4. 通用范式

- **分页**:`next_marker`。各 list 类 API 都返回 `.data.next_marker`,空字符串表示尾页。把上一次 `next_marker` 当下一次入参传回。
- **重试**:不重新发明。API 自带语义,失败原样上报。
- **进度查询**:轮询 `/api/task/profile?id=<task_id>`,2~5s 间隔。状态字段见 [`references/tasks.md`](references/tasks.md)。
- **状态零持久化**:skill 不维护任何 sqlite / json。需要状态由 agent 用 TodoWrite 自管。

## 5. 全局错误处理

### 响应约定
所有 API 返回 HTTP 200(网络层失败除外)。用 `.code` 判定:
- `code == 0` 成功,数据在 `.data`
- `code != 0` 业务失败,中文描述在 `.msg`,**原样转述给用户**

### 三类错与处理

| 错类 | 怎么发生 | agent 怎么做 |
|---|---|---|
| 网络层 | curl 自身失败(connection refused / timeout) | 跑 §1 probe;若 probe 也失败,停 |
| 业务错(code != 0) | API 返回的中文 msg | 原样上报用户,**不重试** |
| 重复同错 ≥ 2 次 | 同样的请求连续失败 | 停,告诉用户,**不进入硬重试循环** |

### 不该做的
- 不要把 `.msg` 翻译 / 改写,逐字转述
- 不要硬重试业务错(409/400/3001 基本不会因重试改变)
- 不要在 skill 内"兜底",反复失败就交还用户判断

## 6. 反模式

- 不替用户启动 / 重启 wx_video_download
- 不替用户登录 / 装证书
- 不在 skill 里持久化任何状态
- 不开 WebSocket(轮询代替)
- 不做长驻订阅
- 不重新发明分页 / 重试
