# +batch(按创作者批量下载)

> **场景:** 用户指定一个视频号创作者(用 `username`,形如 `v2_xxx@finder`),把这个创作者的全部或前 N 条作品下载下来。
>
> **前置:** `WX_SERVER` 已设;先按 SKILL.md §1 跑 probe;创作者 `username` 已知(若用户只给昵称,先用 `+search` 找到 username)。

## 完整链路(2 步)

```bash
USERNAME="<v2_xxx@finder>"
NEXT=""
PAGE=0
ALL_OBJECTS="[]"
MAX_PAGES=10           # 防止失控,根据用户需求调

# Step 1:翻页拿 feed 列表(响应是嵌套 .data.data.object[] 的 ChannelsObject 数组)
while [ "$PAGE" -lt "$MAX_PAGES" ]; do
  RESP=$(curl -fsS "$WX_SERVER/api/channels/contact/feed/list?username=$(jq -nr --arg u "$USERNAME" '$u|@uri')&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')")
  echo "$RESP" | jq -e '.code == 0' >/dev/null || { echo "$RESP" | jq -r '.msg'; exit 1; }
  PAGE_LIST=$(echo "$RESP" | jq '.data.data.object // []')
  ALL_OBJECTS=$(jq -n --argjson a "$ALL_OBJECTS" --argjson b "$PAGE_LIST" '$a + $b')
  HAS_MORE=$(echo "$RESP" | jq -r '.data.data.continueFlag // 0')
  NEXT=$(echo "$RESP" | jq -r '.data.data.lastBuffer // ""')
  [ "$HAS_MORE" != "1" ] && break
  [ -z "$NEXT" ] && break
  PAGE=$((PAGE + 1))
done

# Step 2:把 ChannelsObject 列表映射成 FeedDownloadTaskBody 后批量入队列
# 字段都是小写驼峰:objectDesc.media[0].url / urlToken / decodeKey
FEEDS=$(echo "$ALL_OBJECTS" | jq 'map(
  .objectDesc.media[0] as $m | {
    id: .id,
    nonce_id: .objectNonceId,
    url: ($m.url + $m.urlToken),
    title: .objectDesc.description,
    filename: .objectDesc.description,
    key: ($m.decodeKey | tonumber? // 0),
    spec: "",
    suffix: ".mp4"
  }
)')
RESP=$(curl -fsS -X POST "$WX_SERVER/api/task/create_batch" \
  -H 'Content-Type: application/json' \
  -d "$(jq -nc --argjson feeds "$FEEDS" '{feeds:$feeds}')")
echo "$RESP" | jq -r '.data.ids[]'
```

## 分页范式

| 情况 | 字段 |
|---|---|
| 起始 | `next_marker=""`(空字符串) |
| 下一页入参 | 把上一次响应的 `.data.data.lastBuffer` 当下一次 query 的 `next_marker` |
| 是否还有下页 | `.data.data.continueFlag == 1` 表示有,`0` 表示没 |
| 注意 | feed list 用 `lastBuffer`(末尾 r),与 search 的 `lastBuff` 字段名不同 |

## 字段

### GET `/api/channels/contact/feed/list` 请求(query,源:`internal/api/handler.go:42-51` `handleFetchFeedListOfContact`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `username` | 是 | 创作者 username(`v2_xxx@finder`);服务端会自动补 `@finder` 后缀 |
| `next_marker` | 否 | 分页游标,首页空字符串;下一页传上次响应的 `.data.data.lastBuffer` |

### 响应

实际响应是上游微信原始包装,经 `result.Ok` 再包一层 → **嵌套两层 `.data.data`**(源:`internal/api/types/types.go:196-247` `ChannelsFeedListOfAccountResp`):

```json
{
  "code": 0, "msg": "ok",
  "data": {
    "errCode": 0, "errMsg": "",
    "data": {
      "BaseResponse": {...},
      "object": [<ChannelsObject>, ...],
      "contact": {"username":"...", "nickname":"...", ...},
      "feedsCount": 0,
      "continueFlag": 0,
      "lastBuffer": "...",
      "userTags": [...]
    }
  }
}
```

`ChannelsObject` 列表项字段(源:`types/types.go:166-173` + `ChannelsObjectDesc`/`ChannelsMediaItem`):

| 字段 | 类型 | 含义 |
|---|---|---|
| `.id` | string | feed id(`oid`) |
| `.objectNonceId` | string | feed nonce id |
| `.source_url` | string | 视频原始 URL |
| `.createtime` | int | unix 秒 |
| `.contact.username/nickname/headUrl/signature/coverImgUrl` | — | 创作者 |
| `.objectDesc.description` | string | 视频标题 / 描述 |
| `.objectDesc.mediaType` | int | 媒体类型(图集/视频) |
| `.objectDesc.media[0].url` | string | media URL 主体 |
| `.objectDesc.media[0].urlToken` | string | URL 签名 token |
| `.objectDesc.media[0].decodeKey` | string | 解密 key(转 int) |
| `.objectDesc.media[0].spec[]` | array | 清晰度选项 |
| `.objectDesc.media[0].coverUrl` | string | 封面 URL |
| `.objectDesc.media[0].fileSize` | int | 字节数 |
| `.objectDesc.media[0].videoPlayLen` | int | 时长(秒) |

> 列表项是上游 `ChannelsObject` 原始结构,**不是**通过 `ChannelsObjectToChannelsFeedProfile` 转换后的扁平 `ChannelsFeedProfile`。要用前要按 `objectDesc.media[0]` 嵌套取字段。

### POST `/api/task/create_batch` 同 [`download-by-url.md`](download-by-url.md) 进阶链路。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `missing username` / `不合法的参数` / 上游错误描述 | username 没传或格式错;一般是 `v2_xxx@finder`(可省 `@finder` 服务端会补) |
| 409 | `已存在该下载内容`(出现在 `task/create_batch` 内部去重) | 服务端按 `id|spec|suffix` 自动跳过,**不算错**;`.data.ids` 会比 `.feeds` 短 |
| 反复同错 | — | 走 `+probe`,确认 channels 子系统 `available == true` |

## 链接

- [`search-creator.md`](search-creator.md) — 用昵称找 username
- [`tasks.md`](tasks.md) — 批量任务进度
- [`download-by-url.md`](download-by-url.md) — 单条下载
- [`precondition-probe.md`](precondition-probe.md) — 反复失败时回这里
