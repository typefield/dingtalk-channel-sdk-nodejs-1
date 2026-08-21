**English** | [简体中文](./OVERVIEW.zh-CN.md)

# DingTalk Channel SDK Family · Project Overview

> Origin issue: [DingTalk-Real-AI/dingtalk-workspace-cli#796](https://github.com/DingTalk-Real-AI/dingtalk-workspace-cli/issues/796) — "Will DingTalk provide an integrated SDK like Channel?"
> This project delivers the answer: **Four languages, effect parity, ready to use**.

## 1. Deliverables

| Repository | Language | Dependencies | Tests |
|---|---|---|---|
| [DingTalk-Real-AI/dingtalk-channel-sdk-go](https://github.com/DingTalk-Real-AI/dingtalk-channel-sdk-go) | Go 1.22+ | gorilla/websocket | 19 tests (race-clean) |
| [DingTalk-Real-AI/dingtalk-channel-sdk-nodejs](https://github.com/DingTalk-Real-AI/dingtalk-channel-sdk-nodejs) | Node 18+ | ws | 12 tests |
| [DingTalk-Real-AI/dingtalk-channel-sdk-python](https://github.com/DingTalk-Real-AI/dingtalk-channel-sdk-python) | Python 3.10+ | websockets | 12 tests |
| [DingTalk-Real-AI/dingtalk-channel-sdk-java](https://github.com/DingTalk-Real-AI/dingtalk-channel-sdk-java) | JDK 8+ | Java-WebSocket + Gson | 12 tests |

Each repository contains: complete source code, unit tests, `SPEC.md` (unified contract across four languages), streaming echo example, **livecheck real integration program**, README, and MIT LICENSE.

## 2. Positioning and Boundaries

**Conversation access layer decoupled from Agent runtime**. The SDK handles all the "plumbing" of the channel, while developers only write "what the user said, what the bot replies":

- **Responsibilities**: Stream persistent connection (connect/heartbeat/exponential backoff reconnect/server disconnect self-healing), dual-layer event deduplication + stale message filtering, sessionWebhook replies (text/Markdown/image, automatic chunking for oversized content), AI card streaming output (typewriter effect, frame interval race prevention, watchdog orphan protection), card API global rate limiting and QpsLimit backoff, media upload/download and media messages (file/video/audio), Markdown normalization, proactive messaging (DM/group + @), 🤔Thinking/🥳Done status badges, explicit abort, error fallback cooldown
- **Not Responsible**: Agent runtime (model/prompt/tool orchestration), conversation context persistence, credential storage, business operations (documents/tables/calendar — domain of dws CLI and skills)

## 3. Quick Start (Isomorphic Across Four Languages)

```go
ch := channel.New(channel.Config{ClientID: "ding...", ClientSecret: "..."})
ch.OnMessage(func(ctx context.Context, msg *channel.IncomingMessage, reply channel.Reply) error {
    s, _ := reply.Stream(ctx)              // "Inputing" card appears in seconds
    for _, tok := range myLLM(msg.Text) {
        _ = s.Append(tok)                  // Typewriter append (800ms throttle + trailing flush)
    }
    return s.Finish("")                    // Final frame freezes
})
ch.Start(ctx)
```

```js
ch.on('message', async (msg, reply) => { const s = await reply.stream(); ... await s.finish(); });
```
```python
@ch.on_message
async def handle(msg, reply): s = await reply.stream(); await s.append(tok); await s.finish()
```
```java
ch.onMessage((msg, reply) -> { CardStreamer s = reply.stream(); s.append(tok); s.finish(""); });
```

Non-streaming: `reply.Text/Markdown/Image`, attachment download `reply.DownloadURL`, media upload `reply.UploadMedia`;
Proactive messaging (independent of inbound): `ch.SendText/SendMarkdown/SendImage`, group send supports `AtUserIds/AtAll`;
Card interaction: `ch.OnCardAction` (auto-subscribes to card topic upon registration).

## 4. Effect Parity (Acceptance Checklist E1–E10, see SPEC §0)

| | User-Visible Effect | Implementation |
|---|---|---|
| E1 | "Inputing" card appears within seconds after sending message | `stream()` creates card + delivers INPUTING immediately |
| E2 | Typewriter-style smooth append | streaming interface + 800ms throttle + **trailing flush** (no loss within window) + 300ms batching for long intervals |
| E3 | Loading disappears after completion, Markdown freezes | isFinalize final frame + FINISHED status |
| E4 | Card failure/rate limit transparent to user | Silent fallback to webhook text; QpsLimit backoff 2s retry |
| E5 | Unified experience in group/DM | Same Reply API; delivery target auto-selected; group strips @ prefix |
| E6 | Never duplicate reply | messageId+msgId dual-layer deduplication (TTL 5min) |
| E7 | Card interaction closed loop | OnCardAction + auto-subscribe |
| E8 | Never goes offline | Exponential backoff reconnect + disconnect immediate reconnect + 120s/5s heartbeat + ACK first |
| E9 | Media send/receive | uploadMedia (OAPI multipart) / downloadURL / image reply |
| E10 | Markdown rendering quality | normalizeForCard (code blocks/tables/quotes DingTalk rendering rules) |

Each item has corresponding unit tests across all four languages; E8 has dedicated disconnect-reconnect e2e regression.

## 5. Protocol Fidelity (True Sources, Not Documentation Guesses)

| Capability | True Source |
|---|---|
| Stream wire protocol (open/wss/frame/ACK/heartbeat/topic constants) | Official dingtalk-stream-sdk-go source code line-by-line comparison |
| AI card five-step protocol + rate limiting + Markdown normalization | Official connector (dingtalk-openclaw-connector) card.ts |
| Token (new version/OAPI dual-track) and sessionWebhook payload | Official connector token.ts / messaging.ts + official docs validation |
| Proactive messaging API | dws (dingtalk-workspace-cli) source code |
| Ticket encoding / localIp / UA headers | Cross-comparison across four official stream SDKs |

Protocol-level issues fixed during review rounds: msgParam JSON stringification (official docs requirement), Go version disconnect mis-stop, ACK-first semantics, ticket URL encoding.

## 6. Key Architectural Decision: Why Custom Transport Layer Instead of Referencing Official stream-sdk

1. **Official connector doesn't even trust itself**: DingTalk official connector source code has `autoReconnect:false, keepAlive:false` fully disabled and rewritten (issues #571/#536/#573)
2. **Four-language consistency is this project's acceptance criterion**, while official four SDKs have inconsistent heartbeat/reconnect/encoding behaviors; referencing them inherits divergence
3. **Dependency weight**: Official Java version pulls Netty multi-modules, Python version bundles requests+aiohttp dual HTTP stacks; custom implementation has only 1-2 small dependencies per language, transport layer ~300 lines/language
4. Leaves evolution seam (`Channel → StreamConn → onFrame` single boundary), can add official SDK adapter backend when upstream matures

## 7. Quality Evidence

- **Tests**: Go 14 / Node 12 / Python 12 / Java 12 (BUILD SUCCESS), all include e2e (mock gateway + mock API) and disconnect-reconnect regression
- **Real integration**: Each language provides `example/livecheck` one-click verification (connect → receive message → text reply → card streaming full cycle → media upload, progressive PASS/FAIL):
  `DD_CLIENT_ID=... DD_CLIENT_SECRET=... go run ./example/livecheck` (Node `npm run live`; Python `python example/livecheck.py`; Java `mvn exec:java`)
- **Code review**: Three rounds (protocol consistency / effect parity / stream layer vs official SDK), fixed 8 issues, all with regression tests

## 8. Known Boundaries and Roadmap

- Real credential live run pending execution (livecheck ready, one command)
- >20MB file chunked upload (v0.2)
- Card template default value is connector built-in template, external release should emphasize configurability
- Platform differences (reaction/comments/forward) will follow DingTalk Open Platform evolution
- Optional: official stream-sdk adapter backend (seam left with `WithTransport`)

## 9. Prerequisites

Create **enterprise internal application** in DingTalk developer console and enable bot, obtain ClientID/ClientSecret. Stream mode requires no public IP or domain.

---
License: MIT ｜ Contract: `SPEC.md` in each repo ｜ Version: v0.1.0 (2026-08)
