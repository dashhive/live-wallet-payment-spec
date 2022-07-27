# live-wallet-payment-spec

Extends BIP70 to enable live wallets to act as payment protocol gateways - enabling subscriptions, and eliminating the need for mobile apps to perform multiple handshakes over flakey internet connections.

```txt
POST https://live-wallet.example.com/me/
Content-Type: application/jose+json
Authorization: Bearer <<shared-secret-instead-of-kid-and-sig>>

{
  "protected": base64UrlEncode({
    "alg": "ES256",
    "kid": "<<key-thumbprint>>",
  }),
  "payload": base64UrlEncode({
    "iat": <<seconds-since-epoch>>,
    "exp": <<seconds-since-epoch-plus-60-seconds>>,
    "jti": <<random-base62-nonce>>,
    "payment_protocol": "dash:r=https://example.com/i/12345678"
  }),
  "signature": ec256Sign(header + '.' + payload)
}
```

This uses JOSE and standard cryptographic algorithms because they are widely available across development environments, devices, etc.

## Problems

- Mobile phones have incredibly unreliable network connections
- Most applications are slow to adopt new technology - such as CoinJoin

## Solution

Provide an always-on wallet that requires only a **single network request** in order to complete a payment.

The wallet can accept a signed payment protocol request and respond to it according to the user's preferences.

### 0. Proof of Concept

A proof of concept could be accomplished using a shared secret via `Authorization: Bearer xxxx` for the handshake, and payment requests.

### 1. Payment Protocal Gateway Request

1. Generate and store a random ECDSA or RSA keypair saved to the user's device.
2. Prompt the user for their live wallet address, which MUST use https. \
   Ex: <https://live-wallet.example.com/me/>.
3. Show the user a securely generated random security code. Ex: 123-456.
3. Send a self-issued payment protocol gateway request with the **suggested name of the application** to be approved, and **the security code**.
   ```txt
   POST https://live-wallet.example.com/me/
   Content-Type: application/jose+json
   Authorization: Bearer <<shared-secret-instead-of-jwk-and-sig>>

   {
     "protected": base64UrlEncode({
       "alg": "ES256",
       "kid": "<<jwk-thumbprint>>",
       "jwk": {
         "crv": "P-256",
         "kty": "EC",
         "x": "7tWXBU_zorxgtjSckxO1rxazCsMCo9tDCfMYxzDfplg",
         "y": "kcLHXbGV9Bu2HONXGjxjbBpq0TsASlCgixnE8O-D8EM"
       }
     }),
     "payload": base64UrlEncode({
       "iat": <<seconds-since-epoch>>,
       "exp": <<seconds-since-epoch-plus-15-minutes>>,
       "jti": <<random-base62-nonce>>,
       "suggested_name": "Dash Wallet",
       "code": "123-456"
     }),
     "signature": ec256Sign(header + '.' + payload)
   }
   ```
   Note: `kid` is superfluous, but included to check that the key thumbprint has been calculated correctly
   by both parties
4. Allow the option to resend a new self-signed request.

The user will be prompted by their live wallet to accept the payment protocol gateway request. \
It may request that they enter the code, or it may just show it. \
It may show the suggested name, or it may not.

### 2. Connection Test

If an empty signed message returns `200 OK`, then the user has accepted live wallet payment protocol gateway.

```txt
POST https://live-wallet.example.com/me/
Content-Type: application/jose+json
Authorization: Bearer <<shared-secret-instead-of-kid-and-sig>>

{
  "protected": base64UrlEncode({
    "alg": "ES256",
    "kid": "<<key-thumbprint>>",
  }),
  "payload": base64UrlEncode({
    "iat": <<seconds-since-epoch>>,
    "exp": <<seconds-since-epoch-plus-60-seconds>>,
    "jti": <<random-base62-nonce>>
  }),
  "signature": ec256Sign(header + '.' + payload)
}
```

The merchant app should show the user an error that distinguishes between a network level failure (ex: DNS failed to resolve, `502 BAD GATEWAY`, `500 INTERNAL ERROR`, etc) and `4xx` response which indicates the user should send another request (ex: `403 FORBIDDEN`).

TODO: The live wallet may send the merchant app notification via webhook as an alternative to polling for a connection test.

### 3. Payment Request

If a Payment Protocol Gateway has selected, the app that's requesting money should send a signed request to the Gateway
rather than to app.

```txt
POST https://live-wallet.example.com/me/
Content-Type: application/jose+json
Authorization: Bearer <<shared-secret-instead-of-kid-and-sig>>

{
  "protected": base64UrlEncode({
    "alg": "ES256",
    "kid": "<<jwk-thumbprint>>"
  }),
  "payload": base64UrlEncode({
    "iat": <<seconds-since-epoch>>,
    "exp": <<seconds-since-epoch-plus-60-seconds>>,
    "jti": <<random-base62-nonce>>,
    "payment_protocol": "dash:r=https://example.com/i/12345678"
  }),
  "signature": ec256Sign(header + '.' + payload)
}
```

Payment protocol requests will be responded to automatically, or according to the live wallet rules. \
For example: the user may need to tap "Approve" on a notification for requests over Đ1.0, or if more than Đ10.0 is requested over a rolling window of 7 days, etc.

### Appendix: Proof of Concept Request Example

```txt
POST https://live-wallet.example.com/me/
Content-Type: application/jose+json
Authorization: Bearer <<shared-secret-instead-of-jwk-and-sig>>

{
  "protected": base64UrlEncode({
    "alg": "none"
  }),
  "payload": base64UrlEncode({
    "iat": 1658903036,
    "exp": 1658903963,
    "jti": "HheL9rZ5t5NkhENOtks7mw",
    "suggested_name": "Dash Wallet",
    "code": "V2C-44W",
    "shared_secret": "xGsczJTgJyyjvtLKkhOfMw"
  }),
  "signature": null
}
```

## References

- JOSE
  - iOS (Swift): https://iosexample.com/a-framework-for-the-jose-standards-jws-jwe-and-jwk-written-in-swift/
  - Android: https://connect2id.com/products/nimbus-jose-jwt/examples/jws-with-ec-signature
- ECDSA Keypair Generation
  - iOS: https://developer.apple.com/documentation/cryptokit/p256/keyagreement/privatekey
    - Ex: https://getstream.io/blog/ios-cryptokit-framework-chat/
  - Web: https://rootprojects.org/keypairs/
- JWK Thumbprint
  - https://kjur.github.io/jsrsasign/sample/tool_jwktp.html
  - https://stackoverflow.com/a/42590106
  - https://datatracker.ietf.org/doc/html/rfc7638
