# +search(搜创作者)

> **场景:** 用户给一个昵称或关键词,要找到对应的视频号 username 以喂给 `+batch` / `+list`。
>
> **前置:** `WX_SERVER` 已设;按 SKILL.md §1 跑 probe;关键词非空字符串。

## 完整链路

```bash
KEYWORD="<关键词>"
NEXT=""
RESP=$(curl -fsS "$WX_SERVER/api/channels/contact/search?keyword=$(jq -nr --arg k "$KEYWORD" '$k|@uri')&next_marker=$(jq -nr --arg n "$NEXT" '$n|@uri')")
echo "$RESP" | jq -e '.code == 0' >/dev/null || { echo "$RESP" | jq -r '.msg'; exit 1; }

# 列表项在 .data.data.infoList[].contact
echo "$RESP" | jq '.data.data.infoList[] | {
  username: .contact.username,
  nickname: .contact.nickname,
  signature: .contact.signature,
  avatar: .contact.headUrl
}'

# 翻页 token:.data.data.lastBuff(注意是 lastBuff,没有 er)
NEXT=$(echo "$RESP" | jq -r '.data.data.lastBuff // ""')
HAS_MORE=$(echo "$RESP" | jq -r '.data.data.continueFlag // 0')
[ "$HAS_MORE" = "1" ] && [ -n "$NEXT" ] && echo "下一页 next_marker=$NEXT"
```

## 分页范式

| 情况 | 字段 |
|---|---|
| 下一页入参 | 把上一次响应的 `.data.data.lastBuff` 当下一次 query 的 `next_marker` |
| 是否还有下页 | `.data.data.continueFlag == 1` 表示有,`0` 表示没 |
| 注意 | search 用 `lastBuff`(没 `er`),与 feed list 的 `lastBuffer` 字段名不同 |

## 字段

### 请求(query,源:`internal/api/handler.go:32-41` `handleSearchChannelsContact`)

| 字段 | 必填 | 说明 |
|---|---|---|
| `keyword` | 是 | 搜索关键词,UTF-8 |
| `next_marker` | 否 | 分页游标,首页空字符串;下一页传上次响应的 `.data.data.lastBuff` |

### 响应(源:`internal/api/types/types.go:175-194` `ChannelsContactSearchResp`)

实际响应是上游微信前端的原始包装,经服务端 `result.Ok` 再包一层 `{code,msg,data}` —— **嵌套两层 `.data.data`**:

```json
{
  "code": 0, "msg": "ok",
  "data": {
    "errCode": 0,
    "errMsg": "",
    "data": {
      "infoList": [<InfoListItem>, ...],
      "continueFlag": 0,
      "lastBuff": "..."
    },
    "payload": {"query":"...", "lastBuff":"...", ...}
  }
}
```

`InfoListItem` 字段(源:`types/types.go:139-146`):

| 字段 | 类型 | 含义 |
|---|---|---|
| `.contact.username` | string | 创作者 username(`v2_xxx@finder`),喂给 `+batch` / `+list` |
| `.contact.nickname` | string | 昵称 |
| `.contact.headUrl` | string | 头像 URL |
| `.contact.signature` | string | 个人签名 |
| `.contact.coverImgUrl` | string | 封面背景图 URL |
| `.highlightNickname` | string | 命中关键词高亮的昵称(可能含 `<em>` HTML 片段) |
| `.highlightSignature` | string | 命中签名高亮 |
| `.highlightProfession` | string | 命中职业字段高亮 |
| `.friendFollowCount` | int | 共同关注数 |
| `.reqIndex` | int | 上游请求内部索引 |

## 出错时

| code | 典型 msg | 修法 |
|---|---|---|
| 400 | `missing keyword` / 上游 `RequestFrontend` 错误描述 | keyword 必传;若 msg 是 timeout / 上游拒绝,稍后重试或换关键词 |
| 反复同错 | — | 走 `+probe`,确认 channels 子系统 `available == true` |

## 链接

- [`creator-batch.md`](creator-batch.md) — 搜到 username 后批量下载
- [`list-feed.md`](list-feed.md) — 搜到 username 后只列 feed
- [`precondition-probe.md`](precondition-probe.md)
