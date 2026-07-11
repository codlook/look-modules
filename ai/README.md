# ai — LOOK resmi Claude (Anthropic) modülü

Claude API'ye LOOK'tan bağlanır. **Canlı token akışı** (`ai_stream`) çekirdeğin
`http::stream` primitifi üzerine saf LOOK ile yazılmıştır — sıfır C++ bağımlılığı.

## Kurulum

```bash
lk module install github.com/codlook/look-modules/ai
```

## Kullanım

```lk
use "pkg/ai/ai.lk";

// ── Tam yanıt (bloklamalı) ──────────────────────────────────
$r = ai_chat(
    [["role" => "user", "content" => "LOOK nedir? tek cümle"]],
    ["model" => "claude-sonnet-5", "max_tokens" => 300]
);
print($r["text"]);

// ── Canlı akış (token token) ────────────────────────────────
ai_stream(
    [["role" => "user", "content" => "Bana bir LOOK HTTP endpoint yaz"]],
    function($token) {
        print($token);   // her token geldikçe — WebSocket/SSE ile istemciye it
    },
    ["model" => "claude-sonnet-5", "max_tokens" => 1024]
);
```

`api_key` verilmezse `ANTHROPIC_API_KEY` ortam değişkeninden okunur.

## Fonksiyonlar

| Fonksiyon | Açıklama |
|-----------|----------|
| `ai_chat($messages, $opts)` | Tam yanıt döner: `["text"=>..., "raw"=>...]` veya `["error"=>...]` |
| `ai_stream($messages, $on_token, $opts)` | `$on_token($metin)` her token deltasında çağrılır. Döner: `["ok"=>true]` / `["error"=>...]` |

### `$messages`
```lk
[["role" => "user", "content" => "..."],
 ["role" => "assistant", "content" => "..."],
 ["role" => "user", "content" => "..."]]
```

### `$opts`
| Anahtar | Varsayılan | Açıklama |
|---------|-----------|----------|
| `model` | `claude-sonnet-5` | Model ID |
| `max_tokens` | `4096` | Yanıt token sınırı |
| `system` | — | Sistem prompt'u (string) |
| `cache` | — | `true` ise system prompt'u prompt-cache'e alır (maliyet düşer) |
| `api_key` | `env("ANTHROPIC_API_KEY")` | API anahtarı |
| `endpoint` | Anthropic | Test/proxy için override |

## Prompt caching (maliyet)

Büyük, sabit bir sistem prompt'unu (ör. LOOK dokümanı) tekrar tekrar tam fiyattan
işlememek için `cache => true` kullan:

```lk
ai_stream($messages, $cb, [
    "system" => $buyuk_dokuman,
    "cache"  => true            // ilk istekte yazılır, sonrakiler ~0.1x
]);
```

## WebSocket ile canlı akış örneği

```lk
route("WS", "/looky", function() {
    ws::on("message", function($msg) {
        ai_stream([["role"=>"user","content"=>$msg]], function($token) {
            ws::send($token);    // token'ı anında istemciye ilet
        }, ["model" => "claude-sonnet-5"]);
    });
});
```

## Notlar

- `ai_stream`, LOOK çekirdeğinin **`http::stream`** primitifini (v0.x+) kullanır.
  Eski sürümlerde bu primitif yoksa yalnızca `ai_chat` çalışır.
- SSE ayrıştırma modülün içinde yapılır; Anthropic'in `content_block_delta`
  olaylarından `delta.text` çıkarılır.
