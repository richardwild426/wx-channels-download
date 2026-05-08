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

### POST `/api/task/create_channels` 请求(`ChannelsDownloadPayload`,源:`handler.go:606-614`)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `url` | string | url/oid/nid/eid 至少一个 | share URL |
| `oid` | string | 同上 | object id |
| `nid` | string | 同上 | object nonce id |
| `eid` | string | 同上 | encrypted_objectid |
| `spec` | string | 否 | 清晰度,缺省下载默认(一般最低体积) |
| `mp3` | bool | 否 | 转 mp3,需要 ffmpeg |
| `cover` | bool | 否 | 只下封面 |

> 若 `url` 中含 `?eid=...`,服务端会优先把 eid 提取出来再 resolve(`handler.go:626-635`)。

### POST `/api/task/create_batch` 请求(`FeedDownloadTaskBody`,源:`handler.go:242-251`)

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

### `/api/channels/shared_feed/profile` 请求(GET query,源:`handler.go:228-240`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `url` | 是 | share URL |

### `/api/channels/shared_feed/profile` 响应关键字段(源:`internal/api/types/types.go` `ChannelsFeedProfileResp`,2 步链路用到)

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
{"code":0,"msg":"ok","data":{"id":"<task_id>"}}                  // create_channels
{"code":0,"msg":"ok","data":{"ids":["<task_id_1>", "<task_id_2>"]}} // create_batch
```

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `missing url` / `不合法的参数` / `缺少参数` / `缺少 feed id 参数` | 检查 share URL 完整性 / oid/nid 是否传对 |
| 409 | `已存在该下载内容` | 同 id+spec+suffix 已存在;改 spec/suffix 或 `+task delete` 旧任务 |
| 409 | `不合法的文件名,...` | 标题含非法字符(模板渲染失败);用 oid/nid 走自定义 filename |
| 500 | `获取详情失败:...` | share URL 已失效或微信侧拒绝;确认链接来源时间,可能要换链接 |
| 3001 | `下载 mp3 需要支持 ffmpeg 命令` | 用户本机装 ffmpeg,或改用默认 .mp4 |

## 链接

- [`tasks.md`](tasks.md) — 拿到 task_id 后查进度 / 暂停 / 重启
- [`creator-batch.md`](creator-batch.md) — 同一创作者多条作品时用这个,效率更高
- [`precondition-probe.md`](precondition-probe.md) — 反复失败时回这里
