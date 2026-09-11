# System Design

Diagrams only. For the written breakdown — directory map, naming conventions, and
what each component does — see [structure.md](structure.md).

## Reading the numbers

Edges that represent a **step in a flow** are numbered so a path can be followed
in order. Edges that represent **structure** (an import, "depends on") are never
numbered — that difference is the point of the notation.

| Situation | Notation | Read it as |
|---|---|---|
| Linear step | `1`, `2`, `3` | happens next |
| **Split — either/or** (one branch is taken) | same number, letter suffix: `2a`, `2b` | *or* |
| **Split — fan-out** (every branch is taken) | dotted decimal: `4.1`, `4.2` | *and* |
| **Converge** (branches rejoin) | the shared number repeats on both incoming edges | both paths arrive at the same step |
| Structural / dependency edge | no label | not part of any flow |

So when an arrow reaches a box and then splits in two, the question is whether the
flow picks one exit or takes both. Picks one → `2a` / `2b` (they are the *same*
step, two outcomes, so the counter does **not** advance twice). Takes both →
`4.1` / `4.2` (one step, two effects). When the branches meet again, the next
number is written once on each incoming arrow, not renumbered per branch:

```mermaid
flowchart LR
    a["step"] -->|1| b["branch point"]
    b -->|2a| c["taken when X"]
    b -->|2b| d["taken when not X"]
    c -->|3| e["rejoin"]
    d -->|3| e
    e -->|"4.1"| f["both happen"]
    e -->|"4.2"| g["both happen"]
```

Two diagram types opt out:

- **Sequence diagrams** use mermaid's `autonumber`, which counts messages
  linearly. Branching is already expressed by the `alt` / `else` blocks, so the
  letter suffixes are redundant there — an `alt` block visually brackets its own
  alternative.
- **State diagrams** and the **module dependency graph** have no step order at
  all: their edges are transitions and imports, not a sequence. They stay
  unnumbered.

## Layers

Two delivery mechanisms over one framework-free service core. Numbers trace one
request; `1a`/`1b` are the two entry points, `3a`–`3d` the backend a request
lands on (exactly one per request).

```mermaid
flowchart TD
    client["HTTP client"]
    user["Terminal user"]

    subgraph delivery["Delivery layers"]
        direction LR
        api["app/api/<br/>FastAPI · Starlette<br/><i>HTTP-only</i>"]
        cli["app/cli/<br/>Typer · Rich<br/><i>terminal-only</i>"]
    end

    subgraph core["Shared core"]
        services["app/services/<br/>pure business logic<br/><b>must not import</b><br/>FastAPI · Starlette · Typer · Rich"]
    end

    subgraph shared["Shared primitives"]
        direction LR
        schemas["app/schemas/<br/>pydantic contracts"]
        corepkg["app/core/<br/>exceptions · logging"]
        config["app/config.py<br/>Settings singleton"]
    end

    subgraph backends["MLX backends"]
        direction LR
        mlxlm["mlx-lm<br/>text"]
        mlxvlm["mlx-vlm<br/>multimodal"]
        whisper["mlx-whisper<br/>STT · Whisper"]
        audio["mlx-audio<br/>STT · Parakeet<br/>TTS · Kokoro"]
    end

    disk[("models/<br/>weights · registry.json · runtime markers")]

    client -->|1a| api
    user -->|1b| cli
    api -->|2| services
    cli -->|2| services
    services -->|3a| mlxlm
    services -->|3b| mlxvlm
    services -->|3c| whisper
    services -->|3d| audio
    services -->|4| disk
    delivery -.-> shared
    core -.-> shared
```

## Module dependency graph

Who imports whom inside `app/`. Every module also imports from
`app/config.py` + `app/core/` (and most from `app/schemas/`); those edges are
aggregated as the dotted lines at the bottom to keep the graph readable.

```mermaid
flowchart TB
    appmain["app/main.py<br/>app factory + service singletons"]

    subgraph api["app/api/ — HTTP delivery"]
        direction TB
        ro["routes_openai.py"]
        rm["routes_models.py"]
        ra["routes_audio.py"]
        rh["routes_health.py"]
        mw["middleware.py"]
        resp["response.py"]
        conc["concurrency.py<br/>chat gate + chat thread"]
    end

    subgraph cli["app/cli/ — terminal delivery"]
        direction TB
        cmain["main.py<br/>Typer commands"]
        sel["select.py"]
        cs["chat_session.py<br/>ChatSession"]
        mcs["media_chat_session.py<br/>MediaChatSession"]
    end

    subgraph services["app/services/ — shared core"]
        direction TB
        inf["inference.py<br/>InferenceService"]
        minf["media_inference.py<br/>MediaInferenceService"]
        base["base.py<br/>LoadedModelService (ABC)"]
        aud["audio.py<br/>AudioService"]
        mm["model_manager.py<br/>ModelManager"]
        mrs["model_runtime_state.py<br/>ModelRuntimeState"]
    end

    subgraph patches["app/patches/"]
        direction TB
        pk["mlx_audio_kokoro.py"]
        pv["mlx_vlm_gemma4.py"]
    end

    subgraph shared["app/config.py · app/core/ · app/schemas/"]
        direction LR
        cfg["config.py<br/>Settings"]
        exc["core/exceptions.py"]
        log["core/logging.py"]
        sch["schemas/"]
    end

    appmain --> ro & rm & ra & rh
    appmain --> mw
    appmain --> conc
    appmain --> inf & minf & aud

    ro --> resp
    ro --> conc
    rm --> conc
    rm --> resp
    ra --> resp
    rh --> resp
    ro --> mm
    rm --> mm
    ro -. "_strip_audio_data_uri" .-> minf

    ro -. "deferred import<br/>of the singletons" .-> appmain
    rm -. " " .-> appmain
    ra -. " " .-> appmain
    rh -. " " .-> appmain

    cmain --> sel & cs & mcs
    cmain --> mm
    cs --> inf
    mcs --> mrs
    mcs --> pv

    inf --> base
    minf --> base
    minf --> pv
    aud --> pk
    base --> mrs
    mm --> mrs

    api -.-> shared
    cli -.-> shared
    services -.-> shared
    patches -.-> log
```

`MediaChatSession` is deliberately **not** wired to `MediaInferenceService`: it
imports `mlx_vlm` directly via `importlib` and manages its own runtime marker.
Only the HTTP path uses `MediaInferenceService`.

## Request flow — `POST /v1/chat/completions`

One chat request at a time: the gate is taken after validation and held across
model load *and* the entire generation. All blocking MLX work runs on the single
chat worker thread.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant M as LoggingMiddleware
    participant R as routes_openai.py
    participant G as chat gate<br/>api/concurrency.py
    participant W as chat worker thread<br/>run_chat · aiter_chat
    participant MM as ModelManager
    participant T as InferenceService<br/>mlx-lm
    participant V as MediaInferenceService<br/>mlx-vlm
    participant H as Exception handlers<br/>app/main.py

    C->>M: POST /v1/chat/completions
    M->>M: set correlation_id + request_id<br/>log request (media summarized)
    M->>R: forward

    R->>R: parse OpenAIChatCompletionRequest
    R->>R: _reject_unsupported_chat_features() → 400
    R->>R: _reject_unsupported_media_inputs() → 400
    R->>R: _normalize_stop_sequences()
    Note over R: Validation runs BEFORE the gate,<br/>so a malformed request never queues

    R->>G: acquire_chat_gate(chat_queue_timeout_seconds)
    alt still held when the timeout expires
        G--)R: TimeoutError
        R-->>C: 503 server busy
    else acquired
        G-->>R: held across load AND generation
    end

    Note over R,V: Backend selection — the two are mutually exclusive
    alt _request_uses_vlm(messages) or _model_is_vlm(model)
        R->>W: run_chat(_ensure_media_model_loaded)
        W->>MM: ensure_model_files_ready(name)
        MM-->>W: ModelInfo · 404 not found · 400 bad path · 500 load failed
        W->>T: unload() the text backend
        W->>V: load()
        W-->>R: resident
    else text request
        R->>W: run_chat(_ensure_model_loaded)
        W->>MM: ensure_model_loadable(name)
        MM-->>W: ModelInfo · 404 not found · 400 bad path or unsupported · 500 load failed
        W->>V: unload() the media backend
        W->>T: load()
        W-->>R: resident
    end
    Note over T,V: Generation goes to whichever backend was selected —<br/>both expose the same LoadedModelService interface

    alt stream = false
        R->>W: run_chat(chat) or run_chat(_collect_chat_completion)
        W->>T: generate — drained per token when verbose or stop is set
        T-->>W: text + usage
        W-->>R: text + usage
        R-->>C: 200 OpenAIChatCompletionResponse<br/>(+ x_metrics when verbose)
        R->>G: release — the route's finally
    else stream = true
        R-->>C: first chunk — delta.role = assistant
        loop per token, pumped by aiter_chat()
            R->>W: next() on _stream_with_stop_sequences(chat_stream())
            W->>T: step
            T-->>W: delta
            W-->>R: delta, trimmed at the first stop match
            R-->>C: data: {chunk}
            opt request.is_disconnected()
                R->>W: close() the generator on the same thread
            end
        end
        opt stream_options.include_usage
            R-->>C: usage chunk — empty choices
        end
        R-->>C: data: [DONE]<br/>(x_metrics on the final chunk when verbose)
        R->>G: release — the SSE generator's finally,<br/>which also fires on disconnect
    end

    Note over R,H: Failure BEFORE the response starts →<br/>handlers in app/main.py log once + map to HTTP
    Note over R: Failure MID-SSE → the stream generator is the<br/>boundary: logs with exc_info, emits error frame + [DONE]
```

## Request flow — `/v1/audio/*`

Audio never blocks the event loop and never queues behind chat: every call is
handed to the shared Starlette threadpool, so transcriptions and syntheses run
in parallel with each other and with a chat generation. A request **never
downloads** — `task audio:setup` is the only fetch path.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant M as LoggingMiddleware
    participant R as routes_audio.py
    participant P as Starlette threadpool<br/>shared — audio runs in parallel
    participant A as AudioService
    participant S as STT slot<br/>_ResidentModel
    participant K as TTS slot<br/>_ResidentModel
    participant H as Exception handlers<br/>app/main.py

    C->>M: GET /v1/audio/models · POST /v1/audio/transcriptions · POST /v1/audio/speech
    M->>M: set correlation_id + request_id, log request
    M->>R: forward

    alt GET /v1/audio/models
        R->>A: describe_stt() + describe_tts()
        A-->>R: repos · readiness · accepts_language_hint · voices · lang codes · speed range
        Note over A: Filesystem only — loads nothing,<br/>so a frontend can poll it on page load
        R-->>C: 200 AudioCapabilitiesResponse

    else POST /v1/audio/transcriptions
        R->>R: spool the upload to a NamedTemporaryFile<br/>suffix from filename, default .webm
        R->>P: run_in_threadpool(transcribe)
        P->>A: transcribe(path, language, model)
        A->>A: _resolve_stt_model() — not in STT_MODELS → 400
        A->>A: _stt_backend_for(repo_id) → whisper or parakeet
        A->>S: acquire(repo_id, loader)
        Note over S: same repo resident → straight through, runs in parallel<br/>different repo → wait for in-flight to drain, then swap
        S->>S: _ensure_speech_model_available()<br/>offline cache probe → 503 when absent
        S-->>A: handle + snapshot path
        alt whisper
            A->>A: mlx_whisper.transcribe(path, snapshot, language)
        else parakeet
            A->>A: handle.generate(path, chunked)<br/>language hint dropped — takes none
        end
        A-->>P: text
        P-->>R: text
        R->>R: remove the temp file — finally, runs on every path
        R-->>C: 200 TranscriptionResponse

    else POST /v1/audio/speech
        R->>R: response_format other than wav → 400
        R->>P: run_in_threadpool(synthesize)
        P->>A: synthesize(input, voice, speed, lang_code)
        A->>A: empty input → 500 InferenceError
        A->>A: lang_code outside Kokoro's set → 400
        A->>K: acquire(TTS_MODEL, _ensure_tts_loaded)
        K->>K: _ensure_speech_model_available() → 503<br/>_configure_espeak() · patch_interpolate_ceil_drift()
        K-->>A: kokoro + snapshot path
        A->>A: voices/NAME.safetensors missing?<br/>others cached → 400 bad voice · none cached → 503 re-run setup
        A->>A: kokoro.generate() → float32 chunks + sample rate
        A-->>P: samples + sample_rate
        P-->>R: samples + sample_rate
        R->>R: soundfile.write(buffer, format=WAV)
        R-->>C: 200 audio/wav bytes
    end

    Note over S,K: On release each slot re-arms its idle timer — after<br/>stt/tts_idle_timeout_seconds with nothing in flight, the model is dropped
    Note over R,H: Domain exceptions are mapped to HTTP in the route —<br/>anything else reaches app/main.py, logged once as 500
```

## Audio slot residency — parallel vs swap

Why a shared threadpool is safe: `_ResidentModel` decides per request whether to
run straight through or wait. Same model → concurrent. Different model → the
swap blocks until the in-flight requests finish, so a handle is never pulled out
from under a running generate.

```mermaid
sequenceDiagram
    autonumber
    participant A as Client A — whisper
    participant B as Client B — whisper
    participant D as Client C — parakeet
    participant S as STT slot

    A->>S: acquire(whisper)
    S->>S: slot empty → load whisper<br/>in_flight = 1
    B->>S: acquire(whisper)
    S->>S: already resident → no load<br/>in_flight = 2
    Note over A,S: Both hold the same handle — the lock guards<br/>load/unload only, never generation
    par Client A generates
        A->>A: transcribe
    and Client B generates
        B->>B: transcribe
    end

    D->>S: acquire(parakeet)
    S->>S: different repo AND in_flight > 0<br/>→ Condition.wait()
    Note over D,S: The swap is what waits — the two whisper<br/>requests are never interrupted

    A-->>S: done → in_flight = 1, notify_all()
    B-->>S: done → in_flight = 0, notify_all()
    S->>S: wakes, in_flight == 0 → release whisper<br/>(clears mlx_whisper ModelHolder), load parakeet
    S-->>D: handle + snapshot path
```

## Model lifecycle

States are visible across processes via marker files in `models/runtime/`.

```mermaid
stateDiagram-v2
    [*] --> absent

    absent --> downloading: models download
    downloading --> ready: snapshot_download ok<br/>registry updated
    downloading --> absent: download failed

    ready --> running: load into a backend<br/>API load or CLI chat
    running --> ready: unload · process exit ·<br/>PID check clears a stale marker

    ready --> unsupported: doctor verdict —<br/>model_type unsupported by MLX
    ready --> incomplete: doctor verdict —<br/>required files missing

    ready --> absent: models delete
    note right of running
        update and delete are BLOCKED
        while downloading or running
    end note
```

## Backend mutual exclusion

Loading into one backend releases the other — limited unified memory.

```mermaid
stateDiagram-v2
    [*] --> none

    none --> text: load a text model
    none --> media: load a VLM

    text --> text: swap text model<br/>(unloads previous)
    media --> media: swap VLM<br/>(unloads previous)

    text --> media: media request<br/>(text backend released first)
    media --> text: text request<br/>(media backend released first)

    text --> none: unload
    media --> none: unload

    note right of none
        AudioService (STT/TTS) is independent:
        it loads alongside whichever chat
        backend is resident, in its own two
        slots (one STT, one TTS), each with
        its own idle-unload timer.
    end note
```
