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

### GET `/api/channels/contact/feed/list` 请求(query,源:`internal/api/handler.go:42-51` `handleFetchFeedListOfContact`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `username` | 是 | 创作者 username(`v2_xxx@finder`) |
| `next_marker` | 否 | 分页游标,首页空字符串 |

### 响应(源:`internal/api/types/types.go:343-366` `ChannelsFeedList` / `ChannelsFeedProfile`)

```json
{"code":0, "data": {"list": [<ChannelsFeedProfile>, ...], "next_marker": ""}}
```

`ChannelsFeedProfile` 字段(每项):

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | string | feed id |
| `nonce_id` | string | feed nonce id |
| `source_url` | string | 视频原始 URL |
| `url` | string | media URL(已含签名 token,可直接当 task body 的 url 用) |
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
