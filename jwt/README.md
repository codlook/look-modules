# jwt — LOOK resmi JWT modülü

HS256 (HMAC-SHA256) ile JWT üretme ve doğrulama.

## Kurulum

```bash
look module install jwt
```

## Kullanım

```lk
use "pkg/jwt/jwt.lk";

// Token üret
$token = jwt_sign(
    ["user_id" => 42, "rol" => "admin"],
    "gizli-anahtar",
    ["exp" => 3600]   // 1 saat geçerli
);

// Doğrula (null döner: geçersiz imza veya süresi dolmuş)
$payload = jwt_verify($token, "gizli-anahtar");
if ($payload == null) {
    response::status(401);
    print(json::encode(["error" => "Yetkisiz"]));
    return;
}

print("Kullanıcı: " . $payload["user_id"]);

// İmzasız decode (sadece okuma)
$raw = jwt_decode($token);
```

## Fonksiyonlar

| Fonksiyon | Açıklama |
|-----------|----------|
| `jwt_sign($payload, $secret, $opts)` | Token üretir. `$opts`: `["alg" => "HS256", "exp" => 3600]` |
| `jwt_verify($token, $secret)` | Doğrular, payload döner. Geçersizse `null`. |
| `jwt_decode($token)` | İmza doğrulamadan payload decode eder. |

## Auth middleware örneği

```lk
use "pkg/jwt/jwt.lk";

$JWT_SECRET = "uygulamana-ozel-gizli";

function require_auth() {
    $header = request::header("Authorization");
    if ($header == null) {
        response::status(401);
        print(json::encode(["error" => "Token gerekli"]));
        return null;
    }
    $token = string::replace($header, "Bearer ", "");
    $payload = jwt_verify($token, $JWT_SECRET);
    if ($payload == null) {
        response::status(401);
        print(json::encode(["error" => "Geçersiz veya süresi dolmuş token"]));
        return null;
    }
    return $payload;
}

route("GET", "/profil", function() use ($JWT_SECRET) {
    $user = require_auth();
    if ($user == null) { return; }
    print(json::encode(["user_id" => $user["user_id"]]));
});
```
