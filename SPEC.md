**English** | [简体中文](./SPEC.zh-CN.md)

# DingTalk Channel SDK — Unified Contract Across Four Languages (SPEC v0.1)

> Positioning: **Conversation access layer decoupled from Agent runtime**. The SDK handles the "plumbing" work,
> developers only write "what the user said, what the bot replies".

## 0. Effect Parity Acceptance Checklist (Effect Parity — Acceptance Criterion for All Four Languages)

Acceptance by **end-user perceivable behavior**, demonstrable line-by-line:

| # | User-Visible Effect | This SDK Implementation |
|---|---|---|
| E1 | "Inputing" card (loading) appears **within seconds** after sending message | `reply.stream()` creates card + delivers INPUTING immediately (doesn't wait for first token) |
| E2 | Reply content appends **typewriter-style** smoothly | streaming interface + 800ms throttle + trailing newline removal for non-final frames (prevents flicker) |
| E3 | After completion, loading disappears, content **freezes as complete Markdown** | isFinalize final frame + flowStatus=3 + cardUpdateOptions |
| E4 | Card failure/QPS rate limit **transparent to user** | Create failure→silent fallback to webhook text; QpsLimit→backoff 2s retry; no error popup |
| E5 | **Group/DM same experience** | Same Reply API; delivery target auto-selects IM_GROUP/IM_ROBOT; group auto-strips @ prefix |
| E6 | **Never duplicate reply** to same message | Dual-layer deduplication (messageId+msgId, TTL 5min), discard but still ACK |
| E7 | **Card interaction closed loop**: button clicks reach Agent, can update card | OnCardAction (auto-subscribes /v1.0/card/instances/callback upon registration, all four languages have unit test coverage for dispatch and subscription) + reply update |
| E8 | Bot **never goes offline** (network disconnect/server switch transparent) | Exponential backoff reconnect + SYSTEM/disconnect immediate reconnect + heartbeat keep-alive + ACK prevents loss |
| E9 | **Media send/receive**: received images/files downloadable; can upload media and embed | `reply.image(url)` (sampleImageMsg); `reply.uploadMedia()` (OAPI multipart, mediaId can `![..](mediaId)` embed in card); `reply.downloadURL()` |
| E10 | Markdown **rendering quality** (code blocks/tables/lists/quotes) | normalizeForCard normalization (see §7) |

> Effect demo script (attached to each language's README): echo + streaming simulation (mock LLM emits token every 100ms) must present complete E1→E3 process.

## 1. Responsibility Boundaries

**SDK Responsible For:**
1. Stream persistent connection (connect, subscribe, heartbeat, disconnect-reconnect, server disconnect handling)
2. Event parsing and deduplication (protocol layer messageId + business layer msgId, dual-layer, TTL 5 minutes)
3. Reply sending (sessionWebhook: text / Markdown)
4. AI card streaming output (create → deliver → INPUTING → streaming → FINISHED, typewriter effect)
5. Card API global rate limiting (token bucket + QpsLimit backoff retry)
6. Markdown normalization (adapt to DingTalk AI card renderer's newline/table rules)

**SDK Not Responsible (Left to Agent Side):**
- Agent runtime (model / prompt / tool orchestration)
- Multi-user topic isolation and Session/context persistence
- Credential storage (only receives clientId/clientSecret)

## 2. Wire Protocol (Stream Mode)

### 2.1 Establish Connection
```
POST {apiBase}/v1.0/gateway/connections/open
{
  "clientId": "...", "clientSecret": "...",
  "ua": "dingtalk-channel-sdk-{lang}/v0.1.0",
  "localIp": "<first non-loopback IPv4>",
  "subscriptions": [ {"type": "CALLBACK", "topic": "/v1.0/im/bot/messages/get"} ],
  "extras": {}
}
→ {"endpoint": "wss://...", "ticket": "..."}
```
HTTP headers: `Content-Type/Accept: application/json`, `User-Agent: dingtalk-channel-sdk-{lang}/v0.1.0`.
WebSocket connection: `{endpoint}?ticket=<url-encoded ticket>` (**ticket requires URL encoding**, same as official Python SDK; topic declared in open request, not in URL).

Subscription types: `CALLBACK` (callback) / `EVENT` (event) / `SYSTEM` (system, SDK internally reserves `ping`, `disconnect`).

Fixed topics:
- Bot messages: `/v1.0/im/bot/messages/get` (CALLBACK)
- Card callbacks: `/v1.0/card/instances/callback` (CALLBACK, subscribed only when OnCardAction registered)

### 2.2 Data Frames
Inbound frame (WebSocket text):
```json
{"specVersion":"1.0","type":"CALLBACK|EVENT|SYSTEM","time":0,
 "headers":{"topic":"...","messageId":"...","contentType":"application/json","time":"..."},
 "data":"<JSON string>"}
```
ACK outbound frame (**must reply**, otherwise server re-delivers; **ACK first** — reply with `{"success":true}` immediately upon receipt, business processes async,
aligned with official connector: prevents server timeout re-delivery during Agent long tasks; duplicate delivery covered by dual-layer deduplication):

> **Server-side perspective evidence** (lippi-open-proxy source code, 2026-07-23 production incident postmortem cross-validation):
> ① Server push is `UnaryRequest` — **synchronously waits for ACK**, upstream timeout ~2s; if not received, **re-delivers via MetaQ (at-least-once)**,
> and re-delivery generates new messageId (business layer msgId deduplication is necessary, protocol layer single-layer insufficient) — this SDK's ACK-first + dual-layer dedup
> strictly aligns with this semantic. ② Heartbeat contract is **client ping, server auto pong** (gorilla default behavior); when server read loop blocks,
> pong stops, client should timeout reconnect — this SDK's 120s idle ping + 5s pong death detection is the client-side implementation of this contract.
> ③ Server ACK validation only requires headers non-empty + contains messageId; only receives/sends Text frames; ticket is server connectionId (URL query passed).
> ④ **Risk mitigation**: 0723 incident proved server read loop has mode where late/duplicate ACKs can block (fix is in 20260723 release branch,
> whether deployed to production subject to release system; local master is stale snapshot, does not represent production version). Regardless of whether server fix is in place,
> four client-side defenses are necessary for production self-healing: exactly-once ACK per frame, ACK-first (no late ACKs), pong timeout death reconnect
> (only exit when server wedged), exponential backoff+jitter reconnect (prevents storm amplification). Latency-sensitive scenarios can lower KeepAliveIdleMs
> (default 120s) to accelerate wedged detection.
```json
{"code":200,"headers":{"contentType":"application/json","messageId":"<same as frame>"},
 "message":"ok","data":"{\"success\":true}"}
```
- `SYSTEM/ping`: Reply pong, `data` echoed back.
- `SYSTEM/disconnect`: Close connection and immediately reconnect (server LB switch).

### 2.3 Heartbeat and Reconnect
- Send WebSocket protocol-layer Ping after 120s idle, declare dead if no Pong within 5s.
- Reconnect: exponential backoff 1s→2s→4s…capped at 30s (with jitter); reconnect success resets to zero.
- Read loop exception/disconnection → auto-reconnect (configurable AutoReconnect=false).

> Comparison with official stream-sdk family: official heartbeat Go=120s idle+5s pong, Java(Netty)=60s idle+pong,
> Python=60s ping (no pong detection), Node=isAlive flag+terminate; this SDK uniformly adopts Go tier (120s+5s pong),
> stronger than Python/Node. Official reconnect is fixed 3s/10s, this SDK uses exponential backoff+jitter (aligned with official connector).

## 3. Event Model

### 3.1 IncomingMessage (After Normalization)
| Field | Source | Description |
|---|---|---|
| ConversationID | conversationId | Conversation ID |
| ConversationType | conversationType | "1"=DM "2"=group → normalized to `dm`/`group` |
| ConversationTitle | conversationTitle | Group name (group chat) |
| SenderID / SenderStaffID | senderId / senderStaffId | Encrypted ID / staff ID |
| SenderNick | senderNick | Nickname |
| SenderCorpID | senderCorpId | |
| Text | text.content | **@bot prefix and whitespace removed and trimmed** |
| MsgType / Content | msgtype / content | Rich content (images/files etc. transparently passed through) |
| AtUsers | atUsers[] | [{dingtalkId, staffId}] |
| SessionWebhook | sessionWebhook | Reply webhook (includes SessionWebhookExpiredTime) |
| MsgID / CreateAt | msgId / createAt | Business dedup key / event time |
| Raw | original data | |

### 3.2a Stale Message Filtering

Inbound messages with `createAt` exceeding `StaleMessageWindow` (default 30min, <=0 disables) from now are directly discarded (still ACK) —
stale messages flooding in during reconnect storms/re-delivery backlog no longer trigger replies.
1. Protocol layer: `headers.messageId` (duplicate callbacks of same delivery)
2. Business layer: `data.msgId` (messageId changes on server re-send, msgId doesn't change)
TTL 5 minutes, LRU cleanup. Hit results in discard (still ACK success).

## 4. Reply API (Four-Language Consistent Semantics)

```
reply.text(content)                     → sessionWebhook, msgKey=sampleText
reply.markdown(title, text)             → sessionWebhook, msgKey=sampleMarkdown
reply.image(url)                        → sessionWebhook, msgKey=sampleImageMsg
reply.downloadURL(downloadCode,msgId)   → GET /v1.0/robot/messageFiles/download
reply.uploadMedia(type,name,data[,ct]) → OAPI upload, returns mediaId (see §9a)
s = reply.stream()                      → Creates AI card immediately (E1: "inputing" card first)
s.append(delta) / s.append(fullText)    → Streaming update (accumulation semantics decided by caller)
s.finish() / s.finish(fullText)         → Final frame + FINISHED
s.fail(errText)                         → FINISHED(flowStatus=5) or fallback text
```

**Oversized chunking**: `TextChunkLimit` (default 3500, <=0 disables) —
text/Markdown reply exceeding limit is split by **newline boundary** into multiple sends (hard-cut if no suitable newline), content not lost.

sessionWebhook payload:
```json
{"msgKey":"sampleText","msgParam":"{\"content\":\"...\"}"}
{"msgKey":"sampleMarkdown","msgParam":"{\"title\":\"...\",\"text\":\"...\"}"}
```
**Note**: `msgParam` must be **stringified JSON** (official docs requirement, object form results in 400).
Header: `x-acs-dingtalk-access-token: <token>`.

## 4a. Proactive Messaging (Proactive Send)

Independent of inbound messages, Agent can initiate anytime:

```
channel.SendText(target, text)
channel.SendMarkdown(target, title, text)      // target can carry @: AtUserIds/AtDingtalkIds/AtAll
channel.SendImage(target, imageURL)            // Requires publicly accessible URL
```

- DM (target.UserID): `POST /v1.0/robot/oToMessages/batchSend` `{robotCode, userIds:[...], msgKey, msgParam}`
- Group (target.ConversationID): `POST /v1.0/robot/groupMessages/send` `{robotCode, openConversationId, msgKey, msgParam, atUserIds?, atOpendingtalkIds?, isAtAll?}`

## 4b. Policy Gates and Group-Level Overrides

Global policy (PolicyConfig): group allow/blocklist, `RequireMention` (default true), DM mode
(open/allowlist/blocklist/disabled) with corresponding lists.

**Group-level overrides (GroupOverrides)**: Override per-group by conversationId — `Enabled` (explicitly disable),
`RequireMention` (@ requirement for this group), `AllowFrom`/`BlockFrom` (sender allow/blocklist within group, blocklist takes priority).
Evaluation order (consistent across four languages):

1. Global blocklist (highest priority, group overrides cannot exempt)
2. Allowlist admission: global allowlist hit, **or explicit group entry exists** (explicit entry can allow this group in allowlist mode)
3. `Enabled=false` → reject (`group_disabled`)
4. @bot check (group override takes priority over global)
5. `BlockFrom` → `AllowFrom` (sender filtering within group)

## 5. AI Card Protocol (Five Steps)

Template ID default: `02fcf2f4-5e02-4a85-b672-46d1f715543e.schema` (official AI card, configurable).

1. **Create** `POST /v1.0/card/instances`
   `{cardTemplateId, outTrackId: "card_{ts}_{rand}", cardData:{cardParamMap:{config:"{\"autoLayout\":true}"}}, callbackType:"STREAM", imGroupOpenSpaceModel:{supportForward:true}, imRobotOpenSpaceModel:{supportForward:true}}`
2. **Deliver** `POST /v1.0/card/instances/deliver`
   - Group: `{outTrackId, userIdType:1, openSpaceId:"dtv1.card//IM_GROUP.{conversationId}", imGroupOpenDeliverModel:{robotCode}}`
   - DM: `{outTrackId, userIdType:1, openSpaceId:"dtv1.card//IM_ROBOT.{senderStaffId||senderId}", **imRobotOpenDeliverModel**:{spaceType:"IM_ROBOT", robotCode, extension:{dynamicSummary:"true"}}}`
   - robotCode = clientId
   - ⚠️ DM field must be `imRobotOpenDeliverModel` (Deliver not Space; official connector's `imRobotOpenSpaceModel` variant is rejected by production: `400 param.spaceDeliverModelEmpty` — 2026-08 real device evidence, dws source code is authoritative)
   - ⚠️ **Business-level validation**: deliver returns `{"result":[{"success":false,...}]}` inside HTTP 200 (dws production evidence "observed live"), SDK must scan body for `"success":false` and treat as failure (this SDK's create/deliver both do callChecked)
3. **First frame set INPUTING** `PUT /v1.0/card/instances`
   `{outTrackId, cardData:{cardParamMap:{flowStatus:"2", msgContent:<norm>, staticMsgContent:"", sys_full_json_obj:"{\"order\":[\"msgContent\"]}", config:"{\"autoLayout\":true}"}}}`
4. **Streaming update** `PUT /v1.0/card/streaming`
   `{outTrackId, guid:"{ts}_{rand}", key:"msgContent", content:<norm>, isFull:true, isFinalize:<bool>, isError:false}`
   Non-final frames strip trailing consecutive newlines (prevents flicker).
5. **Finalize FINISHED** `PUT /v1.0/card/instances`
   First send isFinalize=true streaming, then set `{outTrackId, cardData:{cardParamMap:{flowStatus:"3", msgContent, ...}}, cardUpdateOptions:{updateCardDataByKey:true}}`

flowStatus: 1=PROCESSING 2=INPUTING 3=FINISHED 4=EXECUTING 5=FAILED.

**Frame rhythm (dws connect_card.go evidence ported)**:
- **Frame interval 500ms**: Must leave gap between first content frame and delivery, between final frame and previous frame — back-to-back competes with client card pull,
  intermittently renders "content load failed" (this race condition once killed dws #407 card implementation)
- **Single frame content limit 20000** (rune-safe truncation, hermes MAX_MESSAGE_LENGTH equivalent)

**Status badges (aligned with dws/hermes)**: `MarkThinking/MarkDone` marks text emotion on **user message**
(`POST /v1.0/robot/emotion/reply|recall`, emotionType=2, emotionId=2659900):
"🤔Thinking" indicates processing, "🥳Done" indicates completion; only supports user-sent messages (bot messages get 500); best-effort doesn't block reply.

**Throttling**:
- Single card streaming update minimum interval 800ms (DingTalk card has same-card concurrency protection, official connector battle-tested value)
- **Updates within window not discarded**: Arrange trailing flush (`delay = throttle - elapsed`), content ultimately delivered, avoids "output ends at window tail → screen stuck until finish"
- **Long interval batching**: After >2s no update (tool invocation/thinking gap), first flush delayed 300ms for batching, first screen shows meaningful text not 1-2 characters
- flush and finish concurrency-safe: after closed, pending flush auto-voided
**Failure fallback**: create/deliver failure → silent fallback to sessionWebhook text; finish failure → fallback send accumulated text.

> **Template scope (dws A/B evidence)**: Card templates are app-scoped — hermes proprietary template (c629162a-...) renders "content load failed" for other apps;
> default template (02fcf2f4...) is openclaw connector's public template, can be used cross-app.
**Truth exposure**: `streamer.CardDelivered()/cardDelivered/card_delivered/cardDelivered()` returns whether card was actually delivered successfully (for diagnostics/livecheck, prevents fallback mode false positive as success).

**Three lines of defense**:
- **Watchdog** (`CardWatchdog`, default 10min): Timer starts after card established, success frame refreshes; timeout without finalization → force finish + seal —
  card won't spin forever when upstream Agent hangs/dispatch doesn't return (connector CARD_WATCHDOG_TIMEOUT equivalent)
- **Explicit abort** `Abort()/abort()`: External interruption scenario, seals stream + card set to FAILED,
  mutually exclusive and idempotent with Finish (normal finalization) / Fail (error message)
- **Error fallback cooldown** (`ErrorCooldown`, default 60s): Same session error fallback text sent only once within 60s, prevents error spam
  (connector deliveredErrorTypes+ERROR_COOLDOWN equivalent)

## 6. Rate Limiting (Global Token Bucket)

- Capacity/rate: default 20 QPS (official limit ~40, conservative value, configurable).
- Recognition: HTTP 403 and response body code string contains `QpsLimit`.
- Strategy: backoff 2s (drain tokens) → re-acquire token retry once; streaming retry with new guid.

## 7. Markdown Normalization (normalizeForCard)

DingTalk AI card renderer conventions (outside code blocks):
- Single `\n` → `<br>`; `\n\n` paragraph preserved
- Inside code block ```: preserve `\n`
- Markdown block syntax lines (list `- / 1.`, table `|`, heading `#`, separator) preserve preceding `\n`
- Consecutive quote lines `>`: merge into one line with `<br>` connection, continuation lines strip `>` prefix
- Insert empty line before table separator line if absent (otherwise won't render)

## 8. Token

`POST /v1.0/oauth2/accessToken` `{appKey, appSecret}` → `{accessToken, expireIn}`;
cached by clientId, refresh 60s before expiration. Header uniformly `x-acs-dingtalk-access-token`.

## 9a. Media Upload (OAPI, Compare with Official Connector media/common.ts)

1. **OAPI token**: `GET {oapiBase}/gettoken?appkey=&appsecret=` → `{errcode:0, access_token, expires_in}` (cached, refresh 60s early; oapiBase default `https://oapi.dingtalk.com`, mutually independent from new version API token)
2. **Upload**: `POST {oapiBase}/media/upload?access_token=&type={image|file|video|voice}`
   multipart/form-data, field name **`media`** (with filename), Content-Type image uses `image/jpeg`, others `application/octet-stream`
3. **Response**: `{errcode:0, media_id, type, created_at}`; **strip leading `@` from media_id** before use
4. **Media delivery capability matrix (2026-08 real device experiment settled)**:
   - OAPI `media/upload` produced mediaId **has no public URL** — `down.dingtalk.com/media/<id>` (including @ / with extension variants) **all 404** (freshly uploaded instant verification),
     therefore **card embedding OAPI-uploaded images not viable**; card embedding only works for **URLs already publicly accessible** (e.g., already in DingTalk media library)
   - **Reliable delivery = standalone media messages** (aligned with official connector sendVideo/sendAudio/sendFileProactive and dws current behavior — dws deprecated old upload command and clarified in migration notes "file message delivery, doesn't render inline image"):
     `SendFile` (sampleFile: mediaId+fileName+fileType), `SendVideo` (sampleVideo: videoMediaId+picMediaId+duration),
     `SendAudio` (sampleAudio: mediaId+duration) — all three require uploadMedia returned **RawMediaID (with @)**
   - `SendImage` (sampleImageMsg) only accepts public photoURL
   - >20MB files use chunked upload (v0.2 roadmap)

**Real integration (livecheck)**: Each language provides `example/livecheck` (Go: `go run ./example/livecheck`;
Node: `npm run live`; Python: `python example/livecheck.py`;
Java: `mvn -q compile exec:java -Dexec.mainClass=...LiveCheck`).
Set `DD_CLIENT_ID/DD_CLIENT_SECRET` (optional `DD_UPLOAD_FILE`) then send bot a message,
progressive PASS/FAIL: connect → receive message → text reply (token) → card create (E1) → streaming full cycle (E2/E3) → media upload.

## 9. Four-Language API Comparison

| | Go | Node.js | Python | Java |
|---|---|---|---|---|
| Create | `channel.New(cfg)` | `new DingTalkChannel(cfg)` | `DingTalkChannel(cfg)` | `DingTalkChannel.create(cfg)` |
| Receive message | `ch.OnMessage(func(ctx, msg, reply))` | `ch.on('message', async (msg, reply) => {})` | `@ch.on_message` / `ch.on_message(fn)` | `ch.onMessage((msg, reply) -> {})` |
| Card callback | `ch.OnCardAction(...)` | `ch.on('cardAction', ...)` | `ch.on_card_action(fn)` | `ch.onCardAction(...)` |
| Start | `ch.Start(ctx)` | `await ch.start()` | `await ch.start()` | `ch.start()` / `startAsync()` |
| Streaming reply | `st, _ := reply.Stream(); st.Append/Finish` | `const s = reply.stream(); await s.append/finish` | `s = await reply.stream(); await s.append/finish` | `CardStreamer s = reply.stream(); s.append/finish` |

## Directory Structure (Four-Layer Packages)

```
src/
├── (root)             Public API and assembly: channel config stream frame token card
│                      reply send lifecycle bot-identity emotion ratelimit
│                      http-mode media errors index (re-export)
├── normalize/         Inbound normalization: message
├── safety/            Admission and stability: policy dedup processing-lock chat-queue
│                      batching ssrf-guard
└── outbound/          Outbound processing: retry splitter markdown (card render preprocessing)
```
Dependency direction strictly unidirectional: root → sub-packages; index.js re-exports to keep package entry API unchanged.

## 10. Version and Naming

- Repository: `dingtalk-channel-sdk-{go,nodejs,python,java}` (aligned with dingtalk-stream-sdk-* family)
- UA: `dingtalk-channel-sdk-{lang}/v0.1.0`
- License: MIT
