# skink-tts

Text-to-speech for Skink with a pluggable provider interface: an
OpenAI-compatible REST provider (works with LocalAI, sherpa-onnx,
OpenRouter, …) and subprocess providers for offline engines
(espeak/espeak-ng, piper, macOS `say`).

## Install

```
scales add markoxley/skink-tts
```

## Quick start

```skink
import "mod/markoxley/skink-tts"
import "mod/markoxley/skink-tts/tts_iface"
import "std/libc"

// pick whatever's available: espeak-ng/espeak/say on PATH,
// else OpenAI when TTS_API_KEY or OPENAI_API_KEY is set
p := tts.Detect()
if p.Name == "none" {
    println("no TTS engine available")
    return 1
}
bytes, n, err := p.Speak("hello world", tts_iface.DefaultOpts())
// bytes = WAV audio (AIFF for `say`), n = byte count; caller frees
libc.free(bytes)
```

## Providers

| Constructor | Backend | Notes |
|---|---|---|
| `tts_rest.OpenAI(key, model)` | `api.openai.com` | voices: alloy, echo, fable, onyx, nova, shimmer |
| `tts_rest.WithBase(base, key, model)` | any OpenAI-compatible server | e.g. `http://localhost:8080` for LocalAI |
| `tts_proc.Espeak()` | `espeak-ng` or `espeak` | `Opts.Speed` scales wpm around 160; `Voice` = `-v` |
| `tts_proc.Piper(model)` | `piper` | `model` = path to an `.onnx` voice |
| `tts_proc.Say()` | macOS `say` | outputs AIFF, not WAV |
| `tts.Detect()` | first available | offline first, then `TTS_API_KEY`/`OPENAI_API_KEY` |

## Options

```skink
import "mod/markoxley/skink-tts/tts_iface"

o := tts_iface.DefaultOpts()
o.Voice = "alloy"    // voice name (REST) or -v value (espeak)
o.Speed = 1.25       // rate multiplier
o.Format = "wav"     // "mp3" also accepted by OpenAI-compatible APIs
```

## Custom providers

`tts_iface.Provider` is a vtable — implement `speakFn` and set
`State` to your own backend:

```skink
import "mod/markoxley/skink-tts/tts_iface"
import "std/errors"

fn mySpeak(p: *tts_iface.Provider, text: string, o: tts_iface.Opts) -> (string, int, errors.Error) {
    // return malloc'd audio bytes + length
}

p := tts_iface.New("mine")
p.speakFn = mySpeak
```

## Tests

```
bash run_tests.sh
```

- `tts_test` — vtable dispatch, uninitialized-provider error, engine
  detection, and a scripted fake HTTP server validating the REST
  provider's POST end-to-end over real TCP.
- `live_test` — runs every installed offline engine (espeak-ng found
  on PATH, piper via `TTS_PIPER_MODEL`, `say` on macOS) and the real
  OpenAI API when `TTS_API_KEY`/`OPENAI_API_KEY` (+ optional
  `TTS_BASE_URL`) is set. Self-skips when nothing is available.
