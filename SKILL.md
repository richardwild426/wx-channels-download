---
name: wx-channels-download
description: 下载和管理微信视频号资源的 skill,基于 wx_video_download 二进制暴露的本地 HTTP API。提供本 skill 从 GitHub 更新、原始 wx_channels_download 二进制安装、启动、健康检查、share URL 单条下载、按创作者批量下载、搜索创作者、列 feed / 直播回放 / 已互动、查询下载任务等能力。当用户提到视频号下载、share URL、视频号批量、创作者作品、视频号搜索、下载任务、wx_video_download、wx_channels_download 安装或启动、更新此 skill 时使用。服务地址通过环境变量 WX_SERVER 配置。
compatibility: Requires GitHub CLI for binary install, curl and jq for API calls, and a locally running wx_video_download service (default 127.0.0.1:2022) with WeChat PC client logged in.
license: MIT
metadata:
  version: "0.4.2"
  upstream: https://github.com/ltaoo/wx_channels_download
  skill_repository: https://github.com/richardwild426/wx-channels-download
allowed-tools: Bash(curl:*) Bash(jq:*) Bash(gh:*) Bash(uname:*) Bash(mkdir:*) Bash(tar:*) Bash(unzip:*) Bash(chmod:*) Bash(ln:*) Bash(shasum:*) Bash(sha256sum:*) Bash(mktemp:*) Bash(rm:*) Bash(mv:*) Bash(date:*) Bash(grep:*) Bash(command:*) Bash(test:*) Bash(id:*) Bash(cat:*) Bash(sleep:*) Bash(launchctl:*) Bash(systemctl:*) Bash(open:*) Bash(pwsh:*) Bash(powershell:*) Read
---

# wx-channels-download

## 1. Precondition probe(必读)

任何 API 调用之前必跑。失败时智能体自己处理安装 / 启动 / 重连,不要把 CLI 命令交给用户。只有微信登录、系统证书 / 代理授权、打开或刷新视频号页面这些 GUI 动作需要用户介入。失败分支见 [`references/precondition-probe.md`](references/precondition-probe.md);启动服务见 [`references/run-binary.md`](references/run-binary.md)。

```bash
curl -fsS "${WX_SERVER:-http://127.0.0.1:2022}/api/status" \
  | jq -e '.code == 0 and .data.channels.available == true' >/dev/null \
  || { echo "wx_video_download 未就绪,见 references/precondition-probe.md"; exit 1; }
```

## 2. 服务地址

`WX_SERVER` 环境变量,默认 `http://127.0.0.1:2022`。智能体默认使用这个地址并自动探测。`channels.available == true` 才能搜索 / 下载;单纯 API 服务启动成功不够。

```bash
export WX_SERVER=http://127.0.0.1:2022          # 本机
export WX_SERVER=http://192.168.1.10:2022       # NAS 示例
```

## 3. 决策表(用户场景 → reference)

| 用户场景 | 文件 |
|---|---|
| 更新本 skill | [`references/update-skill.md`](references/update-skill.md) |
| 安装原始 wx_channels_download 二进制 | [`references/install-binary.md`](references/install-binary.md) |
| 启动 / 停止原始二进制服务 | [`references/run-binary.md`](references/run-binary.md) |
| 任何调用之前 | [`references/precondition-probe.md`](references/precondition-probe.md) |
| 下载 share URL | [`references/download-by-url.md`](references/download-by-url.md) |
| 创作者批量 | [`references/creator-batch.md`](references/creator-batch.md) |
| 搜创作者 | [`references/search-creator.md`](references/search-creator.md) |
| 列 feed / 直播回放 / 已互动 | [`references/list-feed.md`](references/list-feed.md) |
| 任务进度 / 暂停 / 重启 / 清空 | [`references/tasks.md`](references/tasks.md) |

## 4. 生命周期与资源回收

- **只回收本轮启动的服务**:如果 probe 发现服务已可达,视为用户或外部系统已有服务,本 skill 不停止它。
- **启动即登记**:如果本 skill 通过 [`references/run-binary.md`](references/run-binary.md) 启动了 `wx_video_download`,agent 必须在本轮任务结束、用户取消、报错退出或会话交付前执行对应平台的停止命令。
- **不留长期驻留**:除非用户明确要求保留下载器服务,否则不要让 `wx_video_download` 在 agent 会话结束后继续运行。
- **下载任务边界**:如果下载任务还在运行,先用 [`references/tasks.md`](references/tasks.md) 查询并汇报状态;未获用户明确要求继续后台下载时,仍按本文停止上游二进制。

## 5. 通用范式

- **分页**:`next_marker`。不同 API 的游标字段名不完全一致;按对应 reference 读取 `lastBuff` / `lastBuffer`,空字符串表示尾页。
- **重试**:不重新发明。API 自带语义,失败原样上报。
- **进度查询**:轮询 `/api/task/list?status=all&page=1&page_size=200` 后按 `id` 过滤,2~5s 间隔。状态字段见 [`references/tasks.md`](references/tasks.md)。
- **状态零持久化**:skill 不维护任何 sqlite / json。需要状态由 agent 用 TodoWrite 自管。

## 6. 全局错误处理

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

## 7. 反模式

- 不从非上游 release 来源安装 wx_video_download
- 不从本地工作区更新 skill;用户要求更新 skill 时,始终从 `metadata.skill_repository` 的 GitHub 仓库拉取
- 不把安装、启动、端口探测、curl、jq 这些 CLI 细节交给用户
- 不用 `nohup ... &` 这种会被 agent 执行环境清理的方式启动服务
- 不把本轮由 agent 启动的 `wx_video_download` 留到会话退出后继续运行
- 不替用户登录微信 PC / 手动安装证书 / 处理系统代理授权弹窗
- 不在 skill 里持久化任何状态
- 不开 WebSocket(轮询代替)
- 不做长驻订阅
- 不重新发明分页 / 重试
