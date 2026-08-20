# oauth — OAuth 2.0 + PKCE for LOOK

Provider-agnostic OAuth 2.0 **Authorization Code flow with PKCE (S256)**, built purely
on the core `http::` and `crypto::` builtins. You build the authorize URL, redirect the
user, and exchange the returned code for tokens. CSRF `state` and PKCE helpers are
included, plus presets for Google and GitHub.

## Install

```bash
lk module install github.com/codlook/look-modules/oauth
```

```lk
use oauth
```

## Flow

```lk
use oauth

$cfg = oauth_google($client_id, $client_secret, "https://app.example.com/auth/callback")

# 1) Start — store state + verifier in the session, then redirect
route("GET", "/auth/google", function() {
    $state = oauth_state()
    $verifier = oauth_pkce_verifier()
    session::set("oauth_state", $state)
    session::set("oauth_verifier", $verifier)
    return response::redirect(oauth_authorize_url($cfg, $state, oauth_pkce_challenge($verifier)))
})

# 2) Callback — check state, then exchange the code for tokens
route("GET", "/auth/callback", function() {
    if (request::get("state") != session::get("oauth_state")) {
        return response::error(400, "bad state")
    }
    $token = oauth_exchange($cfg, request::get("code"), session::get("oauth_verifier"))
    # $token["access_token"] — now call the provider's userinfo endpoint with http::get
})
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `oauth_state()` | `string` | Random CSRF state to store and re-check on callback. |
| `oauth_pkce_verifier()` | `string` | Random PKCE `code_verifier` (RFC 7636 length range). |
| `oauth_pkce_challenge($verifier)` | `string` | S256 `code_challenge` = base64url(SHA256(verifier)). |
| `oauth_authorize_url($cfg, $state, $challenge)` | `string` | The provider URL to redirect the user to. |
| `oauth_exchange($cfg, $code, $verifier)` | `assoc` | POST to the token endpoint; returns the parsed JSON token response. |
| `oauth_google($id, $secret, $redirect)` | `assoc` | Google provider config. |
| `oauth_github($id, $secret, $redirect)` | `assoc` | GitHub provider config. |

## Provider config

A `$cfg` is an assoc with `auth_url`, `token_url`, `client_id`, `client_secret`,
`redirect_uri`, `scope`. The presets fill these in for Google and GitHub; build your
own for any other OAuth 2.0 provider.

## Notes

- Always store `state` and `verifier` server-side (session) and verify `state` on the
  callback — this module gives you the values, you enforce the check.
- PKCE uses the S256 method; the challenge is verified against a known RFC 7636 vector.
