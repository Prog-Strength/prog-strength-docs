# Streaming TTS — Lower Time-to-First-Speech for Voice Mode

**Status**: Implemented (pre-deploy) · **Last updated**: 2026-05-31

## Introduction

Voice mode works, but it feels pointless to use. The chat already streams Claude's reply token-by-token, so the user has the full written answer in roughly the time it takes them to glance at the screen. Then we wait. The client buffers the entire reply, fires a single `POST /speak` once Claude is done, OpenAI generates the mp3 of the *whole* response from scratch, the bytes come back, and only then does the user hear anything. End-to-end that's anywhere from ~5 to ~12 seconds after the user stops speaking — long enough that a conversational rhythm is impossible and you might as well read the text. Time-to-first-audio is the load-bearing UX number here, and right now it's bad.

The win is straightforward in principle: the agent doesn't actually have to wait for Claude to finish writing before TTS-ing the first sentence. As soon as the *first* sentence boundary lands in Claude's streamed output, TTS for just that sentence can fire. By the time Claude is mid-paragraph, the first sentence's audio is already playing. Subsequent sentences TTS in parallel as they complete and queue up behind the currently-playing clip. Time-to-first-audio drops from ~5–12s to ~1–2s, and the user gets a conversational pace.

The hard parts aren't streaming the audio itself — `/speak` already returns a complete mp3 per call — they're the **orchestration**: detecting sentence boundaries in a Markdown stream, queueing audio playback in order, cancelling cleanly when the user starts a new turn, and stripping Markdown markers (`*bold*`, code fences) before they get read aloud literally. And to know whether the change is actually working in production, we need a metric for time-to-first-audio on the Grafana agent dashboard with a threshold line, so a regression doesn't quietly slip back to today's behavior.

## Proposed Solution

**Server-side sentence detection + audio streaming, multiplexed onto the existing `/chat` SSE connection.**

The agent buffers the text deltas Anthropic is already streaming, runs a sentence-boundary regex + Markdown stripper inside the same goroutine that's handling the chat stream, and fires OpenAI TTS in parallel as each sentence completes. The resulting mp3 bytes (base64-encoded into a JSON envelope) get pushed back through the same SSE stream as a new event type — `audio_chunk` — alongside the `text_delta` events that already flow today. Clients receive both channels of events, render text deltas to the chat UI as before, and feed `audio_chunk` events into a tiny playback queue.

The architecture asks the agent to do more — text buffering, sentence detection, Markdown stripping, parallel TTS, and base64-encoded audio emission — and asks the clients to do dramatically less: no fetch orchestration, no boundary regex, no Markdown stripping, no AbortController tracking. Each client shrinks to "receive `audio_chunk`, decode, append to a `[Promise<Blob>]`-shaped queue, advance on `onEnded`, report TTFA once." The total client code probably drops from ~150+ lines (per client) to ~30–50.

Three big payoffs from putting this server-side:

1. **No duplication.** The boundary regex, Markdown stripper, queue ordering logic, and abort/cancel semantics live in exactly one place. Tweaking the abbreviation allowlist or adding support for a new Markdown form is a single PR, not three.
2. **Lower first-audio latency.** Client-orchestrated TTS adds a full HTTP round trip per sentence: client receives the delta, parses, posts to `/speak`, server hits OpenAI. Server-side detection skips the client → server hop entirely — boundary is found in the same process already streaming Anthropic's response, so the OpenAI call fires ~50–200ms earlier per sentence depending on network conditions. For the first sentence specifically — the metric we care about — this is meaningful.
3. **Simpler cancellation.** When the user sends a new message, the client closes the SSE connection. The agent's generator goroutine exits, which cancels all in-flight OpenAI calls automatically and drops the buffered state. No per-request AbortControllers on the client, no orphaned audio elements, no leaked blob URLs.

The wire protocol grows by one event type. `audio_chunk` carries `{index, text, mp3_base64}`: a monotonic index for ordering (SSE preserves order anyway but the explicit index makes debugging trivial), the original sentence text (useful for in-client captions or debug overlays), and the base64-encoded mp3 of that sentence. Base64 inflates the payload by 33%, which on a ~10KB per-sentence mp3 means ~14KB on the wire — fine for our scale. A typical 6-sentence reply is maybe 80–100KB of total audio bytes, which is well within reasonable for an SSE stream that's already going to have ~5KB of text deltas.

`ChatRequest` gets a `voice_mode: bool = false` field. When false, the server's behavior is unchanged — no buffering, no TTS, no `audio_chunk` events. When true, the new pipeline runs alongside the existing text streaming. The existing `/speak` endpoint stays as it is for one-shot non-streaming callers (the agent's `/title` flow doesn't use it; nothing else does either, but keeping it is cheap and gives us a fallback if something goes wrong with the new path).

For observability, a new `POST /telemetry/voice` endpoint on the agent accepts a single timing measurement per turn: `{session_id, time_to_first_audio_ms}`. The clients fire this once per turn, right after the first audio element starts playing. The agent records it as a Prometheus histogram (`agent_voice_time_to_first_audio_seconds`) which Grafana scrapes alongside the existing chat metrics. A new panel on the agent dashboard renders p50 + p95 with a horizontal threshold line at 2 seconds so a regression beyond that target is visible immediately.

## Goals and Non-Goals

### Goals

- Time-to-first-audio in voice mode drops from today's ~5–12s to ~1–2s for typical Claude responses. Measured client-side (end-to-end, including network + audio decode) and reported via the new telemetry endpoint.
- Subsequent sentences play without audible gaps for typical-length responses — one continuous spoken stream from the user's perspective.
- Server-side sentence detection, Markdown stripping, and TTS orchestration. Clients hold only an audio playback queue + telemetry POST.
- A new SSE event type `audio_chunk` flows through the existing `/chat` connection alongside `text_delta`. No new connection, no separate transport.
- A new `voice_mode: bool` field on `ChatRequest` gates the entire new pipeline server-side. `voice_mode=false` requests behave identically to today's `/chat`.
- A new Grafana panel on the agent observability dashboard plots `agent_voice_time_to_first_audio_seconds` (p50 + p95) over time with a 2-second horizontal threshold line.
- Markdown markers in the assistant's text (`*bold*`, ``inline code``, code fences, headers, link syntax) are stripped on the agent before being sent to TTS so the user doesn't hear "asterisk hello asterisk" read aloud.
- Mid-turn user interruption (new message sent, voice mode toggled off, page navigated away) cleanly cancels: closing the SSE connection tears down the agent's generator, which cancels all in-flight OpenAI calls. Clients clear their playback queue + stop the currently-playing audio.
- Both web and mobile clients ship the change. Web ships first since the Web Audio API is more mature; mobile follows with the same architecture using expo-audio and file-cached audio chunks.

### Non-Goals

- **Removing `/speak`.** The endpoint stays for one-shot non-streaming callers and as a fallback. We're adding a new path, not replacing the existing one. If after a quarter no clients are using `/speak`, deprecate it then.
- **Streaming bytes inside a sentence.** Each `audio_chunk` event carries a *complete* mp3 for one sentence. We do not split a sentence's mp3 across multiple chunks via MediaSource Extensions or chunked decoding. MSE on web is doable; expo-audio on mobile doesn't support it without third-party deps. Per-sentence granularity is the right level for v1.
- **Cross-sentence prosody.** Each `/speak` call internal to the agent is independent; TTS can't reach across sentence boundaries to adjust intonation. Acceptable tradeoff — the latency win is dramatic, the prosody hit is subtle.
- **Whole-message caching.** No memoization of "user asked X, agent said Y, here's the cached audio." Costs scale linearly with TTS char count and that's fine.
- **Client-side sentence detection.** Considered and rejected — see Proposed Solution. Duplicated logic, slower first audio, harder cancellation, more code per client.
- **Configurable threshold.** The 2-second Grafana threshold is hardcoded in the dashboard JSON. If it ever needs to change, edit the dashboard. No env-var tunability needed.
- **Voice toggle persistence.** Out of scope — voice mode is still per-session state, doesn't survive page refresh. The voice-chat SOW called that out as a non-goal and it remains one.

## Implementation Details

### Wire protocol — new SSE event

```json
{
  "type": "audio_chunk",
  "index": 0,
  "text": "You got this bro!",
  "mp3_base64": "SUQzBAAAAAAAI1RTU0UAAAAPAAA..."
}
```

- `index`: 0-based monotonic counter. SSE preserves order on the wire, but the explicit index simplifies debugging when reading the raw stream and lets the client log "received audio chunk 3" cleanly.
- `text`: the original sentence text (post-Markdown-stripping). Optional for playback; useful as a debug aid and for future caption overlays.
- `mp3_base64`: the base64-encoded mp3 bytes for this sentence's audio. Decoded client-side back into bytes for playback.

Event size scales with sentence length. A typical 10-word sentence yields ~8KB mp3 ≈ 11KB base64. A full turn is usually 4–8 sentences = ~50–100KB of audio across the stream. Fine for SSE; well below browser/proxy frame limits.

### Agent — `prog-strength-agent`

`ChatRequest` gains `voice_mode: bool = false`. The `_route_and_stream` function in `server.py` reads this flag and, when true, wraps the harness's stream output in a new `voice_streamer` that:

1. Buffers `text_delta` events' text content into a running string.
2. After each delta, scans the buffer for sentence boundaries (`[.!?]+\s` or end-of-stream).
3. For each completed sentence: strip Markdown, call `tts_generator.generate()` async (don't await — schedule the task), append the awaitable to an ordered list.
4. Async-iterate the awaitables in order, base64-encoding each completed mp3 and yielding it as an `audio_chunk` SSE event.
5. Continue passing `text_delta` events through unchanged so the existing UI text-rendering keeps working.

The pseudocode:

```python
async def voice_streamer(inner_stream):
    buffer = ""
    audio_tasks: list[asyncio.Task[bytes]] = []
    index = 0

    async for event_bytes in inner_stream:
        yield event_bytes  # pass-through to client (text_delta etc.)
        event = parse_sse(event_bytes)
        if event["type"] != "text_delta":
            continue
        buffer += event["text"]
        sentences, buffer = pop_complete_sentences(buffer)
        for sentence in sentences:
            cleaned = strip_markdown(sentence).strip()
            if not cleaned:
                continue
            task = asyncio.create_task(
                tts_generator.generate(user_id=user_id, text=cleaned, voice=None)
            )
            audio_tasks.append((index, cleaned, task))
            index += 1

    # End of Claude stream — flush any trailing buffer as a final sentence.
    if buffer.strip():
        cleaned = strip_markdown(buffer).strip()
        if cleaned:
            task = asyncio.create_task(
                tts_generator.generate(user_id=user_id, text=cleaned, voice=None)
            )
            audio_tasks.append((index, cleaned, task))

    # Drain audio tasks in order, yielding each as it completes.
    for i, text, task in audio_tasks:
        try:
            mp3 = await task
            yield sse({"type": "audio_chunk", "index": i, "text": text,
                       "mp3_base64": base64.b64encode(mp3).decode()})
        except Exception:
            log.exception("voice: TTS failed for sentence %d", i)
            # Skip this sentence; don't kill the whole turn.
```

Key wrinkle: the audio tasks fire as parallel coroutines, but yields are sequential. So sentence 2's TTS may finish before sentence 1's, and we deliberately hold sentence 2's audio until sentence 1 completes — preserves playback order. If sentence 1 fails, we skip its audio but still emit sentences 2, 3, 4 in order.

`strip_markdown` is a small regex pass:
- Strip fenced code blocks entirely (`````...`````)
- Strip inline code backticks but keep the content
- Strip `*` / `_` / `~` formatting markers but keep content
- Replace `[text](url)` → `text`
- Strip leading `#`/`>`/list markers from each line

`pop_complete_sentences(buffer)` returns `(completed: list[str], remainder: str)`. Heuristic: split on `[.!?]+\s+` but keep the punctuation with the sentence. A small abbreviation allowlist (`Dr.`, `Mr.`, `Mrs.`, `e.g.`, `i.e.`, `vs.`) prevents the most common false splits.

New endpoint:

```python
class VoiceTelemetryRequest(BaseModel):
    session_id: str | None = None
    time_to_first_audio_ms: float

@app.post("/telemetry/voice")
async def voice_telemetry(req: VoiceTelemetryRequest, request: Request) -> dict:
    auth = authenticate(request, config.jwt_signing_key)
    VOICE_TTFA_HISTOGRAM.labels(user_id=auth.user_id).observe(
        req.time_to_first_audio_ms / 1000.0
    )
    return {"ok": True}
```

`VOICE_TTFA_HISTOGRAM` is a Prometheus histogram in `telemetry.py` with buckets `(0.5, 1.0, 1.5, 2.0, 2.5, 3.0, 5.0, 8.0, 15.0)` seconds.

### Web client — `prog-strength-web`

Most of what's there for voice mode today gets deleted. Specifically, `playSpeech(text)` after the SSE stream completes and the `generateChatSpeech` helper go away — replaced with an inline playback queue:

```ts
// Inside the chat page's SSE event loop:
const audioQueue: { blob: Blob; index: number }[] = [];
let playing: HTMLAudioElement | null = null;
let firstAudioReported = false;
const turnStartMs = performance.now();

async function playNext() {
  const next = audioQueue.shift();
  if (!next) { playing = null; return; }
  const url = URL.createObjectURL(next.blob);
  const audio = new Audio(url);
  playing = audio;
  audio.onended = () => { URL.revokeObjectURL(url); playNext(); };
  if (!firstAudioReported) {
    firstAudioReported = true;
    const ms = performance.now() - turnStartMs;
    fetch(`${agentUrl}/telemetry/voice`, { /* ... */ body: JSON.stringify({ session_id, time_to_first_audio_ms: ms }) });
  }
  await audio.play();
}

// New event handler:
case "audio_chunk": {
  const bytes = Uint8Array.from(atob(event.mp3_base64), c => c.charCodeAt(0));
  const blob = new Blob([bytes], { type: "audio/mpeg" });
  audioQueue.push({ blob, index: event.index });
  if (!playing) playNext();
  break;
}
```

`voice_mode: true` is added to the chat request body when the voice toggle is on. The toggle is the only client-side awareness of voice mode the new code path requires.

Cancellation: closing the SSE reader (which the existing code already does on new turn / unmount) tears down the agent's generator, which cancels in-flight OpenAI calls server-side. Client-side: `playing?.pause()`, `audioQueue.length = 0`, revoke any outstanding blob URLs.

### Mobile client — `prog-strength-mobile`

Same architecture, expo-audio replaces `HTMLAudioElement`. Audio bytes can't be played directly from memory on RN — expo-audio's `Audio.Sound.createAsync({ uri })` requires a URI. So each `audio_chunk` gets written to a temp file in `cache/voice-<sessionId>-<index>.mp3`, the `Sound` is created from that URI, and `setOnPlaybackStatusUpdate` advances the queue on `didJustFinish`. Cache files get deleted after playback finishes.

The streamChat helper in `lib/stream.ts` already parses SSE events generically; the new `audio_chunk` event type lands in the same async iterator. The chat page handles it the same way as web — push to queue, advance on finish, fire telemetry on first play.

### Observability — `prog-strength-infra` monitoring/grafana

Existing agent dashboard (`monitoring/grafana/dashboards/agent.json`) gets one new panel:

- **Type**: time series
- **Title**: "Voice — time to first audio (p95)"
- **Queries**:
  - p95: `histogram_quantile(0.95, sum by (le) (rate(agent_voice_time_to_first_audio_seconds_bucket[5m])))`
  - p50: `histogram_quantile(0.50, sum by (le) (rate(agent_voice_time_to_first_audio_seconds_bucket[5m])))` — secondary series, fainter
- **Unit**: seconds
- **Threshold**: 2.0s, red dashed horizontal line
- **Legend**: "p95" and "p50"

The dashboard JSON edit is mechanical — copy an existing panel's shape, swap the queries and thresholds.

### Cancellation flow

Single-direction tear-down from the SSE close:

1. Client closes the SSE reader (user sent new message, toggled voice off, navigated away, switched session, etc.)
2. Agent's generator goroutine receives the closed-connection signal and exits
3. All in-flight `tts_generator.generate()` coroutines get cancelled by Python's asyncio.Task cancellation propagation
4. Open HTTP connections to OpenAI close, partial mp3 data is dropped
5. Client-side: any queued blobs get released, current `HTMLAudioElement` / expo-audio `Sound` gets stopped + freed

No per-request AbortControllers on the client, no orphaned audio elements. The whole thing rides on the single SSE connection's lifecycle.

## Open Questions

1. **Threshold value on the Grafana line.** 2 seconds is the lean — fast enough to feel conversational, achievable based on the architecture math (Claude TTFT ~300–800ms + sentence completion ~500ms + TTS first-call ~500–1000ms). 1.5s would be tighter and arguably better, but if OpenAI's TTS p95 is ~1.2s for a short sentence the line would be permanently red. Tentative lean: 2.0s.

2. **Sentence boundary regex strictness.** Naïve `[.!?]+\s+` works for ~95% of cases but mis-fires on `Dr. Smith` or `e.g.` or numeric lists like `1. Bench`. A small abbreviation allowlist (Dr., Mr., Mrs., e.g., i.e., vs.) handles the worst offenders without a real tokenizer dep. Tentative lean: ship with the allowlist; observe in production; consider a real tokenizer like `pysbd` only if mis-splits become a noticeable problem.

3. **Audio transport: base64 over SSE, or binary out-of-band.** Base64 inflates by 33% but rides on the existing SSE connection — zero new infrastructure. Out-of-band binary (e.g., emitting `{"type":"audio_chunk_ready","url":"/audio/<turn>/<index>.mp3"}` and having the client fetch the URL) avoids the inflation but adds a round trip per chunk, undoing some of the latency win we're chasing. Tentative lean: base64 — payload size at our scale is trivial.

4. **What if a single sentence is very long?** A 200-character sentence is a slow TTS call that delays first audio. We could split on commas as a secondary delimiter past a certain length. Tentative lean: don't optimize early. Claude rarely produces single sentences that long when prompted to be a hyped fitness coach.

5. **Should `audio_chunk` events include the `text` field?** Pure-audio chunks save ~10–20% on each event payload. The text is useful for in-client debug overlays and possibly for future caption/transcript display. Tentative lean: include it — payload savings are small and the debugging value is real.

6. **Mobile cache cleanup.** With many sentences per turn, the `cache/voice-<session>-<index>.mp3` files accumulate. Three options: delete each file in `setOnPlaybackStatusUpdate`'s `didJustFinish` branch, sweep on session change, or rely on the OS-level cache eviction. Tentative lean: delete on `didJustFinish` for simplicity.

7. **TTS failure for a single sentence.** What if OpenAI 429s on sentence 3 of an 8-sentence reply? Agent-side options: emit a placeholder `audio_chunk` with empty mp3 (silent gap), skip that index entirely (jump from 2 → 4), or fail the whole turn. Tentative lean: skip — the user hears uninterrupted speech across the gap, the log records the failure, and the chat UI's text-rendering is unaffected.

8. **Voice mode requires JWT auth, but the telemetry endpoint also.** Should `/telemetry/voice` reject when called by a user whose previous `/chat` didn't have `voice_mode: true`? Probably overkill — the histogram is per-user-id labeled so a misbehaving client only poisons their own bucket. Tentative lean: simple JWT check, no cross-request state.

9. **Cost monitoring.** Sentence chunking increases per-turn TTS call count from 1 to N. The daily-char cap (`TTS_DAILY_CHAR_CAP_PER_USER`) still bounds total chars, but per-call overhead and minimum billing units (gpt-4o-mini-tts charges per token) may matter at scale. Probably negligible at our user count; revisit if the OpenAI bill noticeably moves.

10. **Backpressure if TTS lags Claude.** Could Claude finish faster than TTS can generate? Yes, easily for short replies — Claude's tokens arrive in ~5ms increments, OpenAI TTS takes hundreds of ms per call. The `audio_tasks` list grows during streaming and drains afterwards. For a 6-sentence reply, that means most of the audio is generated *after* Claude finishes. This is fine — user hears speech start within ~1–2s, total turn duration ends up matching the speech duration, not the Claude duration. Worth understanding so we don't optimize the wrong thing.
