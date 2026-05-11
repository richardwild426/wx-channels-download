# wx-channels-download Skill 实施计划(MVP 修订版)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 spec(`docs/superpowers/specs/2026-05-08-wx-channels-skill-design.md`)落地为可安装的本地 skill —— 1 个 SKILL.md 主页 + **9 个** references markdown。2026-05-09 已追加从 GitHub 更新 skill、原始 `wx_channels_download` 二进制安装 / 运行封装;仍不修改上游 Go 项目。

**Architecture:** 文档型 skill。API reference 事实核查以上游 Go 源码为准;二进制安装 reference 以上游 GitHub release 资产命名和 checksums 为准。用户要求更新 skill 时,始终从 GitHub 仓库 `richardwild426/wx-channels-download` 拉取,不从本地工作区或软链接目标同步。

**Tech Stack:** Markdown(规范 frontmatter),YAML,bash + gh + curl + jq,Claude Code skill loader。

**全局约束(每个 task 都遵守 spec §1):**
- 只从上游 release 安装 / 运行原始二进制,不替用户登录 / 装证书 / 点系统代理授权
- 零状态,不持久化任何 SQLite / JSON / 缓存
- 不开 WebSocket(轮询代替),不做长驻订阅
- 本轮由 agent 启动的上游二进制必须在最终回复前停止,不留到会话退出后继续运行
- 不重新发明分页 / 重试 / 错误兜底
- 不翻译 `.msg`,逐字转述
- 反复同错就停

**占位约定:**
- 本 plan 内 skill name 暂用 `wx-channels-download`(spec 已用,合规 hyphen 名)
- 用户改完源目录后告知最终名;若不同,Task 8.0 用 sed 统一替换
- 所有 bash 命令使用相对路径(从仓库根目录执行),不假设绝对路径

**实施流程:**
本 plan 的所有 task 都基于已经核查过的事实(spec §3 已读源码片段)+ 实施期再读对应 handler 补全字段表。**没有真实 curl 实测 step**。

---

## File Structure

实施完成后:

```
<repo-root>/
├── .gitignore                     # Task 0.1
├── SKILL.md                       # Task 1.1,~150 行
├── references/                    # Task 1.1 隐含创建
│   ├── install-binary.md          # 2026-05-09 增补
│   ├── run-binary.md              # 2026-05-09 增补
│   ├── precondition-probe.md      # Task 2.1,~80 行
│   ├── tasks.md                   # Task 3.1,~120 行
│   ├── download-by-url.md         # Task 4.1,~150 行(A 优先级)
│   ├── creator-batch.md           # Task 5.1,~120 行(B 优先级)
│   ├── search-creator.md          # Task 6.1,~80 行
│   └── list-feed.md               # Task 6.2,~120 行(三合一)
└── docs/
    └── superpowers/
        ├── specs/2026-05-08-wx-channels-skill-design.md     # 已存在
        └── plans/2026-05-08-wx-channels-skill-implementation.md  # 本文件
```

**安装 / 更新目标(Task 8.2):**
```
<agent-resolved-skill-dir>/<skill-name>/   ← 由当前智能体自己的 skill 安装机制决定
```

---

## Task 0.1: .gitignore

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: 写 .gitignore**

```
.DS_Store
*.swp
.idea/
.vscode/
node_modules/
```

- [ ] **Step 2: Commit**

```bash
git add .gitignore
git commit -m "chore: 加 .gitignore"
```

---

## Task 1.1: SKILL.md 主页

**Files:**
- Create: `SKILL.md`

按 spec §3 实现。frontmatter + 6 节正文。决策表列二进制安装 / 运行和 6 个 API verb。

- [ ] **Step 1: 写 SKILL.md(完整内容)**

````markdown
---
name: wx-channels-download
description: 下载和管理微信视频号资源的 skill,基于 wx_video_download 二进制暴露的本地 HTTP API。提供原始 wx_channels_download 二进制安装、启动、健康检查、share URL 单条下载、按创作者批量下载、搜索创作者、列 feed / 直播回放 / 已互动、查询下载任务等能力。
compatibility: Requires GitHub CLI for binary install, curl and jq for API calls, and a locally running wx_video_download service with WeChat PC client logged in.
license: MIT
metadata:
  version: "0.4.3"
  upstream: https://github.com/ltaoo/wx_channels_download
allowed-tools: Bash(curl:*) Bash(jq:*) Bash(gh:*) ... Read
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

- **分页**:`next_marker`。各 list 类 API 都返回 `.data.next_marker`,空字符串表示尾页。把上一次 `next_marker` 当下一次入参传回。
- **重试**:不重新发明。API 自带语义,失败原样上报。
- **进度查询**:轮询 `/api/task/profile?id=<task_id>`,2~5s 间隔。状态字段见 [`references/tasks.md`](references/tasks.md)。
- **状态零持久化**:skill 不维护任何 sqlite / json。需要状态由 agent 用 TodoWrite 自管。

## 6. 全局错误处理

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

## 7. 反模式

- 不从非上游 release 来源安装 wx_video_download
- 不替用户登录 / 装证书
- 不把本轮由 agent 启动的 `wx_video_download` 留到会话退出后继续运行
- 不在 skill 里持久化任何状态
- 不开 WebSocket(轮询代替)
- 不做长驻订阅
- 不重新发明分页 / 重试
````

- [ ] **Step 2: 校验 frontmatter**

```bash
# description 长度
awk '/^description: /,0' SKILL.md | head -1 | wc -c
# 期望:< 1024(实际约 220)

# compatibility 长度
awk '/^compatibility: /,0' SKILL.md | head -1 | wc -c
# 期望:< 500(实际约 130)

# name 形态(小写字母 / 数字 / 连字符)
grep -E '^name: [a-z][a-z0-9-]*[a-z0-9]$' SKILL.md
# 期望:有匹配
```

- [ ] **Step 3: 校验 markdown 链接**

```bash
grep -oE 'references/[a-z-]+\.md' SKILL.md | sort -u
# 期望 6 个文件名:
#   precondition-probe.md / download-by-url.md / creator-batch.md
#   search-creator.md / list-feed.md / tasks.md
```

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "feat: 加 SKILL.md 主页和二进制生命周期入口"
```

---

## Task 2.1: precondition-probe reference

**Files:**
- Read: `internal/api/handler.go:88-105`(`handleStatus` 已读)
- Read: `internal/util/util.go:60-89`(响应约定已读)
- Create: `references/precondition-probe.md`

按 spec §7.2 4 分支判定矩阵实现,纯文档,无实测 step。

- [ ] **Step 1: 写完整内容**

````markdown
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
| 3 | `available == false` | 智能体已启动服务;提示用户完成微信登录、证书 / 代理授权、打开或刷新视频号页面 |
| 4 | 全部通过 | 继续后续命令 |

> **不要替用户做这些动作。** 报错给出指引即可,等用户操作完再让他重新触发请求。

## 链接

- [SKILL.md §1](../SKILL.md) — 主页内联的快速 sentinel
- 任何 reference 中"反复失败"都回到这个文件
````

- [ ] **Step 2: 自检无残留 placeholder**

```bash
grep -nE "实施期 TODO|待确认|TBD|TODO" references/precondition-probe.md
# 期望:无匹配,exit 1
```

- [ ] **Step 3: Commit**

```bash
git add references/precondition-probe.md
git commit -m "feat: 加 +probe reference(4 分支判定)"
```

---

## Task 3.1: tasks reference

**Files:**
- Read: `internal/api/handler.go`(`handleFetchTaskList` / `handleFetchTaskProfile` / `handleStartTask` / `handlePauseTask` / `handleResumeTask` / `handleDeleteTask` / `handleClearTasks`)
- Read: `gopeed` 依赖的 `base.Status` 与 `base.Task` 类型(用 `grep -rn "type Task struct\|type Status" $(go env GOMODCACHE)/github.com/!go!peed/...` 或先 `grep` 上游 import)
- Create: `references/tasks.md`

- [ ] **Step 1: 读相关 handler 函数,确认请求/响应字段**

```bash
cd /Users/zvector/ws/wx_channels_download && \
  grep -n "handleFetchTaskList\|handleFetchTaskProfile\|handleStartTask\|handlePauseTask\|handleResumeTask\|handleDeleteTask\|handleClearTasks" internal/api/handler.go
```

预期:每个 handler 一行定位。然后用 Read 看对应函数体(参考已读 `handleFetchTaskList` 在 396 行附近)。

记录每个函数:
- HTTP method
- 入参字段(query 或 JSON body)
- 出参字段(`.data` 内部 schema)

- [ ] **Step 2: 读 task 对象的字段(从 gopeed `base.Task` / `base.Status`)**

```bash
cd /Users/zvector/ws/wx_channels_download && \
  grep -rn "type Status\|type Task struct\|type Meta struct" $(go env GOMODCACHE 2>/dev/null)/github.com/!go!peed/ 2>/dev/null | head -10
# 若 GOMODCACHE 检索不到,fallback:
go list -m -f '{{.Dir}}' github.com/GopeedLab/gopeed 2>/dev/null || \
  find ~/go/pkg/mod -type d -name 'gopeed*' 2>/dev/null | head -3
```

打开 task 类型源码读字段名 + tag。

> 备注:`base.Status` 通常包含字符串枚举如 `"ready"` / `"running"` / `"pause"` / `"wait"` / `"error"` / `"done"`。具体值以源码为准。

- [ ] **Step 3: 写 references/tasks.md(完整内容,字段表填实测值)**

````markdown
# +task(下载任务管理)

> **场景:** 查询任务列表 / 查任务进度 / 暂停 / 重启 / 删除 / 清空。需要轮询场景也用本文件。
>
> **前置:** `WX_SERVER` 已设;先按 SKILL.md §1 跑 probe。

## 完整链路

### 列任务(分页 + 状态过滤)

```bash
curl -fsS "$WX_SERVER/api/task/list?status=all&page=1&page_size=20" | jq '.data'
# 响应:.data.list[] / .data.total / .data.page / .data.page_size
```

`status` 取值见下方"字段"节。

### 查单个任务进度(轮询)

```bash
TASK_ID="<task_id>"
while :; do
  RESP=$(curl -fsS "$WX_SERVER/api/task/profile?id=$TASK_ID")
  STATUS=$(echo "$RESP" | jq -r '.data.status // "unknown"')
  echo "[$(date +%H:%M:%S)] status=$STATUS"
  case "$STATUS" in
    done|error|canceled) break;;
  esac
  sleep 3
done
```

### 启动 / 暂停 / 恢复 / 删除 / 清空

```bash
# 启动(已创建未跑)
curl -fsS -X POST "$WX_SERVER/api/task/start" \
  -H 'Content-Type: application/json' -d "{\"id\":\"$TASK_ID\"}"

# 暂停
curl -fsS -X POST "$WX_SERVER/api/task/pause" \
  -H 'Content-Type: application/json' -d "{\"id\":\"$TASK_ID\"}"

# 恢复
curl -fsS -X POST "$WX_SERVER/api/task/resume" \
  -H 'Content-Type: application/json' -d "{\"id\":\"$TASK_ID\"}"

# 删除单个
curl -fsS -X POST "$WX_SERVER/api/task/delete" \
  -H 'Content-Type: application/json' -d "{\"id\":\"$TASK_ID\"}"

# 清空所有(危险动作,**先和用户确认**)
curl -fsS -X POST "$WX_SERVER/api/task/clear" \
  -H 'Content-Type: application/json' -d '{}'
```

## 字段

### `/api/task/list` 请求(query)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `status` | string | 否 | 状态过滤,见下表;`all` 或省略表示全部 |
| `page` | int | 否 | 页码,默认 1 |
| `page_size` | int | 否 | 每页条数,默认 20 |

### `/api/task/list` 响应

| 字段 | 类型 | 含义 |
|---|---|---|
| `.data.list[]` | array | task 对象列表(字段见下) |
| `.data.total` | int | 全部任务总数 |
| `.data.page` | int | 当前页 |
| `.data.page_size` | int | 每页 |

### `/api/task/profile` 请求(query)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | task_id |

### Task 对象字段(`/api/task/profile` 的 `.data`,以及 list 项)

> 字段名以 gopeed `base.Task` 为准。Step 2 读到的最终字段表如下(若实施期发现差异,以源码为准修订):

| 字段 | 类型 | 含义 |
|---|---|---|
| `.id` | string | task_id |
| `.status` | string | 任务状态(见状态枚举) |
| `.meta.req.url` | string | 下载源 URL |
| `.meta.req.labels` | object | 创建时塞的 labels(含 `id`/`nonce_id`/`title`/`spec`/`suffix`) |
| `.meta.opts.name` | string | 文件名 |
| `.meta.opts.path` | string | 下载目录 |
| `.size` | int | 总字节数 |
| `.progress.downloaded` | int | 已下载字节 |
| `.progress.speed` | int | 下载速度(字节/秒) |
| `.createdAt` / `.updatedAt` | string | RFC3339 时间戳 |

### 状态枚举(gopeed `base.Status`)

| 值 | 含义 |
|---|---|
| `ready` | 已创建,未启动 |
| `running` | 下载中 |
| `pause` | 暂停 |
| `wait` | 排队等待 |
| `error` | 失败,有错误 |
| `done` | 完成 |

### 写操作(start/pause/resume/delete)JSON body

```json
{"id": "<task_id>"}
```

### `/api/task/clear` JSON body

空对象 `{}`,无字段。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `不合法的参数` / `缺少 id` | 检查 task_id 来源(从 `+download` / `+batch` 返回值取) |
| 404 | task 不存在 | task 已被 clear / delete 或 id 错;先调 `/api/task/list` 确认 |
| 500 | 操作失败 + 详情 | 看 `.msg`,反复同错 → `+probe` |

## 链接

- [`download-by-url.md`](download-by-url.md) — 单条 share URL 创建任务,返回 task_id
- [`creator-batch.md`](creator-batch.md) — 批量创建,返回 ids[]
- [`precondition-probe.md`](precondition-probe.md) — 失败回这个文件
````

- [ ] **Step 4: 自检**

```bash
grep -nE "实施期 TODO|待确认|TBD|TODO" references/tasks.md
# 期望:无匹配
```

- [ ] **Step 5: Commit**

```bash
git add references/tasks.md
git commit -m "feat: 加 +task reference(7 个动作 + gopeed 状态枚举)"
```

---

## Task 4.1: download-by-url reference(优先级 A)

**Files:**
- Already read: `internal/api/handler.go:228-322`(`handleFetchSharedFeedProfile` / `handleCreateFeedDownloadTask` / `FeedDownloadTaskBody`)
- Already read: `internal/api/handler.go:507-604`(`handleBatchCreateTask` / `buildBatchCreateTask`)
- Already read: `internal/api/handler.go:606-...`(`handleCreateChannelsTask` / `ChannelsDownloadPayload`)
- Already read: `internal/api/client.go:231-340`(`createFeedTaskBody`)
- Already read: `internal/api/types/types.go:248-340`(`SharedFeedProfile*`、`ChannelsFeedProfileResp`)
- Create: `references/download-by-url.md`

事实已经核查完毕(见 spec §3 / §5.1),直接写。

- [ ] **Step 1: 写完整内容**

````markdown
# +download(share URL 单条下载)

> **场景:** 用户给一个微信视频号 share URL,把它下载到本机。
>
> **前置:** `WX_SERVER` 已设;按 SKILL.md §1 跑 probe;share URL 形如 `https://weixin.qq.com/sph/...` 或 `https://channels.weixin.qq.com/...`。

## 默认链路(1 步,推荐)

服务端自动:取详情 → 解析 media URL+key → 处理文件名模板 / 加签 / 图集 zip:// / mp3 / 封面 → 入下载队列。

```bash
SHARE_URL="<https://share-url>"
RESP=$(curl -fsS -X POST "$WX_SERVER/api/task/create_channels" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --arg url "$SHARE_URL" '{url: $url}')")
echo "$RESP" | jq -e '.code == 0' >/dev/null \
  || { echo "$RESP" | jq -r '.msg'; exit 1; }
TASK_ID=$(echo "$RESP" | jq -r '.data.id')
echo "task_id=$TASK_ID"
# 查进度走 references/tasks.md
```

### 可选参数(同一 endpoint 都支持)

```bash
# 自定义清晰度
jq -nc --arg url "$SHARE_URL" --arg spec "high" '{url:$url, spec:$spec}'
# 转 mp3(用户本机需有 ffmpeg)
jq -nc --arg url "$SHARE_URL" '{url:$url, mp3:true}'
# 只下载封面
jq -nc --arg url "$SHARE_URL" '{url:$url, cover:true}'
# 用 oid/nid/eid 替代 share URL
jq -nc --arg oid "$OID" --arg nid "$NID" '{oid:$oid, nid:$nid}'
```

## 进阶链路(2 步,显式)

适用场景:**先看清晰度选项 / 元数据后再决定**,或**多条 share URL 一次入批量队列**。

```bash
# Step A:取 share URL 详情
PROFILE=$(curl -fsS "$WX_SERVER/api/channels/shared_feed/profile?url=$(jq -nr --arg u "$SHARE_URL" '$u|@uri')")
echo "$PROFILE" | jq -e '.code == 0' >/dev/null \
  || { echo "$PROFILE" | jq -r '.msg'; exit 1; }

# Step B:从 profile 拼 FeedDownloadTaskBody,POST 批量
FEEDS=$(echo "$PROFILE" | jq '
  .data.object as $o |
  $o.objectDesc.media[0] as $m |
  [{
    id: $o.id,
    nonce_id: $o.objectNonceId,
    url: ($m.URL + $m.URLToken),
    title: $o.objectDesc.description,
    filename: $o.objectDesc.description,
    key: ($m.DecodeKey | tonumber? // 0),
    spec: "",
    suffix: ".mp4"
  }]
')
RESP=$(curl -fsS -X POST "$WX_SERVER/api/task/create_batch" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --argjson feeds "$FEEDS" '{feeds:$feeds}')")
echo "$RESP" | jq -r '.data.ids[]'
```

## 字段

### POST `/api/task/create_channels` 请求(`ChannelsDownloadPayload`,源:`handler.go:606`)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `url` | string | url/oid/nid/eid 至少一个 | share URL |
| `oid` | string | 同上 | object id |
| `nid` | string | 同上 | object nonce id |
| `eid` | string | 同上 | encrypted_objectid |
| `spec` | string | 否 | 清晰度,缺省下载默认(一般最低体积) |
| `mp3` | bool | 否 | 转 mp3,需要 ffmpeg |
| `cover` | bool | 否 | 只下封面 |

### POST `/api/task/create_batch` 请求(`FeedDownloadTaskBody`,源:`handler.go:242`)

```json
{"feeds": [{"id":"","nonce_id":"","url":"","title":"","filename":"","key":0,"spec":"","suffix":".mp4"}]}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | feed id |
| `nonce_id` | string | 否 | object nonce id |
| `url` | string | 是 | media URL(含 URLToken) |
| `title` | string | 否 | 显示名 |
| `filename` | string | 是 | 文件名(不含后缀) |
| `key` | int | 是 | media DecodeKey,0 表示无解密 |
| `spec` | string | 否 | 清晰度 |
| `suffix` | string | 是 | `.mp4` / `.mp3` / `.zip` |

### `/api/channels/shared_feed/profile` 请求(GET query)

| 字段 | 必填 | 说明 |
|---|---|---|
| `url` | 是 | share URL |

### `/api/channels/shared_feed/profile` 响应关键字段(源:`types/types.go:287` `ChannelsFeedProfileResp`,2 步链路用到)

| 字段 | 含义 |
|---|---|
| `.data.object.id` | feed id(同 oid) |
| `.data.object.objectNonceId` | feed nonce |
| `.data.object.objectDesc.description` | 标题 |
| `.data.object.objectDesc.media[0].URL` | media URL 主体 |
| `.data.object.objectDesc.media[0].URLToken` | URL 签名 token,拼到 URL 后面 |
| `.data.object.objectDesc.media[0].DecodeKey` | 解密 key(string,转 int) |
| `.data.object.objectDesc.media[0].Spec[]` | 清晰度选项数组 |

### 写操作响应

```json
{"code":0,"msg":"ok","data":{"id":"<task_id>"}}                  # create_channels
{"code":0,"msg":"ok","data":{"ids":["<task_id_1>", "<task_id_2>"]}}  # create_batch
```

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `missing url` / `不合法的参数` / `缺少 feed id 参数` | 检查 share URL 完整性 / oid/nid 是否传对 |
| 409 | `已存在该下载内容` | 同 id+spec+suffix 已存在;改 spec/suffix 或 `+task delete` 旧任务 |
| 500 | `获取详情失败:...` | share URL 已失效或微信侧拒绝;确认链接来源时间,可能要换链接 |
| 3001 | `下载 mp3 需要支持 ffmpeg 命令` | 用户本机装 ffmpeg,或改用默认 .mp4 |

## 链接

- [`tasks.md`](tasks.md) — 拿到 task_id 后查进度 / 暂停 / 重启
- [`creator-batch.md`](creator-batch.md) — 同一创作者多条作品时用这个,效率更高
- [`precondition-probe.md`](precondition-probe.md) — 反复失败时回这里
````

- [ ] **Step 2: 自检**

```bash
grep -nE "实施期 TODO|待确认|TBD|TODO" references/download-by-url.md
# 期望:无匹配
```

- [ ] **Step 3: Commit**

```bash
git add references/download-by-url.md
git commit -m "feat: 加 +download reference(优先级 A,1 步默认 + 2 步进阶)"
```

---

## Task 5.1: creator-batch reference(优先级 B)

**Files:**
- Read: `internal/api/handler.go`(`handleFetchFeedListOfContact` 入参/出参)
- Already read: `internal/api/types/types.go:343-366`(`ChannelsFeedList` / `ChannelsFeedProfile` / `ChannelsFeedAccount`)
- Already read: `internal/channels/client.go` 的 `RequestFrontend` 模式
- Create: `references/creator-batch.md`

- [ ] **Step 1: 读 `handleFetchFeedListOfContact` handler**

```bash
cd /Users/zvector/ws/wx_channels_download && \
  grep -n "handleFetchFeedListOfContact" internal/api/handler.go
```

Read 对应函数体,确认 query 字段名(应为 `username` / `next_marker`)与响应包装。

- [ ] **Step 2: 写完整内容**

````markdown
# +batch(按创作者批量下载)

> **场景:** 用户指定一个视频号创作者(用 `username`,形如 `v2_xxx@finder`),把这个创作者的全部或前 N 条作品下载下来。
>
> **前置:** `WX_SERVER` 已设;先按 SKILL.md §1 跑 probe;创作者 `username` 已知(若用户只给昵称,先用 `+search` 找到 username)。

## 完整链路(2 步)

```bash
USERNAME="<v2_xxx@finder>"
NEXT=""
PAGE=0
ALL_FEEDS="[]"
MAX_PAGES=10           # 防止失控,根据用户需求调

# Step 1:翻页拿 feed 列表
while [ "$PAGE" -lt "$MAX_PAGES" ]; do
  RESP=$(curl -fsS "$WX_SERVER/api/channels/contact/feed/list?username=$(jq -nr --arg u "$USERNAME" '$u|@uri')&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')")
  echo "$RESP" | jq -e '.code == 0' >/dev/null || { echo "$RESP" | jq -r '.msg'; exit 1; }
  ALL_FEEDS=$(jq -n --argjson a "$ALL_FEEDS" --argjson b "$(echo "$RESP" | jq '.data.list')" '$a + $b')
  NEXT=$(echo "$RESP" | jq -r '.data.next_marker // ""')
  [ -z "$NEXT" ] && break
  PAGE=$((PAGE + 1))
done

# Step 2:转换为 FeedDownloadTaskBody,批量入队列
FEEDS=$(echo "$ALL_FEEDS" | jq 'map({
  id, nonce_id,
  url,
  title,
  filename: .title,
  key: (.decrypt_key | tonumber? // 0),
  spec: "",
  suffix: ".mp4"
})')
RESP=$(curl -fsS -X POST "$WX_SERVER/api/task/create_batch" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --argjson feeds "$FEEDS" '{feeds:$feeds}')")
echo "$RESP" | jq -r '.data.ids[]'
```

## 分页范式

- 起始 `next_marker=""`(空字符串)
- 每次响应的 `.data.next_marker` 直接当下次入参
- `.data.next_marker == ""` 表示尾页,停止

## 字段

### GET `/api/channels/contact/feed/list` 请求(query,源:`internal/api/handler.go` `handleFetchFeedListOfContact`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `username` | 是 | 创作者 username(`v2_xxx@finder`) |
| `next_marker` | 否 | 分页游标,首页空字符串 |

### 响应(源:`types/types.go:343-366` `ChannelsFeedList` / `ChannelsFeedProfile`)

```json
{"code":0, "data": {"list": [<ChannelsFeedProfile>, ...], "next_marker": ""}}
```

`ChannelsFeedProfile` 字段(每项):

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | string | feed id |
| `nonce_id` | string | feed nonce id |
| `source_url` | string | 视频原始 URL |
| `url` | string | media URL(已含签名,可直接当 task body 的 url 用) |
| `title` | string | 视频标题 / 描述 |
| `decrypt_key` | string | 解密 key,字符串形式;转 task body 时 `tonumber? // 0` |
| `cover_url` | string | 封面 URL |
| `cover_width` / `cover_height` | int | 封面尺寸 |
| `duration` | int | 秒 |
| `file_size` | int | 字节 |
| `created_at` | int | unix 秒 |
| `contact.username` | string | 创作者 username |
| `contact.nickname` | string | 创作者昵称 |
| `contact.avatar_url` | string | 头像 |

### POST `/api/task/create_batch` 同 [`download-by-url.md`](download-by-url.md) 进阶链路。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `missing username` / `不合法的参数` | username 没传或格式错;一般是 `v2_xxx@finder` |
| 500 | 获取列表失败 | 微信侧拒绝;确认创作者存在,或换 NAS / 本机 |
| 409 | `已存在该下载内容`(出现在 `task/create_batch` 内部去重) | 服务端按 `id|spec|suffix` 自动跳过,**不算错**;`.data.ids` 会比 `.feeds` 短 |

## 链接

- [`search-creator.md`](search-creator.md) — 用昵称找 username
- [`tasks.md`](tasks.md) — 批量任务进度
- [`download-by-url.md`](download-by-url.md) — 单条下载
- [`precondition-probe.md`](precondition-probe.md) — 反复失败时回这里
````

- [ ] **Step 3: 自检**

```bash
grep -nE "实施期 TODO|待确认|TBD|TODO" references/creator-batch.md
# 期望:无匹配
```

- [ ] **Step 4: Commit**

```bash
git add references/creator-batch.md
git commit -m "feat: 加 +batch reference(优先级 B,创作者批量 + 分页)"
```

---

## Task 6.1: search-creator reference

**Files:**
- Read: `internal/api/handler.go`(`handleSearchChannelsContact`)
- Read: `internal/channels/client.go`(对应 `FetchChannelsContactSearch` / `RequestFrontend` 调用)
- Read: `internal/api/types/types.go`(响应类型,grep `ChannelsAccountSearch`)
- Create: `references/search-creator.md`

- [ ] **Step 1: 读 search 相关代码**

```bash
cd /Users/zvector/ws/wx_channels_download && \
  grep -n "handleSearchChannelsContact\|FetchChannelsContactSearch\|ChannelsAccountSearch" \
       internal/api/handler.go internal/channels/client.go internal/api/types/types.go
```

确认:
- query 入参字段名(`keyword` / `next_marker`)
- 响应类型(应是 `ChannelsAccountSearchResp` 或类似)的字段定义,特别是 list 项的字段

- [ ] **Step 2: 写完整内容**

````markdown
# +search(搜创作者)

> **场景:** 用户给一个昵称或关键词,要找到对应的视频号 username 以喂给 `+batch` / `+list`。
>
> **前置:** `WX_SERVER` 已设;按 SKILL.md §1 跑 probe;关键词非空字符串。

## 完整链路

```bash
KEYWORD="<关键词>"
NEXT=""
curl -fsS "$WX_SERVER/api/channels/contact/search?keyword=$(jq -nr --arg k "$KEYWORD" '$k|@uri')&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')" \
  | jq '.data'
```

翻页同 [`creator-batch.md`](creator-batch.md) 的分页范式。

## 字段

### 请求(query,源:`internal/api/handler.go` `handleSearchChannelsContact` + `types/types.go` `ChannelsAccountSearchBody`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `keyword` | 是 | 搜索关键词,UTF-8 |
| `next_marker` | 否 | 分页游标,首页空字符串 |

### 响应(源:Step 1 实读类型)

```json
{"code":0, "data": {"list": [<account>, ...], "next_marker": ""}}
```

list 项字段(以 `ChannelsFeedAccount` 为基础,实施期按 Step 1 读到的真实类型补全):

| 字段 | 类型 | 含义 |
|---|---|---|
| `username` | string | 创作者 username(`v2_xxx@finder`),喂给 `+batch` / `+list` |
| `nickname` | string | 昵称 |
| `avatar_url` | string | 头像 URL |

> 若上游响应类型字段更丰富(粉丝数 / 认证标 / 简介等),Step 2 写作时按真实字段名补充表格。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `missing keyword` | keyword 必传 |
| 500 | 搜索失败 | 微信侧限频或拒绝;稍后重试,或换关键词;反复失败 → `+probe` |

## 链接

- [`creator-batch.md`](creator-batch.md) — 搜到 username 后批量下载
- [`list-feed.md`](list-feed.md) — 搜到 username 后只列 feed
- [`precondition-probe.md`](precondition-probe.md)
````

- [ ] **Step 3: 自检**

```bash
grep -nE "实施期 TODO|待确认|TBD|TODO" references/search-creator.md
# 期望:无匹配
# 注意上面 reference 内容的"若上游响应类型字段更丰富..."这句若被字段表完全填齐,在写作时就该删除
```

如果 Step 1 读到上游字段比 reference 模板里多,**回到 Step 2 把字段表填满,然后删除"若上游响应类型字段更丰富..."这一段说明文字**。

- [ ] **Step 4: Commit**

```bash
git add references/search-creator.md
git commit -m "feat: 加 +search reference(创作者搜索 + 分页)"
```

---

## Task 6.2: list-feed reference

**Files:**
- Read: `internal/api/handler.go`(`handleFetchFeedProfile` / `handleFetchLiveReplayList` / `handleFetchInteractionedFeedList`)
- Already read: `internal/channels/client.go:280-318`(`FetchChannelsInteractionedFeedList` / `FetchChannelsFeedProfile`)
- Already read: `internal/api/types/types.go:287-345`(`ChannelsFeedProfileResp`、`ChannelsFeedListBody`、`ChannelsLiveReplayListBody`、`ChannelsInteractionedFeedListBody`)
- Create: `references/list-feed.md`

合并三个独立路由:`feed/profile` / `live/replay/list` / `interactioned/list`。

- [ ] **Step 1: 读三个 handler 入参**

```bash
cd /Users/zvector/ws/wx_channels_download && \
  grep -n "handleFetchFeedProfile\|handleFetchLiveReplayList\|handleFetchInteractionedFeedList" internal/api/handler.go
```

`handleFetchFeedProfile` 已读(`handler.go:206`,接 `oid/nid/url/eid`)。其余两个 Read 函数体确认 query 字段名(应为 `username` / `next_marker` 等)。

- [ ] **Step 2: 读 `interactioned/list` 的 `flag` 取值**

```bash
cd /Users/zvector/ws/wx_channels_download && \
  grep -rn "Flag\|flag" internal/channels/client.go internal/api/handler.go | grep -i "interaction"
```

确认 flag 是空字符串还是有具体取值(如 `"like"` / `"fav"` / 空)。

- [ ] **Step 3: 写完整内容**

````markdown
# +list(列 feed / 直播回放 / 已互动)

> **场景:** 用户想看某个 feed 的元数据 / 列直播回放 / 列自己点过赞收藏的视频,但**不要立即下载**。要下载请到 `+download` / `+batch`。
>
> **前置:** `WX_SERVER` 已设;按 SKILL.md §1 跑 probe。

## 三个子动作

### A. 单条 feed 详情(by oid/nid 或 url 或 eid)

```bash
# 已知 oid/nid
curl -fsS "$WX_SERVER/api/channels/feed/profile?oid=$OID&nid=$NID" | jq '.data.object'

# 已知 url(channels.weixin.qq.com 形式)
curl -fsS "$WX_SERVER/api/channels/feed/profile?url=$(jq -nr --arg u "$URL" '$u|@uri')" | jq '.data.object'

# 已知 eid(encrypted_objectid)
curl -fsS "$WX_SERVER/api/channels/feed/profile?eid=$EID" | jq '.data.object'
```

`oid/nid/url/eid` 任一即可;若 url 中含 `?eid=...`,服务端会自动提取。

### B. 直播回放列表

```bash
USERNAME="<v2_xxx@finder>"
NEXT=""
curl -fsS "$WX_SERVER/api/channels/live/replay/list?username=$(jq -nr --arg u "$USERNAME" '$u|@uri')&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')" \
  | jq '.data'
```

### C. 已互动 feed 列表

```bash
FLAG=""              # 取值见下方"字段"节
NEXT=""
curl -fsS "$WX_SERVER/api/channels/interactioned/list?flag=$FLAG&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')" \
  | jq '.data'
```

## 字段

### A 请求(query,源:`handler.go:206-226`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `oid` | 与 `nid/url/eid` 至少一个 | object id |
| `nid` | 同上 | object nonce id |
| `url` | 同上 | 视频号 URL(可含 eid) |
| `eid` | 同上 | encrypted_objectid |

### A 响应字段子集(源:`types/types.go:287` `ChannelsFeedProfileResp`)

| 字段 | 含义 |
|---|---|
| `.data.object.id` | feed id(同 oid) |
| `.data.object.objectNonceId` | feed nonce |
| `.data.object.objectDesc.description` | 标题 / 描述 |
| `.data.object.objectDesc.media[0].URL + URLToken` | media URL(可拼成下载 url) |
| `.data.object.objectDesc.media[0].DecodeKey` | 解密 key(string) |
| `.data.object.contact.username/nickname` | 创作者 |

### B 请求(query,源:`types/types.go:326` `ChannelsLiveReplayListBody`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `username` | 是 | 创作者 username |
| `next_marker` | 否 | 分页游标 |

### B 响应字段同 [`creator-batch.md`](creator-batch.md) `ChannelsFeedProfile` 列表。

### C 请求(query,源:`types/types.go:330` `ChannelsInteractionedFeedListBody`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `flag` | 见 Step 2 实读 | 互动类型筛选(若 Step 2 发现 flag 必传,这里改为"是") |
| `next_marker` | 否 | 分页游标 |

### C 响应字段同 [`creator-batch.md`](creator-batch.md) `ChannelsFeedProfile` 列表。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `不合法的参数` / `missing url` | A:至少传一个 oid/nid/url/eid;B:username 必传;C:看 flag 要求 |
| 500 | 获取列表失败 / 获取详情失败 | 微信侧拒绝;反复失败 → `+probe` |

## 链接

- [`download-by-url.md`](download-by-url.md) — 拿到详情后想下载它走这里
- [`creator-batch.md`](creator-batch.md) — 拿到 list 后批量下载
- [`precondition-probe.md`](precondition-probe.md)
````

- [ ] **Step 4: 自检**

```bash
grep -nE "实施期 TODO|待确认|TBD|TODO" references/list-feed.md
# 期望:无匹配
```

- [ ] **Step 5: Commit**

```bash
git add references/list-feed.md
git commit -m "feat: 加 +list reference(feed 详情 / 直播回放 / 已互动)"
```

---

## Task 8.0: 替换 skill name 占位(若需要)

**Files:**
- Conditional Modify: `SKILL.md`(若用户改的目录名不是 `wx-channels-download`)

用户回到 session 时告知最终源目录名 / skill name。若不同,统一替换 placeholder。

- [ ] **Step 1: 用户确认最终 skill name**

```bash
# 当前目录名(应该已等于最终 skill name)
basename "$(git rev-parse --show-toplevel)"
# 与 SKILL.md frontmatter 比对
grep '^name:' SKILL.md
```

- [ ] **Step 2(若不一致):sed 替换**

```bash
ACTUAL=$(basename "$(git rev-parse --show-toplevel)")
sed -i.bak "s/^name: wx-channels-download$/name: $ACTUAL/" SKILL.md
rm SKILL.md.bak
grep '^name:' SKILL.md
# 期望:与 ACTUAL 一致
```

- [ ] **Step 3: Commit(条件性)**

```bash
git diff --quiet SKILL.md || git commit -am "fix: 校准 SKILL.md frontmatter name 与父目录名一致"
```

---

## Task 8.1: skills-ref 验证

**Files:**
- 无新建,只验证

- [ ] **Step 1: 看 skills-ref 是否可用**

```bash
which skills-ref || echo "skills-ref 未装,跑手工核对"
```

- [ ] **Step 2(skills-ref 可用):跑验证**

```bash
skills-ref validate ./
# 期望:无错误
```

- [ ] **Step 3(skills-ref 不可用):手工核对**

按规范逐项检查:
- [ ] frontmatter 字段都在规范允许范围内(`name`/`description`/`compatibility`/`license`/`metadata`/`allowed-tools`)
- [ ] `name` 形态合规:`^[a-z][a-z0-9-]*[a-z0-9]$`,且等于父目录名
- [ ] `description` ≤ 1024 字符(用 `awk` 量)
- [ ] `compatibility` ≤ 500 字符
- [ ] references 引用一层深(`grep -E '\.\./[a-z]+/' references/*.md` 应无匹配)
- [ ] 所有 references 中 markdown 链接指向的文件都存在

- [ ] **Step 4: 不 commit(验证步骤)**

---

## Task 8.2: 落地安装 / 更新

**Files:**
- 无修改源仓库;从 GitHub 最新 main 拉取。目标目录由当前智能体自己的 skill 安装机制决定,更新时直接替换目标 skill 目录,不保留旧副本。

- [ ] **Step 1: 从 GitHub 拉取最新 skill**

```bash
SKILL_NAME=wx-channels-download
TMP="$(mktemp -d)"
gh repo clone richardwild426/wx-channels-download "$TMP/src" -- --depth 1
rm -rf "$TMP/src/.git"
```

- [ ] **Step 2: 验证安装结构**

```bash
test -f "$TMP/src/SKILL.md"
test -d "$TMP/src/references"
```

Expected: SKILL.md 存在,references 有 9 个 .md。

- [ ] **Step 3: 交给当前智能体的 skill 安装机制落地**

```bash
# 目标目录必须由当前智能体运行时解析,不要在本 skill 中写死统一目录。
# 若运行时没有专用安装命令,把解析出的目标目录传给 references/update-skill.md 的 WX_SKILL_INSTALL_DIR。
```

- [ ] **Step 4: 不 commit(安装动作不进 git);开启新会话加载最新 skill**

---

## 实施完成验收

全部任务完成后:

- [ ] SKILL.md + 9 个 references 都存在
- [ ] `skills-ref validate ./` 通过(或手工核对通过)
- [ ] 安装位置由当前智能体的 skill 安装机制决定,`<skill-name>` 等于源目录名 + frontmatter `name`
- [ ] git log 干净,所有 commit 消息中文 + 类型前缀
- [ ] **所有 reference 文件中不残留 `实施期 TODO` / `待确认` / `TBD` / `TODO` 字样**
- [ ] 抽样 grep 验证:`grep -rn "实施期 TODO\|待确认\|TBD" SKILL.md references/` 无匹配
- [ ] spec §9 已知开放问题中,#1(task 字段)、#2(search/list 字段)的 reference 字段表来自 Task 3.1 / 6.1 / 6.2 的源码核查,内容确实填齐
- [ ] feature/skill-mvp 分支可以合并回 main(由用户决定 PR 还是直接 merge)

---

## v0.2 候选(本 MVP 范围外)

- `+mp`(公众号 / `/api/mp/*`)
- `+filehelper`(文件传输助手 / `/api/filehelper/*`)
- `+decrypt`(本地 .mp4 离线解密)
- 端到端冒烟测试:用真实 wx_video_download 实例 + 真实 share URL 跑通 happy path 与错误路径
- 抽 scripts/:若 reference 中 jq 模板出现重复(预计 search/list/batch 三处的字段映射有共性),抽到 `scripts/feed_to_task_body.jq` 之类
- 上游升级跟踪:wx_video_download 新版若改 API,更新 metadata.upstream / version
