---
name: wx-channels-download
description: 下载和管理微信视频号资源的 skill,基于 wx_video_download 二进制暴露的本地 HTTP API。提供原始 wx_channels_download 二进制安装、启动、健康检查、share URL 单条下载、按创作者批量下载、搜索创作者、列 feed / 直播回放 / 已互动、查询下载任务等能力。当用户提到视频号下载、share URL、视频号批量、创作者作品、视频号搜索、下载任务、wx_video_download、wx_channels_download 安装或启动时使用。服务地址通过环境变量 WX_SERVER 配置。
compatibility: Requires GitHub CLI for binary install, curl and jq for API calls, and a locally running wx_video_download service (default 127.0.0.1:2022) with WeChat PC client logged in.
license: MIT
metadata:
  version: "0.2.0"
  upstream: https://github.com/ltaoo/wx_channels_download
allowed-tools: Bash(curl:*) Bash(jq:*) Bash(gh:*) Bash(uname:*) Bash(mkdir:*) Bash(tar:*) Bash(unzip:*) Bash(chmod:*) Bash(ln:*) Bash(shasum:*) Bash(sha256sum:*) Bash(mktemp:*) Bash(rm:*) Bash(grep:*) Bash(command:*) Bash(test:*) Bash(seq:*) Bash(sleep:*) Bash(cat:*) Bash(pgrep:*) Bash(pkill:*) Bash(nohup:*) Bash(kill:*) Bash(pwsh:*) Bash(powershell:*) Read
---

# wx-channels-download

## 1. Precondition probe(必读)

任何 API 调用之前必跑。失败时按 [`references/precondition-probe.md`](references/precondition-probe.md) 分支排错;若服务未安装或未启动,分别走 [`references/install-binary.md`](references/install-binary.md) / [`references/run-binary.md`](references/run-binary.md)。

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
| 安装原始 wx_channels_download 二进制 | [`references/install-binary.md`](references/install-binary.md) |
| 启动 / 停止原始二进制服务 | [`references/run-binary.md`](references/run-binary.md) |
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
| 网络层 | curl 自身失败(connection refused / timeout) | 跑 §1 probe;若未安装走 `+install-binary`,若未启动走 `+run-binary` |
| 业务错(code != 0) | API 返回的中文 msg | 原样上报用户,**不重试** |
| 重复同错 ≥ 2 次 | 同样的请求连续失败 | 停,告诉用户,**不进入硬重试循环** |

### 不该做的
- 不要把 `.msg` 翻译 / 改写,逐字转述
- 不要硬重试业务错(409/400/3001 基本不会因重试改变)
- 不要在 skill 内"兜底",反复失败就交还用户判断

## 6. 反模式

- 不从非上游 release 来源安装 wx_video_download
- 不替用户登录微信 PC / 手动安装证书 / 处理系统代理授权弹窗
- 不在 skill 里持久化任何状态
- 不开 WebSocket(轮询代替)
- 不做长驻订阅
- 不重新发明分页 / 重试
