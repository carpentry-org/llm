## llm

A multi-provider LLM client for Carp. Supports
[Anthropic](https://docs.anthropic.com/),
[OpenAI](https://platform.openai.com/docs/),
[Ollama](https://ollama.com/), and
[Google Gemini](https://ai.google.dev/) behind a single common API.

Built on [http-client](https://github.com/carpentry-org/http-client) and
[json](https://github.com/carpentry-org/json).

## Installation

```clojure
(load "git@github.com:carpentry-org/llm@0.1.0")
```

Requires OpenSSL for HTTPS providers (Anthropic, OpenAI, Gemini). Ollama over
plain HTTP works without OpenSSL.

## Usage

### Basic chat (Ollama, no API key)

```clojure
(let [config (LLM.ollama "http://localhost:11434")
      req (LLM.chat-request "llama3" [(Message.user "hello")] 256 0.7)]
  (match (LLM.chat &config &req)
    (Result.Success r) (println* (LLMResponse.content &r))
    (Result.Error e) (IO.errorln &(LLMError.str &e))))
```

### Switching providers

The same `LLM.chat` works against any provider — just change the config
constructor:

```clojure
(LLM.anthropic "sk-ant-...")     ; Anthropic
(LLM.openai    "sk-...")          ; OpenAI
(LLM.ollama    "http://...")      ; Ollama
(LLM.gemini    "AIza...")         ; Gemini
```

### Streaming

```clojure
(match (LLM.chat-stream &config &req)
  (Result.Success stream)
    (do
      (while-do true
        (match (LlmStream.poll &stream)
          (Maybe.Nothing) (break)
          (Maybe.Just tok) (IO.print &tok)))
      (LlmStream.close stream))
  (Result.Error e) (IO.errorln &(LLMError.str &e)))
```

### Tool use

```clojure
(let [schema (JSON.obj [(JSON.entry @"type" (JSON.Str @"object"))
                        (JSON.entry @"properties"
                          (JSON.obj [(JSON.entry @"city"
                            (JSON.obj [(JSON.entry @"type" (JSON.Str @"string"))]))]))])
      tools [(ToolDef.init @"get_weather" @"Get current weather" schema)]
      req (LLM.chat-request-with-tools "gpt-4"
            [(Message.user "Weather in Paris?")] 256 0.7 tools)]
  (match (LLM.chat &config &req)
    (Result.Success r)
      (when (> (Array.length (LLMResponse.tool-calls &r)) 0)
        (let [tc (Array.unsafe-nth (LLMResponse.tool-calls &r) 0)]
          (println* "calling " (ToolCall.name tc) " with " (ToolCall.arguments tc))))
    (Result.Error e) (IO.errorln &(LLMError.str &e))))
```

To send a tool result back, build a follow-up request including the assistant's
tool call message and a tool result:

```clojure
(let [msgs [(Message.user "Weather in Paris?")
            (Message.from-response &r)
            (Message.tool-result &tool-call-id "22°C, sunny")]
      req2 (LLM.chat-request-with-tools "gpt-4" msgs 256 0.7 @&tools)]
  (LLM.chat &config &req2))
```

### JSON output

```clojure
; Plain JSON mode (any valid JSON)
(LLM.chat-request-json model msgs max-tokens temp)

; Schema-constrained JSON
(LLM.chat-request-with-schema model msgs max-tokens temp schema)
```

Anthropic has no native JSON mode — for them, the library falls back to a
system prompt instruction. Best-effort, not guaranteed. All other providers use
their native JSON mode.

## API

### Construction

| Function | Purpose |
|----------|---------|
| `LLM.anthropic key` | `ProviderConfig` for Anthropic |
| `LLM.openai key` | `ProviderConfig` for OpenAI |
| `LLM.ollama base-url` | `ProviderConfig` for Ollama |
| `LLM.gemini key` | `ProviderConfig` for Gemini |
| `LLM.chat-request model msgs max-tokens temp` | Plain text request |
| `LLM.chat-request-with-tools model msgs max-tokens temp tools` | Request with tool definitions |
| `LLM.chat-request-json model msgs max-tokens temp` | JSON output mode |
| `LLM.chat-request-with-schema model msgs max-tokens temp schema` | Schema-constrained JSON |
| `Message.user content` | User message |
| `Message.assistant content` | Assistant message |
| `Message.system content` | System message |
| `Message.tool-result call-id content` | Tool result message (for follow-ups) |
| `Message.from-response &r` | Convert an `LLMResponse` to an assistant message for history |

### Sending

| Function | Purpose |
|----------|---------|
| `LLM.chat config req` | Synchronous chat. Returns `(Result LLMResponse LLMError)` |
| `LLM.chat-stream config req` | Streaming chat. Returns `(Result LlmStream LLMError)` |
| `LlmStream.poll stream` | Returns `(Maybe String)` — next token, or `Nothing` when done |
| `LlmStream.close stream` | Close the underlying connection |

### Errors

```clojure
(deftype LLMError
  (Transport [String])          ; connection / DNS / network errors
  (Api [Int String String]))    ; HTTP status, error type, message
```

`LLMError.str &e` formats either variant for display. API errors are parsed
from each provider's specific error JSON shape.

## Provider quirks

- **Anthropic**: System messages are extracted from the message array and sent
  in a separate `system` field. No native JSON mode.
- **OpenAI**: Standard format. System messages are kept as a `"system"` role in
  the messages array.
- **Ollama**: Same shape as OpenAI for messages and tool calls. NDJSON for
  streaming (one JSON object per line) instead of SSE.
- **Gemini**: Different shape entirely (`contents`/`parts` instead of
  `messages`). Uses the `/v1beta` endpoint for tool support. Streaming uses
  `:streamGenerateContent?alt=sse`.

The library hides all of this — you just call `LLM.chat` and the provider's
build/parse functions translate.

## Testing

```
carp -x test/llm.carp
```

The unit tests don't make network calls (44 tests, JSON build/parse logic). For
live tests against real APIs, see `examples/anthropic.carp`,
`examples/openai.carp`, `examples/ollama.carp`, `examples/gemini.carp` and
their `_stream`, `_tool`, `_json` variants. Most require an API key in the
appropriate environment variable.

<hr/>

Have fun!
