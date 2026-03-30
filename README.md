# Node.js voice agent with AssemblyAI Universal-3 Pro Streaming

Build a real-time voice agent in **Node.js** using the **AssemblyAI Universal-3 Pro Streaming model** (`u3-rt-pro`) for speech-to-text — no Python required, no heavy framework dependencies.

Two modes in one repo:

1. **Terminal agent** (`src/agent.js`) — mic input via `mic`, plays TTS audio in your terminal
2. **Browser server** (`src/server.js`) — Node.js WebSocket server with a browser UI using `getUserMedia`

## Why AssemblyAI Universal-3 Pro for Node.js?

| Metric | AssemblyAI Universal-3 Pro | Deepgram Nova-3 |
|--------|---------------------------|-----------------|
| P50 latency | **307 ms** | 516 ms |
| Word Error Rate | **8.14%** | 9.87% |
| Neural turn detection | ✅ | ❌ (VAD only) |
| Mid-session prompting | ✅ | ❌ |
| Real-time diarization | ✅ | ❌ |
| Anti-hallucination | ✅ | ❌ |
| Node.js WebSocket (`ws`) | ✅ | ✅ |

AssemblyAI's neural turn detection eliminates the need for a separate VAD library. The model uses both acoustic and linguistic signals to detect when a speaker has finished — not just when they've gone silent.

## Architecture

```
Terminal mode:
  Mic (mic npm) → PCM s16le → AssemblyAI WS → Turn(end_of_turn=true) → GPT-4o → ElevenLabs → afplay

Browser mode:
  getUserMedia → ScriptProcessor → PCM s16le → ws://server/stream
                                                     │
                                           AssemblyAI Universal-3 Pro
                                                     │ Turn event
                                               OpenAI GPT-4o
                                                     │ text
                                             ElevenLabs TTS
                                                     │ base64 MP3
                                     ◄── Browser AudioContext playback
```

## Quick start

```bash
git clone https://github.com/AssemblyAI/voice-agent-nodejs-assemblyai
cd voice-agent-nodejs-assemblyai

npm install
cp .env.example .env
# Edit .env with your API keys
```

### Terminal agent

```bash
npm start
# Speak into your mic — Ctrl+C to quit
```

### Browser agent

```bash
npm run server
# Open http://localhost:3000
```

## AssemblyAI WebSocket URL

```js
const AAI_WS_URL =
  `wss://streaming.assemblyai.com/v3/ws` +
  `?speech_model=u3-rt-pro` +
  `&encoding=pcm_s16le` +              // raw 16-bit signed little-endian PCM
  `&sample_rate=16000` +
  `&end_of_turn_confidence_threshold=0.4` +
  `&min_end_of_turn_silence_when_confident=300` +
  `&max_turn_silence=1500` +
  `&token=${ASSEMBLYAI_API_KEY}`;
```

## Turn detection

AssemblyAI v3 uses three event types. Handle them like this:

```js
ws.on("message", async (data) => {
  const msg = JSON.parse(data.toString());

  if (msg.type === "Begin") {
    // Session started — msg.id is the session ID
    console.log(`Session: ${msg.id}`);
  }

  if (msg.type === "Turn" && !msg.end_of_turn) {
    // Partial transcript — update your UI in real time
    process.stdout.write(`\r${msg.transcript}`);
  }

  if (msg.type === "Turn" && msg.end_of_turn) {
    // End-of-turn detected — respond now
    const reply = await generateResponse(msg.transcript);
    await speak(reply);
  }
});
```

## Sending audio

**Browser** (`getUserMedia` + `ScriptProcessor`):

```js
processor.onaudioprocess = (e) => {
  const float32 = e.inputBuffer.getChannelData(0);
  const int16 = new Int16Array(float32.length);
  for (let i = 0; i < float32.length; i++) {
    int16[i] = Math.max(-32768, Math.min(32767, Math.round(float32[i] * 32767)));
  }
  ws.send(int16.buffer);
};
```

**Terminal** (`mic` package):

```js
const micStream = micInstance.getAudioStream();
micStream.on("data", (chunk) => {
  aaiWs.send(chunk); // raw PCM s16le bytes — no conversion needed
});
```

## Tuning turn detection

| Parameter | Default | Lower → | Higher → |
|-----------|---------|---------|---------|
| `end_of_turn_confidence_threshold` | 0.4 | Faster response | Fewer false triggers |
| `min_end_of_turn_silence_when_confident` | 300ms | Snappier | More natural pauses |
| `max_turn_silence` | 1500ms | Faster cutoff | More thinking time |

## Keyterm prompting (mid-session)

Inject domain-specific vocabulary after the session starts without restarting:

```js
ws.send(JSON.stringify({
  type: "UpdateConfiguration",
  keyterms: ["AssemblyAI", "Universal-3", "your-product-name"],
}));
```

## Requirements

- Node.js 18+
- `npm install` installs: `ws`, `openai`, `elevenlabs`, `mic`, `dotenv`
- macOS: `afplay` (built-in) for audio playback
- Linux: `aplay` or `mpg123` for audio playback

## Deploy to Railway, Render, or Fly.io

```bash
# Set environment variables in the platform dashboard, then:
npm run server
```

The browser server is stateless per-connection — each WebSocket session has its own AssemblyAI connection and conversation history.

## Related tutorials

- [Tutorial 05: raw WebSocket voice agent (Python)](../05-websocket-universal-3-pro) — the same pattern in Python, useful for comparing the two implementations
- [Tutorial 04: Twilio + Universal-3 Pro Streaming](../04-twilio-universal-3-pro) — add phone call support to a Node.js-style architecture
- [Tutorial 01: LiveKit + Universal-3 Pro Streaming](../01-livekit-universal-3-pro) — production-grade Python framework if you need managed WebRTC infrastructure

## Resources

- [AssemblyAI Universal Streaming docs](https://www.assemblyai.com/docs/speech-to-text/universal-streaming)
- [AssemblyAI Streaming API reference](https://www.assemblyai.com/docs/api-reference/streaming)
- [AssemblyAI Node.js SDK](https://github.com/AssemblyAI/assemblyai-node-sdk)
- [ws — Node.js WebSocket library](https://github.com/websockets/ws)

---

<div class="blog-cta_component">
  <div class="blog-cta_title">Build a Node.js voice agent today</div>
  <div class="blog-cta_rt w-richtext">
    <p>Sign up for a free AssemblyAI account and connect to Universal-3 Pro Streaming from your Node.js app in under 15 minutes.</p>
  </div>
  <a href="https://www.assemblyai.com/dashboard/signup" class="button w-button">Start building</a>
</div>

<div class="blog-cta_component">
  <div class="blog-cta_title">Experiment with real-time turn detection</div>
  <div class="blog-cta_rt w-richtext">
    <p>Try streaming transcription in our Playground and observe how punctuation and silence handling shape turn boundaries in real time. Compare behaviors across Universal-3 Pro Streaming and Universal-streaming models.</p>
  </div>
  <a href="https://www.assemblyai.com/playground" class="button w-button">Open playground</a>
</div>
