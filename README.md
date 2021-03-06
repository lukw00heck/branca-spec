# Branca Token

Authenticated and encrypted API tokens using modern crypto.

## What?

Branca is a secure easy to use token format which makes it hard to shoot yourself in the foot. It uses IETF XChaCha20-Poly1305 AEAD symmetric encryption to create encrypted and tamperproof tokens. Payload itself is an arbitrary sequence of bytes. You can use for example a JSON object, plain text string or even binary data serialized by [MessagePack](http://msgpack.org/) or [Protocol Buffers](https://developers.google.com/protocol-buffers/).

Although not a goal, it is possible to use [Branca as an alternative to JWT](https://appelsiini.net/2017/branca-alternative-to-jwt/). Also see [getting started](https://branca.io/) instructions.

This specification defines the external format and encryption scheme of the token to help developers create their own implementations. Branca is closely based on [Fernet specification](https://github.com/fernet/spec/blob/master/Spec.md).

## Design Goals

1. Secure
2. Easy to implement
3. Small token size

## Token Format

Branca token consists of header, ciphertext and an authentication tag. Header consists of version, timestamp and nonce. Putting them all together we get following structure.

```
Version (1B) || Timestamp (4B) || Nonce (24B) || Ciphertext (*B) || Tag (16B)
```

String representation of the above binary token must use base62 encoding with the following character set.


```
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxy
```

### Version

Version is 8 bits ie. one byte. Currently the only version is `0xBA`. This is a magic byte which you can use to quickly identify a given token. Version number guarantees the token format and encryption algorithm.

### Timestamp

Timestamp is 32 bits ie. unsigned big endian 4 byte UNIX timestamp. By having a timestamp instead of expiration time enables the consuming side to decide how long tokens are valid. You cannot accidentaly create tokens which are valid for the next 10 years.

Storing timestamp as unsigned integer allows us to avoid 2038 problem. Unsigned integer overflow will happen in year 2106.

### Nonce

Nonce is 192 bits ie. 24 bytes. These should be cryptographically secure random bytes and never reused between tokens.

### Ciphertext

Payload is encrypted and authenticated using [IETF XChaCha20-Poly1305](https://download.libsodium.org/doc/secret-key_cryptography/xchacha20-poly1305_construction.html). Note that this is [Authenticated Encryption with Additional Data (AEAD)](https://tools.ietf.org/html/rfc7539#section-2.8) where the he header part of the token is the additional data. This means the data in the header (version, timestamp and nonce) is not encrypted, it is only authenticated. In laymans terms, header can be seen but it cannot be tampered.

### Tag

The authentication tag is 128 bits ie. 16 bytes. This is the
[Poly1305](https://en.wikipedia.org/wiki/Poly1305) message authentication code. It is used to make sure that the payload, as well as the non-encrypted header have not been tampered with.

## Working With Tokens

Instructions below assume your crypto library supports combined mode. In combined mode the authentication tag and the encrypted message are stored together. If your crypto library does not provide combined mode the `tag` is last 16 bytes of the `ciphertext|tag` combination.

### Generating a Token

Given a 256 bit ie. 32 byte secret `key` and an arbitrary `payload`, generate a token with the following steps in order:

1. Generate a cryptocraphically secure `nonce`.
2. If user has not provided `timestamp` use the current unixtime.
3. Construct the `header` by concatenating `version`, `timestamp` and `nonce`.
4. Encrypt the user given payload with IETF XChaCha20-Poly1305 AEAD with user    provided secret `key`. Use `header` as the additional data for AEAD.
5. Concatenate the `header` and the returned `ciphertext|tag` combination from step 4.
6. Base62 encode the entire token.

### Verifying a Token

Given a 256 bit ie. 32 byte secret `key` and a `token` to verify that the `token` is valid and recover the original unencrypted `payload`, perform the following steps, in order.

1. Base62 decode the token.
2. Make sure the first byte of the decoded token is `0xBA`.
3. Extract the `header` ie. the first 29 bytes from the decoded token.
4. Extract the `nonce` ie. the last 24 bytes from the `header`.
5. Extract the `timestamp` ie. bytes 2 to 5 from the `header`.
6. Extract `ciphertext|tag` combination ie. everything starting from byte 30.
7. Decrypt and verify the `ciphertext|tag` combination with IETF XChaCha20-Poly1305    AEAD using the secret `key` and  `nonce`. Use `header` as the additional data for    AEAD.
8. Optionally if the user has specified a `ttl`, when verifying the token add the `ttl` to `timestamp` and compare this to current unixtime.

## Libraries

Currently known implementations in the wild.

| Language | License | Crypto library used |
| -------- | ------- | ------------------- |
| [Elixir](https://github.com/tuupola/branca-elixir) |  MIT | [ArteMisc/libsalty](https://github.com/ArteMisc/libsalty) |
| [Go](https://github.com/hako/branca) | MIT | [GoKillers/libsodium-go](https://github.com/GoKillers/libsodium-go)
| [JavaScript](https://github.com/tuupola/branca-js) |  MIT | [jedisct1/libsodium.js](https://github.com/jedisct1/libsodium.js) |
| [PHP](https://github.com/tuupola/branca-php) | MIT | [paragonie/sodium_compat](https://github.com/paragonie/sodium_compat) |
| [Python](https://github.com/tuupola/branca-python) | MIT | [jedisct1/libsodium](https://github.com/jedisct1/libsodium) |
## Acceptance Test Vectors

TODO... In the meanwhile see [JavaScript](https://github.com/tuupola/branca-js/blob/master/test.js) and [PHP](https://github.com/tuupola/branca-php/blob/master/tests/BrancaTest.php) example tests.

## Similar Projects

* [PASETO](https://github.com/paragonie/paseto) ie. Platform-Agnostic Security Tokens.
* [Fernet](https://github.com/fernet) which provides AES 128 in CBC mode tokens.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.