# +list(列 feed / 直播回放 / 已互动)

> **场景:** 用户想看某个 feed 的元数据 / 列直播回放 / 列自己点过赞收藏的视频,但**不要立即下载**。要下载请到 `+download` / `+batch`。
>
> **前置:** `WX_SERVER` 已设;按 SKILL.md §1 跑 probe。

## 三个子动作

### A. 单条 feed 详情(by oid/nid 或 url 或 eid)

```bash
# 已知 oid/nid
curl -fsS "$WX_SERVER/api/channels/feed/profile?oid=$OID&nid=$NID" | jq '.data.data.object'

# 已知 url(channels.weixin.qq.com 形式)
curl -fsS "$WX_SERVER/api/channels/feed/profile?url=$(jq -nr --arg u "$URL" '$u|@uri')" | jq '.data.data.object'

# 已知 eid(encrypted_objectid)
curl -fsS "$WX_SERVER/api/channels/feed/profile?eid=$EID" | jq '.data.data.object'
```

`oid/nid/url/eid` 任一即可;若 url 中含 `?eid=...`,服务端会自动提取并清空 url(`handler.go:212-219`)。

### B. 直播回放列表

```bash
USERNAME="<v2_xxx@finder>"
NEXT=""
RESP=$(curl -fsS "$WX_SERVER/api/channels/live/replay/list?username=$(jq -nr --arg u "$USERNAME" '$u|@uri')&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')")
echo "$RESP" | jq '.data.data.object'
NEXT=$(echo "$RESP" | jq -r '.data.data.lastBuffer // ""')
```

### C. 已互动 feed 列表

```bash
FLAG=""              # 留空走上游默认 tabFlag=7;取值见下方"字段"节
NEXT=""
RESP=$(curl -fsS "$WX_SERVER/api/channels/interactioned/list?flag=$FLAG&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')")
echo "$RESP" | jq '.data.data.object'
NEXT=$(echo "$RESP" | jq -r '.data.data.lastBuffer // ""')
```

## 分页范式(B 与 C 共用)

| 情况 | 字段 |
|---|---|
| 起始 | `next_marker=""` |
| 下一页入参 | `.data.data.lastBuffer` 当下次 query 的 `next_marker` |
| 是否还有下页 | `.data.data.continueFlag == 1` |
| 列表项 | `.data.data.object[]`(`ChannelsObject` 数组,字段同 [`creator-batch.md`](creator-batch.md)) |

## 字段

### A 请求(query,源:`internal/api/handler.go:206-226`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `oid` | 与 `nid/url/eid` 至少一个 | object id |
| `nid` | 同上 | object nonce id |
| `url` | 同上 | 视频号 URL(可含 `eid` query) |
| `eid` | 同上 | encrypted_objectid |

### A 响应(`ChannelsFeedProfileResp`,源:`types/types.go:287-316`)

经 `result.Ok` 包装 → 嵌套两层 `.data.data`。常用子集:

| 字段 | 含义 |
|---|---|
| `.data.data.object.id` | feed id(同 oid) |
| `.data.data.object.objectNonceId` | feed nonce |
| `.data.data.object.objectDesc.description` | 标题 / 描述 |
| `.data.data.object.objectDesc.media[0].url + .urlToken` | media URL(可拼成下载 url) |
| `.data.data.object.objectDesc.media[0].decodeKey` | 解密 key(string) |
| `.data.data.object.objectDesc.media[0].spec[]` | 清晰度选项 |
| `.data.data.object.contact.username/nickname/headUrl` | 创作者 |

### B 请求(query,源:`internal/api/handler.go:53-62` `handleFetchLiveReplayList`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `username` | 是 | 创作者 username;服务端自动补 `@finder` |
| `next_marker` | 否 | 分页游标 |

### B 响应

同 [`creator-batch.md`](creator-batch.md) `ChannelsFeedListOfAccountResp` —— 嵌套 `.data.data.object[]`(`ChannelsObject` 数组),分页用 `.data.data.lastBuffer` + `.data.data.continueFlag`。

### C 请求(query,源:`internal/api/handler.go:64-73` `handleFetchInteractionedFeedList`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `flag` | 否 | 上游 `tabFlag`(int 字符串)。留空时 inject 端默认 `7`(源:`internal/interceptor/inject/src/downloader.js:417-420`)。具体 tab 取值由微信 PC 端 UI 决定,本仓库未枚举 |
| `next_marker` | 否 | 分页游标 |

### C 响应

同 B,嵌套 `.data.data.object[]`。

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `不合法的参数` / `missing url` / 上游错误描述 | A:至少传一个 oid/nid/url/eid;B:username 必传;C:多看上游日志 |
| 反复同错 | — | 走 `+probe`,确认 channels 子系统 `available == true` |

## 链接

- [`download-by-url.md`](download-by-url.md) — 拿到详情后想下载它走这里
- [`creator-batch.md`](creator-batch.md) — 拿到 list 后批量下载
- [`precondition-probe.md`](precondition-probe.md)
