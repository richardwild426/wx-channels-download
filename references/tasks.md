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

### 查单个任务进度(轮询 list 并按 id 过滤)

> **注意:** `/api/task/profile?id=...` 只返回 `{path, name}`(用于展示落盘文件路径),**不含进度字段**。要拿 `status` / `progress` 必须从 `/api/task/list` 中按 `id` 过滤。

```bash
TASK_ID="<task_id>"
while :; do
  RESP=$(curl -fsS "$WX_SERVER/api/task/list?status=all&page=1&page_size=200")
  TASK=$(echo "$RESP" | jq --arg id "$TASK_ID" '.data.list[] | select(.id == $id)')
  if [ -z "$TASK" ]; then
    echo "task $TASK_ID 不在列表(可能被 clear/delete)"; break
  fi
  STATUS=$(echo "$TASK" | jq -r '.status')
  DL=$(echo "$TASK" | jq -r '.progress.downloaded // 0')
  SPD=$(echo "$TASK" | jq -r '.progress.speed // 0')
  echo "[$(date +%H:%M:%S)] status=$STATUS downloaded=$DL speed=$SPD"
  case "$STATUS" in
    done|error) break;;
  esac
  sleep 3
done
```

> 若 task 列表很长,把 `page_size` 调大一次拿到全部;不要并发轮询。

### 查任务落盘路径(profile)

```bash
TASK_ID="<task_id>"
curl -fsS "$WX_SERVER/api/task/profile?id=$TASK_ID" | jq '.data'
# 响应:.data.path / .data.name
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
  -H 'Content-Type: application/json' -d ''
```

> `/api/task/clear` handler 不读 body(`handleClearTasks`,`handler.go:773`),传空字符串 / `{}` 都行。

## 字段

### `/api/task/list` 请求(query,源:`internal/api/handler.go:396-432`)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `status` | string | 否 | 状态过滤,见状态枚举;`all` 或省略表示全部 |
| `page` | int | 否 | 页码,默认 1 |
| `page_size` | int | 否 | 每页条数,默认 20 |

### `/api/task/list` 响应

| 字段 | 类型 | 含义 |
|---|---|---|
| `.data.list[]` | array | task 对象列表(字段见下) |
| `.data.total` | int | 全部任务总数 |
| `.data.page` | int | 当前页 |
| `.data.page_size` | int | 每页 |

### `/api/task/profile` 请求(query,源:`internal/api/handler.go:915-934`)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 是 | task_id |

### `/api/task/profile` 响应

| 字段 | 类型 | 含义 |
|---|---|---|
| `.data.path` | string | 下载目录(`Meta.Opts.Path`) |
| `.data.name` | string | 文件名(`Meta.Opts.Name`) |

> profile **不含** status / progress / size,要进度走 list。

### Task 对象字段(`/api/task/list` 的 `.data.list[]`,源:`pkg/gopeed/pkg/download/model.go:24-95`)

| 字段 | 类型 | 含义 |
|---|---|---|
| `.id` | string | task_id |
| `.protocol` | string | 下载协议(`http` / `stream` / `zip` / `officialaccount`) |
| `.status` | string | 任务状态(见状态枚举) |
| `.name` | string | 显示名(`MarshalJSON` 拼装,落盘文件名) |
| `.uploading` | bool | 是否上传中(本仓库未用) |
| `.meta.req.url` | string | 下载源 URL |
| `.meta.req.labels` | object | 创建时塞的 labels(含 `id`/`nonce_id`/`title`/`spec`/`suffix`) |
| `.meta.opts.name` | string | 自定义文件名 |
| `.meta.opts.path` | string | 下载目录 |
| `.meta.res.name` | string | 资源名(协议 resolve 后填) |
| `.meta.res.size` | int | 资源总字节(协议 resolve 后填) |
| `.meta.res.files[]` | array | 文件列表(.zip / 多文件协议才有) |
| `.progress.used` | int | 累计耗时(纳秒) |
| `.progress.speed` | int | 下载速度(字节/秒) |
| `.progress.downloaded` | int | 已下载字节 |
| `.progress.uploadSpeed` | int | 上传速度(本仓库未用) |
| `.progress.uploaded` | int | 已上传字节(本仓库未用) |
| `.createdAt` | string | RFC3339 时间戳 |
| `.updatedAt` | string | RFC3339 时间戳 |

### 状态枚举(源:`pkg/gopeed/pkg/base/constants.go:5-12`)

| 值 | 含义 |
|---|---|
| `ready` | 已创建,未启动 |
| `running` | 下载中 |
| `pause` | 暂停 |
| `wait` | 排队等待 |
| `error` | 失败,有错误 |
| `done` | 完成 |

### 写操作(start/pause/resume/delete)JSON body(源:`handler.go:701-770`)

```json
{"id": "<task_id>"}
```

每个 handler 都用 `Id string` 字段,缺失返回 `400 缺少 feed id 参数`。

### `/api/task/clear` body(源:`handler.go:773-780`)

无字段,handler 不读 body。任何空 body 即可。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `不合法的参数` / `缺少 feed id 参数` / `missing task id` | 检查 task_id 来源(从 `+download` / `+batch` 返回值取);check JSON body |
| 404 | `task not found` | task 已被 clear / delete 或 id 错;先调 `/api/task/list` 确认 |
| 500 | 操作失败 + 详情 | 看 `.msg`,反复同错 → `+probe` |

## 链接

- [`download-by-url.md`](download-by-url.md) — 单条 share URL 创建任务,返回 task_id
- [`creator-batch.md`](creator-batch.md) — 批量创建,返回 ids[]
- [`precondition-probe.md`](precondition-probe.md) — 失败回这个文件
