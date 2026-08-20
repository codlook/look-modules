# webhook — verify webhook signatures for LOOK

Verify inbound webhook signatures with **HMAC-SHA256 in constant time**, built on the
core `crypto::` builtins. A generic verify plus the GitHub and Stripe header formats.
Comparison always uses `crypto::constant_compare`, so an attacker can't discover a valid
signature byte-by-byte via timing.

## Install

```bash
lk module install github.com/codlook/look-modules/webhook
```

```lk
use webhook
```

## Use

```lk
use webhook

route("POST", "/webhooks/stripe", function() {
    $payload = request::body()                       # RAW body — verify before parsing
    $sig = request::header("Stripe-Signature")
    if (!webhook_verify_stripe($payload, $sig, env("STRIPE_WEBHOOK_SECRET"))) {
        return response::error(400, "bad signature")
    }
    $event = json::decode($payload)
    # ... handle the verified event
})
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `webhook_sign($payload, $secret)` | `string` | Hex HMAC-SHA256 of the payload (for generating/testing). |
| `webhook_verify($payload, $signature_hex, $secret)` | `bool` | Constant-time compare against a bare hex signature. |
| `webhook_verify_github($payload, $header, $secret)` | `bool` | GitHub's `X-Hub-Signature-256: sha256=<hex>`. |
| `webhook_verify_stripe($payload, $sig_header, $secret)` | `bool` | Stripe's `Stripe-Signature: t=..,v1=..` (signs `t.payload`). |

## Notes

- Verify the **raw** request body (`request::body()`) before parsing JSON. Re-serializing
  the body first (even a whitespace change) breaks the signature.
- Stripe's scheme signs `"{t}.{payload}"`; this module reconstructs that and compares
  against `v1`. Timestamp-tolerance (replay window) enforcement, if you want it, is a
  simple check on `t` you can add at the call site.
