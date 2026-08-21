**English** | [简体中文](./GUIDE.zh-CN.md)

# DingTalk Channel SDK Integration Guide

Connect your Agent to DingTalk for real-time conversations in **group chats and direct messages**. The SDK handles event integration, message parsing and deduplication, reply sending, streaming output (typewriter effect), media upload/download, and card interactions — you just tell it "what the user said, what the bot replies".

## Usage Effects

Using a customer service Agent as an example, after integration you get three out-of-the-box capabilities:

- **Streaming replies**: "Inputing" card appears within seconds after user question, answer appends character by character as LLM generates, Markdown freezes after completion
- **Card interactions**: When card buttons sent by bot are clicked, events return to your Agent to continue conversation or update card
- **Proactive notifications**: Without depending on user initiation, Agent can send messages to individuals/groups anytime (supports @)

## What Channel SDK Does

| Without SDK (DIY) | With Channel SDK |
|---|---|
| Research Stream protocol, WebSocket connection, heartbeat, disconnect-reconnect | `channel.New(Config{...})` one line, connection self-healing |
| Parse raw callback frames, field mapping, prevent re-delivery | Unified `IncomingMessage` / `CardAction`, dual-layer deduplication |
| Interface with AI card create/deliver/streaming update/finalize four APIs | `reply.Stream()` + `Append()` auto-refresh (throttle + backfill) |
| Handle rate limiting, failure fallback, group/DM differences | Built-in: QpsLimit backoff retry, failure fallback text, target auto-selection |

## SDK Integration (Five Phases, SDK Handles Four)

1. **Transport connection**: Stream persistent connection establishment, subscription, heartbeat keep-alive, disconnect exponential backoff reconnect, server disconnect self-healing — *SDK built-in*
2. **Event transformation**: Raw callback frames normalized to unified structures (`IncomingMessage`, `CardAction`) — *SDK built-in*
3. **Reply strategy**: Message deduplication (protocol layer + business layer), same session serialization, group @ stripping, card failure fallback — *SDK built-in*
4. **Business dispatch**: Register your handler with `OnMessage / on('message') / @on_message` — **The only phase you need to write**
5. **Outbound rendering**: Agent output converted to DingTalk messages/AI cards, includes streaming refresh, throttling, final frame freeze, media embedding — *SDK built-in*

## Multi-Language SDKs

| Language | Repository | Installation |
|---|---|---|
| Go | [dingtalk-channel-sdk-go](https://github.com/DingTalk-Real-AI/dingtalk-channel-sdk-go) | `go get github.com/DingTalk-Real-AI/dingtalk-channel-sdk-go` |
| Node.js | [dingtalk-channel-sdk-nodejs](https://github.com/DingTalk-Real-AI/dingtalk-channel-sdk-nodejs) | `npm install dingtalk-channel-sdk` |
| Python | [dingtalk-channel-sdk-python](https://github.com/DingTalk-Real-AI/dingtalk-channel-sdk-python) | `pip install dingtalk-channel-sdk` |
| Java | [dingtalk-channel-sdk-java](https://github.com/DingTalk-Real-AI/dingtalk-channel-sdk-java) | Maven `com.dingtalk:dingtalk-channel-sdk` |

## Prerequisites (One-Time Setup)

1. [DingTalk Developer Console](https://open-dev.dingtalk.com) create **enterprise internal application**
2. Add **bot** capability within application, record **ClientID (AppKey) / ClientSecret (AppSecret)**
3. No public IP required, no domain required, no webhook required (Stream persistent connection mode)

## Quick Start (Same Example · Complete Four-Language Comparison)

Same logic: receive message → streaming card reply (typewriter) → card button handling → blocking run.
Each snippet is a complete, copyable example; steps correspond one-to-one across the four languages.

### Go

```go
ch := channel.New(channel.Config{
    ClientID:     os.Getenv("DD_CLIENT_ID"),
    ClientSecret: os.Getenv("DD_CLIENT_SECRET"),
})

ch.OnMessage(func(ctx context.Context, msg *channel.IncomingMessage, reply channel.Reply) error {
    if msg.Text == "" {
        return nil // Non-text messages in msg.Content / msg.MsgType
    }
    s, _ := reply.Stream(ctx)                  // ① "Inputing" card appears immediately
    answer := myAgent(ctx, msg.Text)           // ② Your Agent
    for _, tok := range streamTokens(answer) { // ③ Streaming append (throttle + backfill built-in)
        _ = s.Append(tok)
    }
    return s.Finish(answer)                    // ④ Final frame freeze Markdown
})

ch.OnCardAction(func(ctx context.Context, a *channel.CardAction, reply channel.Reply) error {
    return reply.Text(ctx, "Button clicked: "+string(a.DataContent))
})

// Other reply methods
_ = reply.Markdown(ctx, "Title", "# Content")
_ = reply.Image(ctx, "https://.../a.png")
media, _ := reply.UploadMedia(ctx, "image", "a.png", "", imgBytes) // → ![..](media.MediaID) embed in card
url, _ := reply.DownloadURL(ctx, code, msg.MsgID)                  // Attachment download URL

// Proactive messaging (independent of inbound)
_ = ch.SendText(ctx, channel.SendTarget{UserID: "staff-1"}, "Proactive DM notification")
_ = ch.SendMarkdown(ctx, channel.SendTarget{ConversationID: "cid...", AtUserIds: []string{"u1"}}, "Daily Report", "@u1 Build completed")

log.Fatal(ch.Start(ctx)) // Blocking run, auto-reconnect on disconnect
```

### Node.js

```js
const ch = new DingTalkChannel({
  clientId: process.env.DD_CLIENT_ID,
  clientSecret: process.env.DD_CLIENT_SECRET,
});

ch.on('message', async (msg, reply) => {
  if (!msg.text) return; // Non-text messages in msg.content / msg.msgType
  const s = await reply.stream();              // ①
  const answer = await myAgent(msg.text);      // ②
  for await (const tok of streamTokens(answer)) {
    await s.append(tok);                       // ③
  }
  await s.finish(answer);                      // ④
});

ch.on('cardAction', async (action, reply) => {
  await reply.text('Button clicked: ' + JSON.stringify(action.dataContent));
});

// Other reply methods
await reply.markdown('Title', '# Content');
await reply.image('https://.../a.png');
const media = await reply.uploadMedia('image', 'a.png', imgBytes); // → ![..](media.mediaId)
const url = await reply.downloadURL(code, msg.msgId);

// Proactive messaging
await ch.sendText({ userId: 'staff-1' }, 'Proactive DM notification');
await ch.sendMarkdown({ conversationId: 'cid...', atUserIds: ['u1'] }, 'Daily Report', '@u1 Build completed');

const controller = new AbortController();
process.on('SIGINT', () => controller.abort());
await ch.start(controller.signal); // Blocking run, auto-reconnect on disconnect
```

### Python

```python
ch = DingTalkChannel(
    client_id=os.environ["DD_CLIENT_ID"],
    client_secret=os.environ["DD_CLIENT_SECRET"],
)

@ch.on_message
async def handle(msg, reply):
    if not msg.text:
        return  # Non-text messages in msg.content / msg.msg_type
    s = await reply.stream()               # ①
    answer = await my_agent(msg.text)      # ②
    for tok in stream_tokens(answer):
        await s.append(tok)                # ③
    await s.finish(answer)                 # ④

@ch.on_card_action
async def on_card(action, reply):
    await reply.text(f"Button clicked: {action.data_content}")

# Other reply methods
await reply.markdown("Title", "# Content")
await reply.image("https://.../a.png")
media = await reply.upload_media("image", "a.png", img_bytes)  # → ![..](media["mediaId"])
url = await reply.download_url(code, msg.msg_id)

# Proactive messaging
await ch.send_text(SendTarget(user_id="staff-1"), "Proactive DM notification")
await ch.send_markdown(SendTarget(conversation_id="cid...", at_user_ids=["u1"]), "Daily Report", "@u1 Build completed")

asyncio.run(ch.start())  # Blocking run, auto-reconnect on disconnect
```

### Java

```java
DingTalkChannel ch = DingTalkChannel.create(Config.builder(
        System.getenv("DD_CLIENT_ID"), System.getenv("DD_CLIENT_SECRET")).build());

ch.onMessage((msg, reply) -> {
    if (msg.text.isEmpty()) return;          // Non-text messages in msg.content / msg.msgType
    CardStreamer s = reply.stream();         // ①
    String answer = myAgent(msg.text);       // ②
    for (String tok : streamTokens(answer)) {
        s.append(tok);                       // ③
    }
    s.finish(answer);                        // ④
});

ch.onCardAction((action, reply) ->
        reply.text("Button clicked: " + action.dataContent));

// Other reply methods (within handler)
reply.markdown("Title", "# Content");
reply.image("https://.../a.png");
OapiClient.MediaUploadResult media = reply.uploadMedia("image", "a.png", "", imgBytes); // → ![..](media.mediaId)
String url = reply.downloadUrl(code, msg.msgId);

// Proactive messaging
ch.sendText(SendTarget.user("staff-1"), "Proactive DM notification");
ch.sendMarkdown(SendTarget.group("cid...").atUserIds("u1"), "Daily Report", "@u1 Build completed");

Runtime.getRuntime().addShutdownHook(new Thread(ch::close));
ch.start(); // Blocking run, auto-reconnect on disconnect
```

### API Quick Reference

| Capability | Go | Node.js | Python | Java |
|---|---|---|---|---|
| Create | `channel.New(cfg)` | `new DingTalkChannel(cfg)` | `DingTalkChannel(...)` | `DingTalkChannel.create(cfg)` |
| Receive message | `ch.OnMessage(fn)` | `ch.on('message', fn)` | `@ch.on_message` | `ch.onMessage(fn)` |
| Card callback | `ch.OnCardAction(fn)` | `ch.on('cardAction', fn)` | `@ch.on_card_action` | `ch.onCardAction(fn)` |
| Start | `ch.Start(ctx)` | `await ch.start(signal)` | `await ch.start()` | `ch.start()` |
| Streaming reply | `reply.Stream(ctx)` → `s.Append/Finish` | `await reply.stream()` → `await s.append/finish` | `await reply.stream()` → `await s.append/finish` | `reply.stream()` → `s.append/finish` |
| Text/MD/Image | `reply.Text/Markdown/Image` | `reply.text/markdown/image` | `reply.text/markdown/image` | `reply.text/markdown/image` |
| Media upload | `reply.UploadMedia` | `reply.uploadMedia` | `await reply.upload_media` | `reply.uploadMedia` |
| Attachment download | `reply.DownloadURL` | `reply.downloadURL` | `await reply.download_url` | `reply.downloadUrl` |
| Proactive messaging | `ch.SendText/SendMarkdown(SendTarget{...})` | `ch.sendText/sendMarkdown({userId\|conversationId, atUserIds})` | `ch.send_text/send_markdown(SendTarget(...))` | `ch.sendText/sendMarkdown(SendTarget.user/group)` |

## Verification: livecheck One-Click Integration

```bash
DD_CLIENT_ID=ding... DD_CLIENT_SECRET=... go run ./example/livecheck
# Then send a message to the bot in DingTalk; progressive PASS/FAIL: connect→receive message→text reply→card streaming full cycle→(optional DD_UPLOAD_FILE) media upload
```

Node: `npm run live`; Python: `python example/livecheck.py`; Java: `mvn -q compile exec:java -Dexec.mainClass=...LiveCheck`.

## Capability Boundaries (What SDK Does Not Do)

The following are left to your Agent side:

- **Agent runtime**: Model invocation, prompt, tool orchestration (SDK only handles channel)
- **Multi-user topic isolation**: Multi-session routing and isolation strategy
- **Session/context persistence**: Conversation history storage
- **Credential storage**: SDK only receives clientId/clientSecret, not responsible for safekeeping

## More

- Complete contract and effect acceptance checklist (E1–E10): [`SPEC.md`](./SPEC.md) in each repo
- Project overview: [`OVERVIEW.md`](./OVERVIEW.md) in each repo
- Complete API and examples for each language: README and `example/` in each repo

---
License: MIT ｜ v0.1.0
