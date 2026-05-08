# wx-channels-download Skill 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 spec(`docs/superpowers/specs/2026-05-08-wx-channels-skill-design.md`)落地为可安装、可激活、可调通的本地 skill —— 1 个 SKILL.md 主页 + 9 个 references markdown,纯 curl/jq 实现,无独立二进制,无 scripts。

**Architecture:** 文档型 skill,产物全是 markdown,源仓库 `/Users/zvector/ws/wx_channels_agent/`,通过软链接(待 Task 0.1 实测)或 rsync 落地到 `~/.claude/skills/wx-channels-download/`。每个 reference 按 spec §4.2 五节模板写;每个步骤的"测试"= 用真实 wx_video_download 实例跑 curl,确认行为符合文档描述。

**Tech Stack:** Markdown(Lark-flavored),YAML frontmatter,bash + curl + jq,Claude Code skill loader。

**全局约束(每个 task 都要遵守 spec §1 8 条边界):**
- 不替用户启动二进制 / 登录微信 / 装证书
- 零状态,不持久化任何 SQLite / JSON / 缓存
- 不做长驻订阅,不开 WebSocket(轮询代替)
- 不重新发明分页 / 重试 / 错误兜底
- 不翻译 `.msg`,逐字转述
- 反复同错就停

**前置条件(实施期):**
- 用户已在 `127.0.0.1:2022` 启动 `wx_video_download`
- 微信 PC 已登录
- `WX_SERVER` 环境变量未设置时默认指向上述地址
- 测试期间偶尔需要真实 share URL / 创作者 username,实施者要让用户提供

---

## File Structure

实施完成后的最终目录:

```
/Users/zvector/ws/wx_channels_agent/
├── .gitignore                     # Task 0.2 新建
├── SKILL.md                       # Task 1.1 新建,~150 行
├── references/                    # Task 1.1 隐含创建
│   ├── precondition-probe.md      # Task 2.1,~80 行
│   ├── tasks.md                   # Task 3.1,~120 行
│   ├── download-by-url.md         # Task 4.1,~120 行(A 优先级)
│   ├── creator-batch.md           # Task 5.1,~120 行(B 优先级)
│   ├── search-creator.md          # Task 6.1,~80 行
│   ├── list-feed.md               # Task 6.2,~120 行
│   ├── official-account.md        # Task 6.3,~120 行
│   ├── filehelper.md              # Task 6.4,~100 行
│   └── decrypt-offline.md         # Task 7.1,**条件性**(若上游不存在则不创建)
└── docs/
    └── superpowers/
        ├── specs/2026-05-08-wx-channels-skill-design.md     # 已存在
        └── plans/2026-05-08-wx-channels-skill-implementation.md  # 本文件
```

**安装目标(Task 8.2 落地):**
```
~/.claude/skills/wx-channels-download/   ← 软链接或 rsync 副本到本仓库
```

每个 reference 有单一职责:覆盖一个 verb 的所有变体(默认 + 进阶),不跨 verb。SKILL.md 主页只做 routing + 通用范式 + 错误约定。

---

## Task 0.1: 决定安装方式(symlink vs rsync vs 重命名仓库)

> **解锁后续 Task 8.2 的安装动作。** spec §9 #4 明确这是开放问题,实施第一步必须先验证。

**Files:**
- 无新建,只决策

**Background:** Skill 规范要求 frontmatter `name` 必须等于父目录名(`wx-channels-download`)。本仓库源目录是 `wx_channels_agent`(下划线)。Claude Code 加载 skill 时若 resolve symlink 到 target,父目录名会不匹配。三种方案见 spec §9 #4。

- [ ] **Step 1: 准备一个最小测试 skill 验证 Claude Code 是否 resolve symlink**

```bash
# 在临时目录建一个名为 underscore_dir 的目录,放一个最小 SKILL.md(name 是 hyphen-name)
mkdir -p /tmp/skill_test/underscore_dir
cat > /tmp/skill_test/underscore_dir/SKILL.md <<'EOF'
---
name: hyphen-name
description: 测试 skill,用于探测 Claude Code 是否 resolve symlink 到 target 父目录。仅用于一次性安装方式验证。
---
# Test
EOF
ln -sf /tmp/skill_test/underscore_dir ~/.claude/skills/hyphen-name
```

- [ ] **Step 2: 启动新 Claude Code session,看 skill 是否被加载**

```bash
# 新开终端
claude --print "list available skills" 2>&1 | grep -i "hyphen-name" || echo "NOT LOADED"
```

Expected:两种结果之一 — 若加载成功 → 方案 a(symlink 不 resolve);若加载失败 → 方案 b 或 c。

- [ ] **Step 3: 根据结果记录决策**

把决策写入 `docs/superpowers/specs/2026-05-08-wx-channels-skill-design.md` 的 §9 #4 末尾,具体如下:

```markdown
**实测结论(2026-05-08):**
- 方案选定:[a / b / c]
- 实测命令与输出:[贴出 Step 1~2 的输出]
- 后续 Task 8.2 按此方案执行。
```

- [ ] **Step 4: 清理测试 skill**

```bash
rm ~/.claude/skills/hyphen-name
rm -rf /tmp/skill_test
```

- [ ] **Step 5: Commit**

```bash
git add docs/superpowers/specs/2026-05-08-wx-channels-skill-design.md
git commit -m "docs: 锁定 skill 安装方式(spec §9 #4)"
```

---

## Task 0.2: 加 .gitignore

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

按 spec §3 实现。frontmatter + 6 节正文。

- [ ] **Step 1: 写 SKILL.md(完整内容如下)**

````markdown
---
name: wx-channels-download
description: 下载和管理微信视频号资源的 skill,基于 wx_video_download 二进制暴露的本地 HTTP API。提供 share URL 单条下载、按创作者批量下载、搜索创作者、查询下载任务、订阅公众号、操作文件传输助手等能力。当用户提到视频号下载、share URL、视频号批量、创作者作品、视频号搜索、下载任务、公众号订阅、wx_video_download 时使用。需用户先启动 wx_video_download 并登录微信 PC,服务地址通过环境变量 WX_SERVER 配置。
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
| 公众号 | [`references/official-account.md`](references/official-account.md) |
| 文件传输助手 | [`references/filehelper.md`](references/filehelper.md) |
| 本地 .mp4 解密 | [`references/decrypt-offline.md`](references/decrypt-offline.md) |

## 4. 通用范式

- **分页**:`next_marker`。各 list 类 API 都返回 `.data.next_marker`,空字符串表示尾页。把上一次 `next_marker` 当下一次入参传回。具体格式见各 reference。
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
- 不要硬重试业务错(409/400/3001 等基本不会因重试改变)
- 不要在 skill 内"兜底",反复失败就交还用户判断

## 6. 反模式

- 不替用户启动 / 重启 wx_video_download
- 不替用户登录 / 装证书
- 不在 skill 里持久化任何状态
- 不开 WebSocket(轮询代替)
- 不做长驻订阅
- 不重新发明分页 / 重试
````

- [ ] **Step 2: 验证 frontmatter 合法**

```bash
# 检查 description 长度
head -20 SKILL.md | awk '/description:/,/^[a-z]+:/' | wc -c
# 期望:< 1024
# 检查 compatibility 长度
head -20 SKILL.md | awk '/compatibility:/,/^[a-z]+:/' | wc -c
# 期望:< 500
# 检查 name 等于安装目录名(虽然此时还没 link,本步只确认 frontmatter)
grep '^name:' SKILL.md
# 期望:name: wx-channels-download
```

Expected: description ≈ 230,compatibility ≈ 130,name 严格等于 wx-channels-download。

- [ ] **Step 3: 验证 markdown 链接路径正确**

```bash
# 决策表里所有 references/* 链接对应的文件应稍后会创建
grep -oE 'references/[a-z-]+\.md' SKILL.md | sort -u
```

Expected: 9 个文件名:precondition-probe.md / download-by-url.md / creator-batch.md / search-creator.md / list-feed.md / tasks.md / official-account.md / filehelper.md / decrypt-offline.md。

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "feat: 加 SKILL.md 主页(frontmatter + 6 节骨架)"
```

---

## Task 2.1: precondition-probe reference

**Files:**
- Create: `references/precondition-probe.md`

按 spec §7.2 4 分支判定矩阵实现。

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

## 字段(响应 schema)

| 字段 | 类型 | 含义 |
|---|---|---|
| `.code` | int | `0` 表示服务自身 OK,非 0 → 服务异常 |
| `.msg` | string | 服务自身的错误描述(中文) |
| `.data.version` | string | wx_video_download 的版本号 |
| `.data.channels.available` | bool | 视频号子系统是否接通 |

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
````

- [ ] **Step 2: 实测分支 4(happy path)**

```bash
# 期望:wx_video_download 在跑,微信 PC 已登录
WX_SERVER=http://127.0.0.1:2022 bash -c "$(awk '/^```bash/,/^```$/' references/precondition-probe.md | sed '1d;$d')"
```

Expected: `✓ 就绪 (wx_video_download <version>)`,exit 0。

- [ ] **Step 3: 实测分支 1(连不上)**

```bash
# 故意指向一个未监听端口
WX_SERVER=http://127.0.0.1:65535 bash -c "$(awk '/^```bash/,/^```$/' references/precondition-probe.md | sed '1d;$d')" 2>&1 || true
```

Expected: 输出"❌ 服务不可达",exit 1。

- [ ] **Step 4: 实测分支 3(微信未登录)**

如果当前微信 PC 是登录态,跳过此 step,在 spec §9 #2 的实施期再做(关掉微信再跑一次)。
否则:

```bash
# 关闭微信 PC 后重启 wx_video_download,再跑
WX_SERVER=http://127.0.0.1:2022 bash -c "$(awk '/^```bash/,/^```$/' references/precondition-probe.md | sed '1d;$d')" 2>&1 || true
```

Expected: `❌ 视频号客户端未连接`,exit 3。

- [ ] **Step 5: Commit**

```bash
git add references/precondition-probe.md
git commit -m "feat: 加 +probe reference(4 分支判定)"
```

---

## Task 3.1: tasks reference

**Files:**
- Create: `references/tasks.md`

覆盖 7 个 task 操作:list / profile / start / pause / resume / delete / clear。轮询节奏 2~5s。

- [ ] **Step 1: 写完整内容**

````markdown
# +task(下载任务管理)

> **场景:** 查询任务列表 / 查任务进度 / 暂停 / 重启 / 删除 / 清空。需要轮询场景也用本文件。
>
> **前置:** `WX_SERVER` 已设;先按 SKILL.md §1 跑 probe。

## 完整链路

### 列任务

```bash
curl -fsS "$WX_SERVER/api/task/list?status=all&page=1&page_size=20" | jq '.data'
# .data.list[] / .data.total / .data.page / .data.page_size
# status 可填:all / running / paused / done / error(具体看上游 base.Status 枚举)
```

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

### 暂停 / 恢复 / 删除 / 清空

```bash
# 暂停
curl -fsS -X POST "$WX_SERVER/api/task/pause" \
  -H 'Content-Type: application/json' -d "{\"id\":\"$TASK_ID\"}"

# 恢复
curl -fsS -X POST "$WX_SERVER/api/task/resume" \
  -H 'Content-Type: application/json' -d "{\"id\":\"$TASK_ID\"}"

# 启动(已创建但未跑的任务)
curl -fsS -X POST "$WX_SERVER/api/task/start" \
  -H 'Content-Type: application/json' -d "{\"id\":\"$TASK_ID\"}"

# 删除单个
curl -fsS -X POST "$WX_SERVER/api/task/delete" \
  -H 'Content-Type: application/json' -d "{\"id\":\"$TASK_ID\"}"

# 清空所有(危险动作,**先和用户确认**)
curl -fsS -X POST "$WX_SERVER/api/task/clear" \
  -H 'Content-Type: application/json' -d '{}'
```

## 字段

### 请求(POST 类)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 视情况 | task_id;`clear` 不需要 |

### 响应(`/api/task/profile` 关键字段)

> **实施期 TODO(spec §9 #2):** 实测一个真实运行中任务,确认下方字段准确性,删除待确认标记。

| 字段 | 类型 | 含义 |
|---|---|---|
| `.data.id` | string | task_id |
| `.data.status` | string | 任务状态(创建态 / 运行 / 暂停 / 完成 / 错误) |
| `.data.progress` | float | 进度 0~1(待确认) |
| `.data.downloaded` | int | 已下载字节数(待确认) |
| `.data.total_size` | int | 总字节数(待确认) |
| `.data.url` | string | 下载源 URL |
| `.data.labels` | object | 创建时塞的 labels(含 id / nonce_id / title / spec / suffix) |

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `不合法的参数` / `缺少 id` | 检查 task_id 来源(从 `+download` / `+batch` 返回值取) |
| 404 | `任务不存在` | task 已被 clear / delete 或 id 错;先调 `/api/task/list` 确认 |
| 500 | `操作失败:...` | 看 `.msg`,反复同错 → `+probe` |

## 链接

- [`download-by-url.md`](download-by-url.md) — 单条 share URL 创建任务,返回 task_id
- [`creator-batch.md`](creator-batch.md) — 批量创建,返回 ids[]
- [`precondition-probe.md`](precondition-probe.md) — 失败回这个文件
````

- [ ] **Step 2: 实测 task list(空仓也应返回 `.data.list = []` 或 null)**

```bash
curl -fsS "$WX_SERVER/api/task/list?status=all&page=1&page_size=20" | jq '.code, .data | keys'
```

Expected: code=0,keys 含 `list / total / page / page_size`。

- [ ] **Step 3: 实测 task profile schema(用真实 task)**

```bash
# 先用 +download(Task 4.1)创建一个任务拿 task_id;若本任务先于 4.1 执行,
# 用上游已有任务 id(从 list 拿)
TASK_ID=$(curl -fsS "$WX_SERVER/api/task/list?page=1&page_size=1" | jq -r '.data.list[0].id // empty')
if [ -z "$TASK_ID" ]; then
  echo "SKIP: 暂无任务可测,等 Task 4.1 完成后回填字段表"
else
  curl -fsS "$WX_SERVER/api/task/profile?id=$TASK_ID" | jq '.data | keys, .data.status, .data.progress'
fi
```

Expected: 列出真实字段名;若与 reference 中字段表有差,回到 Step 1 修订并删除"待确认"标记。

- [ ] **Step 4: Commit**

```bash
git add references/tasks.md
git commit -m "feat: 加 +task reference(7 个动作 + 轮询)"
```

---

## Task 4.1: download-by-url reference(优先级 A)

**Files:**
- Create: `references/download-by-url.md`

覆盖 spec §5.1:1 步默认路径(`/api/task/create_channels`)+ 2 步进阶路径(`shared_feed/profile` + `task/create_batch`)。

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
# 用 oid/nid/eid 替代 share URL(如果用户能拿到)
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

### POST `/api/task/create_channels` 请求

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `url` | string | 与 oid/nid/eid 互斥,至少一个 | share URL |
| `oid` | string | 同上 | object id |
| `nid` | string | 同上 | object nonce id |
| `eid` | string | 同上 | encrypted_objectid |
| `spec` | string | 否 | 清晰度,缺省下载默认(一般最低体积) |
| `mp3` | bool | 否 | 转 mp3,需要 ffmpeg |
| `cover` | bool | 否 | 只下封面 |

### POST `/api/task/create_batch` 请求

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

### 响应

```json
{"code":0,"msg":"ok","data":{"id":"<task_id>"}}
{"code":0,"msg":"ok","data":{"ids":["<task_id_1>", "<task_id_2>"]}}
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

- [ ] **Step 2: 实测 1 步默认路径**

让用户提供一个真实 share URL,导出为 `SHARE_URL`,然后:

```bash
RESP=$(curl -fsS -X POST "$WX_SERVER/api/task/create_channels" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --arg url "$SHARE_URL" '{url: $url}')")
echo "$RESP" | jq '.'
```

Expected: `code:0`,`data.id` 是非空字符串。

- [ ] **Step 3: 实测 2 步进阶路径**

```bash
PROFILE=$(curl -fsS "$WX_SERVER/api/channels/shared_feed/profile?url=$(jq -nr --arg u "$SHARE_URL" '$u|@uri')")
echo "$PROFILE" | jq '.code, .data.object | keys, .data.object.objectDesc.media[0] | keys'
```

Expected: `code:0`,object 含 `id / objectNonceId / objectDesc / contact`,media[0] 含 `URL / URLToken / DecodeKey / Spec`。若字段名与 reference 不符,回 Step 1 修订。

- [ ] **Step 4: 实测 409 重复**

```bash
# 用同一 share URL 再创建一次,期望 409
curl -fsS -X POST "$WX_SERVER/api/task/create_channels" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --arg url "$SHARE_URL" '{url: $url}')" | jq '.code, .msg'
```

Expected: code:409,msg:`已存在该下载内容`。

- [ ] **Step 5: Commit**

```bash
git add references/download-by-url.md
git commit -m "feat: 加 +download reference(优先级 A,1 步默认 + 2 步进阶)"
```

---

## Task 5.1: creator-batch reference(优先级 B)

**Files:**
- Create: `references/creator-batch.md`

覆盖 spec §5.2:`contact/feed/list` + `task/create_batch`。

- [ ] **Step 1: 写完整内容**

````markdown
# +batch(按创作者批量下载)

> **场景:** 用户指定一个视频号创作者(用 `username` / `finder_username`,形如 `v2_xxx@finder`),把这个创作者的全部或前 N 条作品下载下来。
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

### GET `/api/channels/contact/feed/list` 请求(query)

| 字段 | 必填 | 说明 |
|---|---|---|
| `username` | 是 | 创作者 username(`v2_xxx@finder`) |
| `next_marker` | 否 | 分页游标,首页空字符串 |

### 响应

```json
{"code":0, "data": {"list": [<ChannelsFeedProfile>, ...], "next_marker": ""}}
```

`ChannelsFeedProfile` 字段(每项):

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | string | feed id |
| `nonce_id` | string | feed nonce id |
| `url` | string | media URL(已含签名,可直接当 task body 的 url 用) |
| `title` | string | 视频标题 / 描述 |
| `decrypt_key` | string | 解密 key,字符串形式;转 task body 时 `tonumber? // 0` |
| `cover_url` | string | 封面 URL |
| `duration` | int | 秒 |
| `file_size` | int | 字节 |
| `created_at` | int | unix 秒 |
| `contact` | object | `{username, nickname, avatar_url}` |

### POST `/api/task/create_batch` 同 [`download-by-url.md`](download-by-url.md) 进阶链路。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `missing username` / `不合法的参数` | username 没传或格式错;一般是 `v2_xxx@finder` |
| 500 | 获取列表失败 | 微信侧拒绝;确认创作者存在,或换 NAS / 本机 |
| 409 | `已存在该下载内容`(出现在 `task/create_batch` 内部去重) | 服务端按 `id|spec|suffix` 自动跳过,**不算错**;.data.ids 会比 .feeds 短 |

## 链接

- [`search-creator.md`](search-creator.md) — 用昵称找 username
- [`tasks.md`](tasks.md) — 批量任务进度
- [`download-by-url.md`](download-by-url.md) — 单条下载
- [`precondition-probe.md`](precondition-probe.md) — 反复失败时回这里
````

- [ ] **Step 2: 实测翻页拿 feed 列表(空 username 应 400)**

```bash
curl -fsS "$WX_SERVER/api/channels/contact/feed/list?username=&next_marker=" | jq '.code, .msg'
```

Expected: code 非 0(400),msg 含 username 相关错误。

- [ ] **Step 3: 实测真实 username 翻页**

让用户提供一个真实 username(可以从 `+search` 或他自己的关注列表拿),然后:

```bash
USERNAME="<v2_xxx@finder>"
curl -fsS "$WX_SERVER/api/channels/contact/feed/list?username=$(jq -nr --arg u "$USERNAME" '$u|@uri')&next_marker=" \
  | jq '.code, .data.list | length, .data.next_marker'
```

Expected: code:0,list 长度 > 0,next_marker 是字符串(可空可非空)。

- [ ] **Step 4: 实测批量创建**

```bash
# 接 Step 3,把头 3 条作品入队列
LIST=$(curl -fsS "$WX_SERVER/api/channels/contact/feed/list?username=$(jq -nr --arg u "$USERNAME" '$u|@uri')&next_marker=" | jq '.data.list[:3]')
FEEDS=$(echo "$LIST" | jq 'map({id,nonce_id,url,title,filename:.title,key:(.decrypt_key|tonumber?//0),spec:"",suffix:".mp4"})')
curl -fsS -X POST "$WX_SERVER/api/task/create_batch" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --argjson feeds "$FEEDS" '{feeds:$feeds}')" | jq '.code, .data.ids'
```

Expected: code:0,ids 数组长度 ≤ 3(可能因去重而少)。

- [ ] **Step 5: Commit**

```bash
git add references/creator-batch.md
git commit -m "feat: 加 +batch reference(优先级 B,创作者批量下载 + 分页)"
```

---

## Task 6.1: search-creator reference

**Files:**
- Create: `references/search-creator.md`

- [ ] **Step 1: 写完整内容**

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
  | jq '.data.list[] | {username, nickname, avatar_url}'
```

翻页同 [`creator-batch.md`](creator-batch.md) 的分页范式。

## 字段

### 请求(query)

| 字段 | 必填 | 说明 |
|---|---|---|
| `keyword` | 是 | 搜索关键词,UTF-8 |
| `next_marker` | 否 | 分页游标,首页空字符串 |

### 响应 list 项

| 字段 | 类型 | 含义 |
|---|---|---|
| `username` | string | 创作者 username(`v2_xxx@finder`),喂给 `+batch` / `+list` |
| `nickname` | string | 昵称 |
| `avatar_url` | string | 头像 URL |

> 实施期 TODO:核查 `/api/channels/contact/search` 完整响应字段(读 `internal/channels/client.go` 中对应方法)。如果上游返回更多字段,加进来。

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

- [ ] **Step 2: 实测搜索**

```bash
KEYWORD="测试"
curl -fsS "$WX_SERVER/api/channels/contact/search?keyword=$(jq -nr --arg k "$KEYWORD" '$k|@uri')&next_marker=" \
  | jq '.code, .data.list[0] // null'
```

Expected: code:0,list[0] 是 object(或 list 为空)。

- [ ] **Step 3: 核查 contact/search 响应字段(spec §9 #2 范畴)**

```bash
grep -n "FetchChannelsContactSearch\|ChannelsAccountSearchBody" /Users/zvector/ws/wx_channels_download/internal/channels/client.go
```

读对应方法的响应 unmarshal 类型,确认字段表完整,删除"实施期 TODO"标记。

- [ ] **Step 4: Commit**

```bash
git add references/search-creator.md
git commit -m "feat: 加 +search reference(创作者搜索 + 分页)"
```

---

## Task 6.2: list-feed reference

**Files:**
- Create: `references/list-feed.md`

合并三个独立路由:
- `/api/channels/feed/profile` — 单条 feed 详情(by oid/nid 或 url)
- `/api/channels/live/replay/list` — 直播回放
- `/api/channels/interactioned/list` — 已互动(点赞/收藏)

- [ ] **Step 1: 写完整内容**

````markdown
# +list(列 feed / 直播回放 / 已互动)

> **场景:** 用户想看某个 feed 的元数据 / 列直播回放 / 列自己点过赞收藏的视频,但**不要立即下载**。要下载请到 `+download` / `+batch`。
>
> **前置:** `WX_SERVER` 已设;按 SKILL.md §1 跑 probe。

## 三个子动作

### A. 单条 feed 详情(by oid/nid 或 url)

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
FLAG=""              # 实施期 TODO:核查 flag 取值(可能区分点赞/收藏/评论)
NEXT=""
curl -fsS "$WX_SERVER/api/channels/interactioned/list?flag=$FLAG&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')" \
  | jq '.data'
```

## 字段

### A 响应字段子集

| 字段 | 含义 |
|---|---|
| `.data.object.id` | feed id(同 oid) |
| `.data.object.objectNonceId` | feed nonce |
| `.data.object.objectDesc.description` | 标题 / 描述 |
| `.data.object.objectDesc.media[0].URL + URLToken` | media URL(可拼成下载 url) |
| `.data.object.objectDesc.media[0].DecodeKey` | 解密 key(string) |
| `.data.object.contact.username/nickname` | 创作者 |

### B/C 响应字段同 [`creator-batch.md`](creator-batch.md) `ChannelsFeedProfile` 列表。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `不合法的参数` / `missing url` | A:至少传一个 oid/nid/url/eid;B/C:见对应接口要求 |
| 500 | 获取列表失败 / 获取详情失败 | 微信侧拒绝;反复失败 → `+probe` |

## 链接

- [`download-by-url.md`](download-by-url.md) — 拿到详情后想下载它走这里
- [`creator-batch.md`](creator-batch.md) — 拿到 list 后批量下载
- [`precondition-probe.md`](precondition-probe.md)
````

- [ ] **Step 2: 实测 A(feed/profile)**

让用户提供一个 share URL 或 oid/nid,跑:

```bash
curl -fsS "$WX_SERVER/api/channels/feed/profile?url=$(jq -nr --arg u "$URL" '$u|@uri')" \
  | jq '.code, .data.object | keys'
```

Expected: code:0,object 含 id/objectNonceId/objectDesc/contact。

- [ ] **Step 3: 实测 B(live/replay)**

```bash
curl -fsS "$WX_SERVER/api/channels/live/replay/list?username=$(jq -nr --arg u "$USERNAME" '$u|@uri')&next_marker=" \
  | jq '.code, .data | keys'
```

Expected: code:0,keys 含 list/next_marker。

- [ ] **Step 4: 实测 C(interactioned),确认 flag 取值**

```bash
# 试 flag 空
curl -fsS "$WX_SERVER/api/channels/interactioned/list?flag=&next_marker=" | jq '.code, .data | keys'
# 若返回错误 msg 提示 flag 必填,记录 msg 并尝试 1 / 2 / "like" / "fav" 等
```

把实测结果回填到 reference 的 C 节,删除"实施期 TODO"标记。

- [ ] **Step 5: Commit**

```bash
git add references/list-feed.md
git commit -m "feat: 加 +list reference(feed 详情 / 直播回放 / 已互动)"
```

---

## Task 6.3: official-account reference

**Files:**
- Create: `references/official-account.md`

REST 子集 only。WS 部分(`/ws/mp`、`/ws/manage`)显式声明不在范围内(spec §9 #3)。

- [ ] **Step 1: 核查上游 mp REST 路由实际能力**

```bash
grep -n "/api/mp/" /Users/zvector/ws/wx_channels_download/internal/api/routes.go
```

预期 routes(从已读 routes.go):
- `POST /api/mp/refresh_with_frontend` — 仅 local
- `GET /api/mp/ws_pool` — 仅 local
- `GET /api/mp/list`
- `GET /api/mp/msg/list`
- `GET /api/mp/article/list`
- `POST /api/mp/delete`
- `POST /api/mp/refresh`
- `GET /rss/mp`
- `GET /mp/proxy`
- `GET /mp/home`

读对应 handler(`internal/api/handler.go` 或 `internal/officialaccount/`)确认请求 / 响应结构。

- [ ] **Step 2: 写完整内容(基于 Step 1 实测)**

````markdown
# +mp(公众号)

> **场景:** 列已订阅公众号 / 看公众号文章列表 / 删订阅 / 刷新订阅 / 拿 RSS。
>
> **前置:** `WX_SERVER` 已设;按 SKILL.md §1 跑 probe;微信 PC 已订阅过想要看的公众号。
>
> **不在本 reference 范围:** `/ws/mp`、`/ws/manage` 推送类能力。Skill 设计已锁定不暴露 WS。

## 完整链路

### 列已订阅公众号

```bash
curl -fsS "$WX_SERVER/api/mp/list" | jq '.data'
```

### 列某公众号的文章

```bash
BIZ="<公众号 biz id>"
curl -fsS "$WX_SERVER/api/mp/article/list?biz=$(jq -nr --arg b "$BIZ" '$b|@uri')" | jq '.data'
```

### 列消息

```bash
curl -fsS "$WX_SERVER/api/mp/msg/list?biz=$(jq -nr --arg b "$BIZ" '$b|@uri')" | jq '.data'
```

### 删订阅

```bash
curl -fsS -X POST "$WX_SERVER/api/mp/delete" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --arg biz "$BIZ" '{biz:$biz}')"
```

### 刷新

```bash
curl -fsS -X POST "$WX_SERVER/api/mp/refresh" \
  -H 'Content-Type: application/json' -d '{}'
```

### RSS

```bash
curl -fsS "$WX_SERVER/rss/mp?biz=$(jq -nr --arg b "$BIZ" '$b|@uri')" -o feed.xml
```

## 字段

> **实施期 TODO(spec §9 #3):** 把每个 endpoint 的请求 / 响应字段表填进来,基于 Step 1 实测的 handler 阅读。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `不合法的参数` | biz 必传 |
| 500 | 获取列表失败 | 微信侧拒绝或本地索引未建;先 `refresh` 一次 |

## 链接

- [`precondition-probe.md`](precondition-probe.md)
````

- [ ] **Step 3: 实测 mp/list**

```bash
curl -fsS "$WX_SERVER/api/mp/list" | jq '.code, (.data | keys?)'
```

Expected: code:0(若用户无订阅,data 可能空)。

- [ ] **Step 4: 把字段表填齐**

读对应 handler 把请求 / 响应 schema 完整列出,删除"实施期 TODO"标记。

- [ ] **Step 5: Commit**

```bash
git add references/official-account.md
git commit -m "feat: 加 +mp reference(公众号 REST 子集)"
```

---

## Task 6.4: filehelper reference

**Files:**
- Create: `references/filehelper.md`

REST 子集 only。

- [ ] **Step 1: 核查上游 filehelper 路由**

```bash
grep -n "/api/filehelper" /Users/zvector/ws/wx_channels_download/internal/api/routes.go
```

预期(从已读 routes.go):
- `GET /api/filehelper/qrcode`
- `GET /api/filehelper/login/wait`
- `GET /api/filehelper/status`
- `GET /api/filehelper/synccheck`
- `GET /api/filehelper/sync`
- `GET /api/filehelper/messages`
- `POST /api/filehelper/send`
- `POST /api/filehelper/logout`
- `POST /api/filehelper/parse_finder_feed`

读 `internal/api/filehelper.go` 确认 handler。

- [ ] **Step 2: 写完整内容(基于 Step 1 实测)**

````markdown
# +filehelper(文件传输助手)

> **场景:** 通过文件传输助手会话拉消息 / 拿文件 / 解析转发的视频号链接。**注意:这套 endpoint 需要单独的微信网页登录态(扫二维码),与视频号核心功能独立。**
>
> **前置:** `WX_SERVER` 已设;按 SKILL.md §1 跑 probe;**filehelper 子系统额外要求一次性扫码登录**。

## 登录(一次性)

```bash
# 1) 拿二维码 PNG
curl -fsS "$WX_SERVER/api/filehelper/qrcode" -o qrcode.png
open qrcode.png   # macOS;Linux 用 xdg-open

# 2) 等待用户扫码(长轮询,服务端会等)
curl -fsS "$WX_SERVER/api/filehelper/login/wait" | jq '.data'

# 3) 检查状态
curl -fsS "$WX_SERVER/api/filehelper/status" | jq '.data'
```

> **提示用户去扫码,不替用户扫。** 二维码渲染、扫描动作必须由用户完成。

## 同步消息

```bash
curl -fsS "$WX_SERVER/api/filehelper/synccheck" | jq '.data'
curl -fsS "$WX_SERVER/api/filehelper/sync" | jq '.data'
curl -fsS "$WX_SERVER/api/filehelper/messages?limit=20" | jq '.data'
```

## 发消息

```bash
curl -fsS -X POST "$WX_SERVER/api/filehelper/send" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --arg c "$CONTENT" '{content:$c}')"
```

## 解析转发的视频号 feed

```bash
curl -fsS -X POST "$WX_SERVER/api/filehelper/parse_finder_feed" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --arg msg "$MSG_ID" '{msg_id:$msg}')"
# 返回的 feed 信息可以再喂给 +download
```

## 登出

```bash
curl -fsS -X POST "$WX_SERVER/api/filehelper/logout" -d '{}'
```

## 字段

> **实施期 TODO:** 各 endpoint 请求 / 响应字段表,基于 `internal/api/filehelper.go` 实测填齐。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `不合法的参数` | content / msg_id 必传 |
| 401 | `未登录` 或类似 | 重新走"登录"流程,扫码 |
| 500 | 服务端异常 | 回 `+probe` |

## 链接

- [`download-by-url.md`](download-by-url.md) — 解析出 feed 后再下载
- [`precondition-probe.md`](precondition-probe.md)
````

- [ ] **Step 3: 实测 status(无需登录的安全 endpoint)**

```bash
curl -fsS "$WX_SERVER/api/filehelper/status" | jq '.code, .data'
```

Expected: 任一情况都应返回 200 + 合理 status(可能是未登录 / 已登录)。不要试登录,因为会卡住或要扫码;让用户在自己 session 里跑。

- [ ] **Step 4: 把字段表填齐**

读 `internal/api/filehelper.go` 把请求 / 响应 schema 完整列出,删除"实施期 TODO"标记。

- [ ] **Step 5: Commit**

```bash
git add references/filehelper.md
git commit -m "feat: 加 +filehelper reference(REST 子集)"
```

---

## Task 7.1: decrypt-offline 存在性核查

**Files:**
- Conditional Create: `references/decrypt-offline.md`(若上游有对应路由)
- Conditional Modify: `SKILL.md`(若上游无路由,从决策表删除该行)

按 spec §9 #1 处理。

- [ ] **Step 1: 核查上游是否暴露离线解密 HTTP 端点**

```bash
grep -rn "decrypt\|Decrypt" /Users/zvector/ws/wx_channels_download/internal/api/routes.go /Users/zvector/ws/wx_channels_download/internal/api/handler.go
```

期望情况之一:
- **A. 找到路由**(如 `/api/decrypt` / `/api/file/decrypt`)→ 走 Step 2
- **B. 没找到路由**,但有 `internal/channels/decryptor.go` 内部使用 → 走 Step 3

- [ ] **Step 2(条件 A):写 references/decrypt-offline.md**

按上一步找到的 endpoint 写,覆盖典型场景:用户手上有一个加密 .mp4(如下载中断或之前用了 wx_video_download 的旧版),给文件路径或 byte stream + decode_key,API 返回解密后的 mp4。完整 5 节模板。

(若到这里发现 endpoint 复杂,本 reference 篇幅可达 150 行,不超出。)

```bash
# 写完后实测
echo "<实测命令依 endpoint 而定>"
```

- [ ] **Step 3(条件 B):从 SKILL.md 删该行**

```bash
# 编辑 SKILL.md,删决策表中那一行
# 用 sed 或手工 edit:
sed -i.bak '/decrypt-offline\.md/d' SKILL.md && rm SKILL.md.bak
diff -u <(git show HEAD:SKILL.md) SKILL.md   # 验证只删了一行
```

并且更新 spec 标记此开放问题已关闭:

```bash
# 编辑 spec §9 #1,加实测结论一行:
#   **实测结论(YYYY-MM-DD):上游不暴露离线解密 HTTP 端点,9 verb → 8。**
```

- [ ] **Step 4: Commit**

条件 A:
```bash
git add references/decrypt-offline.md
git commit -m "feat: 加 +decrypt reference(离线解密)"
```

条件 B:
```bash
git add SKILL.md docs/superpowers/specs/2026-05-08-wx-channels-skill-design.md
git commit -m "docs: 关闭 spec §9 #1(decrypt 不暴露 HTTP),9 verb → 8"
```

---

## Task 8.1: skills-ref 验证

**Files:**
- 无新建,只验证

- [ ] **Step 1: 看 skills-ref 是否可用**

```bash
which skills-ref || npm install -g skills-ref 2>&1 | head -5
# 或
which skills-ref || pip install skills-ref 2>&1 | head -5
# 工具仓:https://github.com/agentskills/agentskills/tree/main/skills-ref
```

- [ ] **Step 2: 跑验证**

```bash
cd /Users/zvector/ws/wx_channels_agent && skills-ref validate ./
```

Expected: 无错误。

如果工具不可获得(npm 没注册 / pip 没包),手工核对:
- frontmatter 字段都在规范允许范围内
- `name` 形态合规
- `description` ≤ 1024
- `compatibility` ≤ 500
- references 引用一层深

- [ ] **Step 3:(无需 commit,验证步骤)**

---

## Task 8.2: 落地安装

**Files:**
- 视 Task 0.1 选定方案而定

- [ ] **Step 1: 按 Task 0.1 决策执行**

**方案 a(symlink 不被 resolve):**
```bash
ln -sfn /Users/zvector/ws/wx_channels_agent ~/.claude/skills/wx-channels-download
ls -la ~/.claude/skills/wx-channels-download
```

**方案 b(改源仓库目录名):**

> 警告:目录改名会影响 git remote / 用户工作记忆,先和用户确认。

```bash
# 在用户确认后
cd /Users/zvector/ws && mv wx_channels_agent wx-channels-download
ln -sfn /Users/zvector/ws/wx-channels-download ~/.claude/skills/wx-channels-download
# 更新本机其它指向旧路径的引用(终端 cd / IDE 工作区等),用户处理
```

**方案 c(rsync 副本):**
```bash
rsync -av --delete \
  --exclude '.git' --exclude 'docs' \
  /Users/zvector/ws/wx_channels_agent/ \
  ~/.claude/skills/wx-channels-download/
# 后续每次源仓库改动后重跑;考虑加 git post-commit hook 自动同步
```

- [ ] **Step 2: 验证安装结构**

```bash
ls ~/.claude/skills/wx-channels-download/SKILL.md \
   ~/.claude/skills/wx-channels-download/references/
```

Expected: SKILL.md 存在,references 目录有 8 或 9 个 .md(看 Task 7.1 走的分支)。

- [ ] **Step 3:(无需 commit,安装动作不进 git)**

---

## Task 8.3: 端到端冒烟测试

**Files:**
- 无新建,只验证

- [ ] **Step 1: 启新 Claude Code session,确认 skill 自动加载**

```
启新终端:
  claude --print "available skills?"
检查输出含 wx-channels-download
```

- [ ] **Step 2: 让 skill 跑一遍 happy path**

```
对话内容:
  "用 wx-channels-download skill,先 probe 一下,然后下载这个 share URL: <真实 share URL>"
```

Expected: agent 跑 probe → 通过 → 调 `/api/task/create_channels` → 拿 task_id → 用 `+task` 轮询 → 报告完成。

- [ ] **Step 3: 验证错误路径**

```
对话内容:
  "再下一遍同一个 URL"
```

Expected: agent 收到 409 → 原样转述 `已存在该下载内容` → **不重试**。

- [ ] **Step 4: Commit + Tag**

```bash
git tag v0.1.0
git log --oneline | head -10
```

> 不 push 到 remote(用户全局 CLAUDE.md "禁止 force push、修改已 push 历史";本仓库尚无 remote)。push 由用户自行决定。

---

## 实施完成验收

全部任务完成后:

- [ ] SKILL.md + references 都存在
- [ ] `skills-ref validate ./` 通过(或手工核对通过)
- [ ] 安装在 `~/.claude/skills/wx-channels-download/`
- [ ] 新 session 能加载 skill
- [ ] `+probe` happy path 通过
- [ ] `+download` 1 步默认路径通过
- [ ] `+task` 轮询能正确读到 task 状态
- [ ] `+batch` 至少 1 条创作者作品入队列
- [ ] 错误路径(409 重复)不进硬重试
- [ ] git log 干净,所有 commit 消息中文 + 类型前缀
- [ ] spec §9 所有开放问题都已关闭(每条都有实测结论)
- [ ] **所有 reference 文件中不残留 `实施期 TODO` / `待确认` / `TBD` 字样**(产物洁净要求;每个 task 的实测 step 必须把这些标记替换为具体内容)
- [ ] 抽样 grep 验证:`grep -rn "实施期 TODO\|待确认\|TBD" SKILL.md references/` 应无匹配

---

## 下游事项(本计划范围外)

- 改进 reference:实施完后用户日常使用中如有别扭,迭代修订
- 抽 scripts/:当 reference 中 jq 模板出现重复(预计 search/list/batch 三处的字段映射有共性),抽到 `scripts/feed_to_task_body.jq` 之类,再在 reference 中改用 `jq -f`
- v0.2 范围:`+mp` / `+filehelper` 字段表完整化,WS 子集评估
- 上游升级跟踪:wx_video_download 新版若改 API,更新本 skill 的 metadata.upstream / version

