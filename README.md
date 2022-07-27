# live-wallet-payment-spec

Extends BIP70 to enable live wallets to act as payment protocol gateways - enabling subscriptions, and eliminating the need for mobile apps to perform multiple handshakes over flakey internet connections, and supporting specialty features, such as CoinJoin.

```txt
POST https://live-wallet.example.com/me/
Content-Type: application/jose+json
Authorization: Bearer <<auth-string>>

{
  "protected": base64UrlEncode({
    "alg": "none"
  }),
  "payload": base64UrlEncode({
    "iat": 1658903036, // when the request was generated (in seconds since epoch)
    "exp": 1658903963, // when the request expires (set to 30 seconds more than 'iat' by default)
    "jti": "HheL9rZ5t5NkhENOtks7mw", // a random nonce
    "payment_protocol": "dash:r=https://example.com/i/12345678"
  }),
  "signature": null
}
```

## Problems

- DashDirect relies on unreliable, ever-changing mobile wallets
- Mobile wallets don't support current features (CoinJoin, BIP70, etc)

## Solution

Use a reliable, always-on, online live wallet that requires only a **single network request** in order to complete a payment.

**In short**:

If the user has entered a live wallet URL in the settings, POST the payment protocol URL to that URL rather than opening it on the users phone.

### 1. Setup

There needs to be two inputs in DashDirect settings:
- Live Wallet URL
- Live Wallet Auth (random base64 or base62 string)

It's fine to auto-generate the auth string, as long as the user has a way to unhide it and see it in full.

### 2. Payment Request

If a Live Wallet URL exists, the app should POST to it rather than opening an on-phone wallet:

```txt
POST https://live-wallet.example.com/me/
Content-Type: application/jose+json
Authorization: Bearer <<auth-string>>

{
  "protected": base64UrlEncode({
    "alg": "none"
  }),
  "payload": base64UrlEncode({
    "iat": <<issued at: seconds since epoch>>, // the time at which the request was generated
    "exp": <<expires at: seconds since epoch>>, // seconds until the request expires (set to 30 seconds by default)
    "jti": "<<nonce: random string>>",
    "payment_protocol": "dash:r=https://example.com/i/12345678"
  }),
  "signature": null
}
```

If there's a `200 OK` response, DashDirect should continue "waiting for payment". Otherwise it should error "Live Wallet failed" and try the on-phone wallet app as it already does.
