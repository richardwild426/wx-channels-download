# wx-channels-download skill 设计

**日期**: 2026-05-08
**状态**: 已实施;2026-05-09 增补原始二进制安装 / 运行封装
**规范依据**: [Agent Skills Specification](https://agentskills.io/specification)

---

## MVP 范围(2026-05-09 修订)

本次实施为 **8 verb,集中在原始二进制生命周期 + 视频号下载与浏览**:

| 保留 | 砍掉 |
|---|---|
| `+update-skill`、`+install-binary`、`+run-binary`、`+probe`、`+download`、`+batch`、`+search`、`+list`、`+task` | `+mp`(公众号)、`+filehelper`(文件传输助手)、`+decrypt`(离线解密) |

**实测策略**: API reference 事实核查以上游 Go 源码为准 —— 读 `internal/api/handler.go` / `internal/channels/client.go` / `internal/api/types/types.go` 的 struct 定义。二进制安装 reference 以上游 GitHub release 资产命名、checksums 和临时目录安装验证为准。

skill 名称以源仓库目录名为准(spec 写作期间目录尚未最终改名,占位 `wx-channels-download`,落地前以 frontmatter `name` 与父目录名一致为唯一约束)。

---

## 一句话目标

做一个本地 skill,让 agent 能安装并运行上游 release 中的原始 `wx_video_download` 二进制(默认监听 `127.0.0.1:2022` 的 HTTP API),再完成视频号 share URL 下载、按创作者批量下载、搜索、任务管理、列 feed 等动作。**不修改上游 Go 项目。**

---

## 1. 边界

skill 不做以下事情:

| # | 边界 | 处理方式 |
|---|---|---|
| 1 | 只从 GitHub 更新本 skill | `+update-skill` 克隆 `richardwild426/wx-channels-download`,不从本地工作区同步 |
| 2 | 只从上游 release 安装 `wx_video_download` | `+install-binary` 使用 `gh release download` + checksums |
| 3 | 自动连接 / 启动本地二进制服务 | macOS 用 LaunchAgent,Windows 用 Start-Process,Linux 用 systemd user service;本轮由 agent 启动的服务必须在最终回复前停止 |
| 4 | 不替用户登录微信 / 装证书 / 点系统代理授权 | 这些动作需要用户在 GUI 完成 |
| 5 | 零状态零持久化 | 只有 SKILL.md + references;启动所有权由 agent 用 TodoWrite 自管 |
| 6 | 不做长驻订阅 | YAGNI |
| 7 | 不封装 WebSocket 推送 | 进度走轮询 |
| 8 | NAS / 本机统一 | 一个 `WX_SERVER` 环境变量切换 |
| 9 | 不重新发明分页 / 重试 | API 自带 `next_marker`,直接文档化 |
| 10 | 不替用户做错误兜底 | 反复同错就停,交还用户判断 |

---

## 2. 仓库结构

源仓库目录(本仓库):

```
/Users/zvector/ws/wx_channels_agent/
├── HANDOFF.md                            # 临时交接物,spec 落地后归档
├── docs/
│   ├── superpowers/specs/                # brainstorming 产出
│   │   └── 2026-05-08-wx-channels-skill-design.md  # 本文件
│   └── handoff/                          # HANDOFF 归档目录
├── SKILL.md                              # skill 入口
└── references/
    ├── update-skill.md
    ├── install-binary.md
    ├── run-binary.md
    ├── precondition-probe.md
    ├── download-by-url.md                # 优先级 A
    ├── creator-batch.md                  # 优先级 B
    ├── search-creator.md
    ├── list-feed.md
    └── tasks.md

# v0.2 候选(MVP 不实施):
# ├── official-account.md
# ├── filehelper.md
# └── decrypt-offline.md
```

**安装 / 更新方式**: 用户要求更新 skill 时,以 GitHub 仓库为唯一来源:

```bash
gh repo clone richardwild426/wx-channels-download <tmp-dir> -- --depth 1
# 再把 <tmp-dir> 内容复制到 ~/.agents/skills/wx-channels-download
# 如果 ~/.codex/skills 或 ~/.claude/skills 已存在,再同步兼容副本
```

不要从当前本地工作区、软链接目标或旧拷贝更新 skill。`<skill-name>` 必须满足 agentskills 规范:小写字母 / 数字 / 连字符,且等于父目录名。

> 注:不在 skill 仓库中提供 `scripts/` 目录。所有命令以可复制 `gh` / `curl` / `jq` / shell 片段直接写在 reference 里。当主页或某个 reference 中模板出现重复,再考虑抽出 `scripts/`,届时仍在同一 skill 目录,不另开仓库。

---

## 3. SKILL.md 设计

### 3.1 frontmatter

```yaml
---
name: <skill-name>   # 与父目录名一致;落地前由用户确定
description: 下载和管理微信视频号资源的 skill,基于 wx_video_download 二进制暴露的本地 HTTP API。提供原始 wx_channels_download 二进制安装、启动、健康检查、share URL 单条下载、按创作者批量下载、搜索创作者、列 feed / 直播回放 / 已互动、查询下载任务等能力。
compatibility: Requires GitHub CLI for binary install, curl and jq for API calls, and a locally running wx_video_download service with WeChat PC client logged in.
license: MIT
metadata:
  version: "0.4.2"
  upstream: https://github.com/ltaoo/wx_channels_download
  skill_repository: https://github.com/richardwild426/wx-channels-download
allowed-tools: Bash(curl:*) Bash(jq:*) Bash(gh:*) ... Read
---
```

字段长度核对:
- `description` ≈ 220 字符 / 1024 上限
- `compatibility` ≈ 130 字符 / 500 上限
- `name` 满足正则 `^[a-z][a-z0-9-]*[a-z0-9]$`,且等于父目录名

### 3.2 主页正文骨架(6 节)

预期 100~180 行,远低于规范 500 行 / 5000 token 推荐上限。

```
## 1. Precondition probe(必读)
一行 curl /api/status + 期望响应,失败立即停,具体诊断指向 references/precondition-probe.md。

## 2. 服务地址
WX_SERVER 环境变量,默认 http://127.0.0.1:2022。本机 / NAS 共用,改 env 即切。

## 3. 决策表(用户场景 → reference)
| 场景 | 文件 |
|---|---|
| 更新本 skill | references/update-skill.md |
| 安装原始二进制 | references/install-binary.md |
| 启动 / 停止原始二进制 | references/run-binary.md |
| (任何 API 调用之前) | references/precondition-probe.md |
| 下载 share URL | references/download-by-url.md |
| 创作者批量 | references/creator-batch.md |
| 搜创作者 | references/search-creator.md |
| 列 feed / 直播回放 / 已互动 | references/list-feed.md |
| 任务进度 / 暂停 / 重启 / 清空 | references/tasks.md |

## 4. 通用范式
- 分页:next_marker(具体在各 reference)
- 重试:不重新发明,API 自带,失败原样上报
- 进度查询:轮询 /api/task/profile,2~5s 间隔
- 状态零持久化,需要状态由 agent 用 TodoWrite 自管

## 5. 全局错误处理
- 响应约定:HTTP 永远 200(除网络层),用 .code 判定。code == 0 成功,非 0 业务错,.msg 是中文,原样转述给用户。
- 三类错(表见下文 §6)
- 不要硬重试业务错;不要翻译 .msg;反复同错就停。

## 6. 反模式
- 只从 GitHub 仓库更新本 skill,不从本地工作区更新
- 只从上游 release 安装二进制
- 不替用户登录 / 装证书 / 点系统代理授权
- 不把本轮由 agent 启动的上游二进制留到会话退出后继续运行
- 不在 skill 里持久化任何状态
- 不开 WebSocket(轮询代替)
- 不做长驻订阅
```

### 3.3 主页第 1 节内联 probe(SKILL.md 中实际写的命令)

```bash
# 任何调用之前必跑。失败立即停,read references/precondition-probe.md 排错。
curl -fsS "${WX_SERVER:-http://127.0.0.1:2022}/api/status" \
  | jq -e '.code == 0 and .data.channels.available == true' >/dev/null \
  || { echo "wx_video_download 未就绪,见 references/precondition-probe.md"; exit 1; }
```

---

## 4. References 设计

### 4.1 切分(MVP:6 个 verb / 6 个 reference 文件)

| Verb 命名 | 文件 | 用户场景 | 优先级 | 主要路由 |
|---|---|---|---|---|
| `+probe` | precondition-probe.md | 服务起来了吗? | 跨场景必读 | `GET /api/status` |
| `+download` | download-by-url.md | 下载这个视频号链接 | **A** | `POST /api/task/create_channels`(1 步默认) / `GET /api/channels/shared_feed/profile` + `POST /api/task/create_batch`(2 步进阶) |
| `+batch` | creator-batch.md | 把 X 创作者的视频都下了 | **B** | `GET /api/channels/contact/feed/list` + `POST /api/task/create_batch` |
| `+search` | search-creator.md | 搜叫 X 的视频号 | C | `GET /api/channels/contact/search` |
| `+list` | list-feed.md | 列 X 的 feed / 直播回放 / 已互动 | C | `GET /api/channels/feed/profile` / `live/replay/list` / `interactioned/list` |
| `+task` | tasks.md | 下载到几号了 / 暂停 / 重启 / 清空 | C(必备支撑) | `GET /api/task/list`、`profile`、`POST start/pause/resume/delete/clear` |

**v0.2 候选(本 MVP 不做):**
- `+mp`(公众号 / `/api/mp/*`)
- `+filehelper`(文件传输助手 / `/api/filehelper/*`)
- `+decrypt`(本地 .mp4 解密)

> `+verb` 命名是文档约定,便于用户和 agent 脑内类比 lark-* skill 风格。规范不要求,可后续重命名而不破坏 skill 行为。

### 4.2 单 reference 文件骨架(5 节模板)

每个 reference 预期 80~150 行。

```
# <verb 一句话定位>

## 场景 + 前置(2~5 行)
什么场景读这个 / 运行前要满足什么 / 误用提示

## 完整链路
bash + curl + jq,真实可复制,多步链路一步一步给

## 字段(只列用到的)
表格:字段名 / 类型 / 必填 / 说明

## 出错时
3~5 条最高频错误 + 触发条件 + 修法;反复失败 → 回到 +probe

## 链接
兄弟 reference(下一步该读啥 / 配套 verb)
```

---

## 5. 关键链路细节

§3 事实核查后确定的实现路径。所有路由均已对照 `internal/api/routes.go` + `internal/api/handler.go` + `internal/channels/client.go` 验证。

### 5.1 优先级 A:share URL 单条下载

**默认路径(1 步,推荐):**

```bash
curl -fsS -X POST "$WX_SERVER/api/task/create_channels" \
  -H 'Content-Type: application/json' \
  -d '{"url": "<share_url>"}'
# 也支持: oid, nid, eid, spec, mp3 (bool), cover (bool)
# → {"code":0, "msg":"ok", "data":{"id": "<task_id>"}}
```

服务端(`internal/api/client.go:231` `createFeedTaskBody`)内部自动:
- 调 `FetchChannelsFeedProfile` 拿详情
- 解析 `media[0]` URL + DecodeKey
- 处理文件名模板 / URL 加签 / 图集 zip:// / mp3 转码 / 封面下载
- 入下载队列

**进阶路径(2 步,显式):**

```bash
# 1) 拿 share URL 详情
curl -fsS "$WX_SERVER/api/channels/shared_feed/profile?url=$(printf %s "$URL" | jq -sRr @uri)"
# 响应:.data.object.objectDesc.media[0] 含 URL / URLToken / DecodeKey / Spec[]

# 2) 拼装 FeedDownloadTaskBody 数组,POST 批量
curl -fsS -X POST "$WX_SERVER/api/task/create_batch" \
  -H 'Content-Type: application/json' \
  -d '{"feeds":[{"id":"<feed_id>","nonce_id":"<nonce>","url":"<media.URL+URLToken>","title":"<标题>","filename":"<安全文件名>","key":<atoi(decode_key)>,"spec":"","suffix":".mp4"}]}'
# → {"code":0,"msg":"ok","data":{"ids":[...]}}
```

适用场景:需要预览清晰度选项 / 元信息 / 想一次混合多条 share URL 入批量队列。

### 5.2 优先级 B:创作者批量下载

```bash
# 1) 列创作者 feed (按 username)
curl -fsS "$WX_SERVER/api/channels/contact/feed/list?username=<finder_username>&next_marker=" 
# 响应:.data.list[] 每项就是 ChannelsFeedProfile,字段 id / nonce_id / url(已含签名) / title / decrypt_key / ...

# 2) 转换为 feeds 数组,POST 批量
curl -fsS -X POST "$WX_SERVER/api/task/create_batch" \
  -H 'Content-Type: application/json' \
  -d "{\"feeds\":$(echo "$LIST_RESP" | jq '.data.list | map({id, nonce_id, url, title, filename: .title, key: (.decrypt_key | tonumber? // 0), spec: "", suffix: ".mp4"})')}"
```

分页:重复第 1 步,把上一页的 `.data.next_marker` 传到下一次 `next_marker=` 直到为空。

### 5.3 任务进度

```bash
curl -fsS "$WX_SERVER/api/task/profile?id=<task_id>"
# 轮询 2~5s 一次。状态字段在 .data 里,具体由 reference 文档化。
```

不开 WebSocket。zero state — agent 用 TodoWrite 自己记录哪个 task 对应哪个用户请求。

---

## 6. 全局错误处理

### 6.1 响应约定(实测)

上游 `internal/util/util.go:69-89` 的 `Ok` / `Err` helper 都返回 `http.StatusOK`,业务码在 JSON `.code`:

- `code == 0` → 成功,数据在 `.data`
- `code != 0` → 业务失败,中文描述在 `.msg`

HTTP 非 200 只在网络层(connection refused / timeout / 502)出现。

### 6.2 三类错与处理

| 错类 | 触发 | agent 行为 |
|---|---|---|
| 网络层 | curl 自身失败 | 跑 `+probe`;若 probe 也失败,停并告诉用户启动二进制 |
| 业务错 (code != 0) | API 返回的中文 msg | 原样上报用户,**不重试** |
| 重复同错 ≥ 2 次 | 同一请求连续失败 | 停,告诉用户,**不进入硬重试循环** |

### 6.3 已知错误码样本

(从 handler 收集到的 result.Err 调用,不穷举,各 reference 列本 verb 高频错)

| code | 典型 msg | 含义 |
|---|---|---|
| 400 | "missing url" / "不合法的参数" / "缺少 feed id 参数" | 参数错 |
| 409 | "已存在该下载内容" / "不合法的文件名" | 重复或冲突 |
| 500 | "创建任务失败:..." / "获取详情失败:..." | 服务端异常 |
| 3001 | "下载 mp3 需要支持 ffmpeg 命令" | 环境缺 ffmpeg |

### 6.4 不该做的

- 不要把 `.msg` 翻译 / 改写,逐字转述
- 不要硬重试业务错(409/400/3001 基本不会因重试改变)
- 不要在 skill 内"兜底",反复失败就交还用户判断

---

## 7. precondition probe 设计

### 7.1 主页内联(快速 sentinel,~5 行 bash,见 §3.3)

### 7.2 reference 完整 4 分支(`references/precondition-probe.md`,~80 行)

判定矩阵:

| 分支 | 判定 | 用户报错(中文) | 用户该做的 |
|---|---|---|---|
| 1 | curl 失败(connection refused / timeout) | "服务不可达 ($WX_SERVER)" | 先走 `+install-binary`,再走 `+run-binary`;确认端口未被占用 |
| 2 | code != 0 | "服务异常: $msg" | 看上游日志,可能要重启 |
| 3 | code == 0 但 data.channels.available == false | "视频号客户端未连接" | 智能体已启动服务;提示用户完成微信登录、证书 / 代理授权、打开或刷新视频号页面 |
| 4 | code == 0 且 channels.available == true | "✓ 就绪 (version: ...)" | 继续后续命令 |

reference 内提供完整 bash 脚本(每分支独立 echo + exit 不同非 0 码),agent 排错时打开按需读取。

---

## 8. 不在范围内 / 反模式

不做的事(对应 §1 边界):

- 不从非上游 release 来源安装 wx_video_download
- 不替用户装证书 / 改系统代理 / 登录微信 / 点系统代理授权
- 不把本轮由 agent 启动的上游二进制留到会话退出后继续运行
- 不在本仓库或 skill 安装目录持久化任何状态(SQLite / JSON / 缓存等)
- 不暴露或封装 WebSocket(`/ws/downloader`、`/ws/channels`、`/ws/mp` 等)
- 不做长驻订阅(场景 C)
- 不做硬重试循环
- 不翻译 / 改写上游 `.msg`,逐字转述
- 不在 SKILL.md 主页堆细节(progressive disclosure,主页 < 500 行)
- 不嵌套 reference 引用链(规范要求"一层")

---

## 9. 开放问题(实施前最终核查)

1. **`+task` 各子操作的状态字段**:`/api/task/profile` 和 `/api/task/list` 的响应 schema 不在已读源码片段中。实施 `references/tasks.md` 时需读 `internal/api/handler.go` 中 `handleFetchTaskProfile` 与 `handleFetchTaskList` 的实现 + `gopeed` 库的 task 类型,补全字段表。

2. **`+search` / `+list` 响应字段细节**:`contact/search` 和 `interactioned/list` 等 endpoint 的响应在 `internal/channels/client.go` 已有 `RequestFrontend` 包装,但具体字段映射(尤其 `interactioned/list` 的 `flag` 取值)需读 `internal/api/types/types.go` 确认。

3. **运行时 skill 更新来源**:更新必须从 GitHub 仓库 `richardwild426/wx-channels-download` 拉取,先复制到 canonical 目录 `~/.agents/skills/<name>`,再在 `~/.codex/skills` / `~/.claude/skills` 已存在时同步兼容副本。不要依赖本地工作区、软链接目标或旧拷贝,否则会出现"仓库已更新但运行时加载旧 skill"。

4. **MVP 不做的事**:`+mp` / `+filehelper` / `+decrypt` 推到 v0.2,实施期不写对应 reference。SKILL.md 主页的决策表里也不列。

5. **不实测策略的代价**:本 MVP 完全靠源码核查,意味着 reference 里写的 curl 命令**未在真实运行实例上验证过**。v0.2 引入端到端冒烟测试时若发现差异,以实测结果回填修订各 reference。

---

## 10. 验证

skill 落地后用 [skills-ref](https://github.com/agentskills/agentskills/tree/main/skills-ref) 验证 frontmatter:

```bash
skills-ref validate ./
```

通过即上线(按 `references/update-skill.md` 从 GitHub 安装到运行时 skill 目录)。

---

## 11. 后续

- 把 HANDOFF.md 挪到 `docs/handoff/2026-05-08-handoff.md` 归档,或在 spec 落地后直接删除并依赖 git 历史
- 调用 `superpowers:writing-plans` 生成实施计划,把 §3~§7 拆成可独立 commit 的步骤
