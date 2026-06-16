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
(load "git@github.com:carpentry-org/llm@0.3.0")
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

### Streaming with tool calls

Use `poll-event` instead of `poll` to receive both text and tool calls from
a streaming response. Incremental tool call data (OpenAI/Anthropic) is
accumulated internally and emitted as complete `ToolCall` values:

```clojure
(let [tools [(ToolDef.init @"get_weather" @"Get weather" schema)]
      req (LLM.chat-request-with-tools "gpt-4"
            [(Message.user "Weather in Paris?")] 256 0.7 tools)]
  (match (LLM.chat-stream &config &req)
    (Result.Success stream)
      (do
        (while-do true
          (match (LlmStream.poll-event &stream)
            (Maybe.Nothing) (break)
            (Maybe.Just evt)
              (match-ref &evt
                (StreamEvent.Text tok) (IO.print tok)
                (StreamEvent.ToolCallEvent tc)
                  (println* "tool: " (ToolCall.name tc)
                            " args: " (ToolCall.arguments tc)))))
        (LlmStream.close stream))
    (Result.Error e) (IO.errorln &(LLMError.str &e))))
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

### Agentic tool-use loop

`LLM.chat-loop` handles the full call-handle-respond cycle automatically:

```clojure
(let [config (LLM.ollama "http://localhost:11434")
      tools [(ToolDef.init @"get_weather" @"Get weather" schema)]
      msgs [(Message.user "Weather in Paris?")]
      handler (fn [tc]
                (if (= (ToolCall.name tc) "get_weather")
                  @"22C, sunny"
                  @"unknown tool"))]
  (match (LLM.chat-loop &config "llama3" msgs 256 0.7 &tools handler 10)
    (Result.Success r) (println* (LLMResponse.content &r))
    (Result.Error e) (IO.errorln &(LLMError.str &e))))
```

The handler receives each `ToolCall` by reference and returns the result as a
`String`. The loop repeats until the model stops calling tools or the iteration
limit is reached.

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

### Embeddings

```clojure
(let [config (LLM.openai "sk-...")
      req (LLM.embedding-request "text-embedding-3-small" [@"hello" @"world"])]
  (match (LLM.embed &config &req)
    (Result.Success r)
      (println* "got " (Array.length (EmbeddingResponse.embeddings &r)) " embeddings")
    (Result.Error e) (IO.errorln &(LLMError.str &e))))
```

Anthropic does not offer an embeddings API — calling `LLM.embed` with an
Anthropic config returns a `Transport` error.

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
| `LLM.embedding-request model input` | Embedding request (input is an array of strings) |
| `Message.user content` | User message |
| `Message.assistant content` | Assistant message |
| `Message.system content` | System message |
| `Message.tool-result call-id content` | Tool result message (for follow-ups) |
| `Message.from-response &r` | Convert an `LLMResponse` to an assistant message for history |

### Sending

| Function | Purpose |
|----------|---------|
| `LLM.chat config req` | Synchronous chat. Returns `(Result LLMResponse LLMError)` |
| `LLM.chat-loop config model msgs max-tokens temp tools handler max-iters` | Agentic tool-use loop. Calls `chat`, invokes `handler` per tool call, repeats until done or limit reached |
| `LLM.chat-stream config req` | Streaming chat. Returns `(Result LlmStream LLMError)` |
| `LlmStream.poll stream` | Returns `(Maybe String)` — next text token, or `Nothing` when done |
| `LlmStream.poll-event stream` | Returns `(Maybe StreamEvent)` — text or tool call event, or `Nothing` when done |
| `LLM.embed config req` | Generate embeddings. Returns `(Result EmbeddingResponse LLMError)` |
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
  in a separate `system` field. No native JSON mode. No embeddings API.
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

The unit tests don't make network calls. For
live tests against real APIs, see `examples/anthropic.carp`,
`examples/openai.carp`, `examples/ollama.carp`, `examples/gemini.carp` and
their `_stream`, `_tool`, `_json` variants, plus `examples/embeddings.carp`
for the embeddings API. Most require an API key in the appropriate environment
variable.

<hr/>

Have fun!
