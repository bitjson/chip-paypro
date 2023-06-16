# CHIP-2023-05-PayPro: Bitcoin Cash Payment Protocol

        Title: Bitcoin Cash Payment Protocol
        Type: Standards
        Layer: Applications
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2023-05-31
        Latest Revision Date: 2023-05-31
        Version: 0.1.0

<details>

<summary><strong>Table of Contents</strong></summary>

- [Summary](#summary)
- [Deployment](#deployment)
- [Motivation](#motivation)
- [Benefits](#benefits)
- [Technical Specification](#technical-specification)
- [Rationale](#rationale)
- [Test Vectors](#test-vectors)
- [Implementations](#implementations)
- [Feedback & Reviews](#feedback--reviews)
- [Acknowledgements](#acknowledgements)
- [Changelog](#changelog)
- [Copyright](#copyright)

</details>

## Summary

A standard for sharing Bitcoin Cash and CashToken payment information via Uniform Resource Identifiers (URIs) and a web-based payment protocol.

## Deployment

This proposal does not require coordinated deployment.

## Motivation

Existing standards for sharing payment information ([BIP21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki), [BIP70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki), [BIP71](https://github.com/bitcoin/bips/blob/master/bip-0071.mediawiki), [BIP72](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki), [BIP73](https://github.com/bitcoin/bips/blob/master/bip-0073.mediawiki), [JSON Payment Protocol](https://github.com/bitpay/jsonPaymentProtocol/blob/master/v1/specification.md), and [JSON Payment Protocol v2](https://github.com/bitpay/jsonPaymentProtocol/blob/master/v2/specification.md)) do not address standardization of payments using CashTokens and also fail to accommodate recent market changes: the widespread usage of alternative cryptocurrency systems, the existence of long-term splits in cryptocurrencies, and the evolution of industry practices in cryptocurrency acceptance.

Additionally, existing standards require developers to reconcile implementation details from multiple, sometimes-conflicting documents. In some cases, portions of standards have fallen out of use by industry, but standards documents do not accurately reflect current practice. This proposal aims to serve as a primary reference document for new Bitcoin Cash payment protocol implementations and as a foundation for future standards.

## Benefits

_(wip)_

## Technical Specification

### Technical Summary

- Wallets should indicate support for receiving and sending CashTokens by appending `?t` (or `&t`) to backwards-compatible (version `0` and `1`) addresses.
- To request specific transfers of CashTokens, wallets must use CashToken-supporting addresses ([v2 and v3 CashAddresses](https://cashtokens.org/docs/spec/chip#cashaddress-token-support)) and new payment URI parameters: `c` (category), `f` (fungible token amount), `n` (NFT commitment), and `s` (satoshis).
- The specification maintains compatibility with existing applications of [BIP21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki), [BIP73](https://github.com/bitcoin/bips/blob/master/bip-0073.mediawiki), [JSON Payment Protocol](https://github.com/bitpay/jsonPaymentProtocol/blob/master/v1/specification.md), and [JSON Payment Protocol v2](https://github.com/bitpay/jsonPaymentProtocol/blob/master/v2/specification.md).
- New [MIME types](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types) and relevant extensions to [BIP72](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki), [BIP73](https://github.com/bitcoin/bips/blob/master/bip-0073.mediawiki), [BCMR](https://cashtokens.org/docs/bcmr/chip), and [JSON Payment Protocol v2](https://github.com/bitpay/jsonPaymentProtocol/blob/master/v2/specification.md) are defined.

### Bitcoin Cash Payment URIs

A Bitcoin Cash Payment URI is:

- A request for a single transaction paying to an address using a single output, where all provided parameters modify the properties of the requested output, or
- Connection information for another protocol by which Bitcoin Cash and/or CashToken payment information is to be transmitted.

A non-normative summary of the payment URI syntax:

```
bitcoin[cash]:[address][?[t][&s=<satoshis>][&e=<expiration_timestamp>][&m=<message>][&r=<request_url>][&c=<category_hex>][&f=<fungible_token_amount>][&n=<nft_commitment_hex>][&<param>=<value>]]
```

Compatibility with [BIP21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki) is also supported:

```
bitcoin[cash]:[address][?[t][&amount=<decimal_amount_BCH>][&message=<message>][&r=<request_url>]]
```

In further detail:

- The URI scheme must be `bitcoincash` or `bitcoin` (see [Support for `bitcoin` URI scheme](#support-for-bitcoin-uri-scheme)).
- The `address` is a Bitcoin Cash address in [CashAddress format](https://reference.cash/protocol/blockchain/encoding/cashaddr); if `address` is not provided, the `r` parameter (Request URL) is required.
- Any number of optional query parameters may follow `?`, in any order, with each definition separated by `&`.

#### Payment URI Parameters

| Parameter         | Description                                                                                                                                                                                                                                                                                                                                                                                                                                         | Additional Validation When Present                                                                                                                                                                                                         |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `c`               | The category identifier of the requested token(s), hex-encoded in user interface order<sup>1</sup>.                                                                                                                                                                                                                                                                                                                                                 | Error if `f` and/or `n` are not set. Error if address is not CashToken-supporting (e.g. [version `2` and `3` CashAddresses](https://cashtokens.org/docs/spec/chip#cashaddress-token-support)). Error if `amount` is set (`s` may be used). |
| `e`               | The Unix timestamp at which this URI expires. For the purpose of expiration, payments are considered to have occurred at the moment they are heard over the public network (rather than the time at which they are confirmed in a block).                                                                                                                                                                                                           | Error if the indicated expiration time has passed. Error if `amount` is set (`s` may be used).                                                                                                                                             |
| `f`               | The fungible token amount requested in category `c` tokens. The amount is in terms of on-chain fungible token units, without respect for display standards. E.g. for a category in which `100` fungible tokens are considered to be `1.00 XAMPL`, `f=100` would be a request for `1.00 XAMPL`.                                                                                                                                                      | Error if `c` is not set.                                                                                                                                                                                                                   |
| `m`               | A message that describes the transaction to the user. The message must be [utf8-based, percent encoded](https://en.wikipedia.org/wiki/URL_encoding); uppercase letters should also be percent encoded for reliable transmission<sup>2</sup>.                                                                                                                                                                                                        | Error if `message` is also set.                                                                                                                                                                                                            |
| `n`               | The requested non-fungible token commitment of category `c`. The commitment must be hex-encoded, e.g. `n=01` requests an NFT with commitment `0x01`; `n` or `n=` requests an NFT with an empty commitment.                                                                                                                                                                                                                                          | Error if `c` is not set.                                                                                                                                                                                                                   |
| `r`               | An `HTTPS` request URL, similar to [BIP72](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki), but used by the [below described Payment Protocol](#payment-protocol). The `https://` is implied and must be omitted (see [Behavior of `r` parameter](#behavior-of-r-parameter)). If provided, all other parameters are ignored (other parameters may be used as a fallback for implementations without full payment protocol support). |                                                                                                                                                                                                                                            |
| `s`               | The requested number of satoshis, e.g. `s=123456` is a request for `0.00123456 BCH`. If `s` is not set, the minimum value for the requested output (dust limit) is implied.                                                                                                                                                                                                                                                                         | Error if `amount` is also set.                                                                                                                                                                                                             |
| **Compatibility** |
| `t`               | The receiver can safely receive CashTokens. Implied by CashToken-supporting [v2 and v3 CashAddresses](https://cashtokens.org/docs/spec/chip#cashaddress-token-support). The `t` parameter should only be used for backwards-compatibility; specific token requests must use a token-aware address (see the `c` parameter).                                                                                                                          | Error if `amount`, `c`, or `s` is also set.                                                                                                                                                                                                |
| `amount`          | The requested amount in whole BCH, e.g. `amount=.00123456` is a request for `0.00123456 BCH`. Included for backwards-compatibility with BIP21, but the `s` parameter is recommended for applications not requiring BIP21 compatibility. (See [Inclusion of `s` parameter](#inclusion-of-s-parameter).)                                                                                                                                              | Error if `s` is also set.                                                                                                                                                                                                                  |
| `message`         | Identical to `m`, included for backwards-compatibility with BIP21. Note that the [BIP21 `label` parameter is not supported](#exclusion-of-label-parameter).                                                                                                                                                                                                                                                                                         | Error if `m` is also set.                                                                                                                                                                                                                  |
| `<custom>`        | An unknown, custom parameter that implementations can safely ignore.                                                                                                                                                                                                                                                                                                                                                                                |                                                                                                                                                                                                                                            |

<details>

<summary>Notes</summary>

1. This is the byte order typically seen in block explorers and user interfaces (as opposed to little-endian byte order, which is used in standard P2P network messages). For reference, the genesis block header in this byte order is little-endian – `000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f` – and can be produced by this script: `<0x0100000000000000000000000000000000000000000000000000000000000000000000003ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4a29ab5f49ffff001d1dac2b7c> OP_HASH256 OP_REVERSEBYTES`.
2. Most percent-encoding libraries do not percent-encode uppercase letters; for lossless transmission over case-insensitive protocols like [Payment URI Alphanumeric Mode](#payment-uri-alphanumeric-mode), URI generators must also percent-encode all uppercase letters. An example JavaScript implementation is provided below.

##### Percent-Encoding of URI Messages

```js
/**
 * Encode a message for inclusion in a payment URI. This method differs from
 * `encodeURIComponent` in that it encodes uppercase letters and does not encode
 * `:`, `/`, or `$` (all transmissible in alphanumeric mode).
 */
const encodeMessage = (message) =>
  encodeURI(message).replace(
    /[^A-Za-z0-9 $%*-\./:]|(?<!(?:%)|(?:%[A-F0-9]))[A-Z]/g,
    (char) => '%' + char.charCodeAt(0).toString(16).toUpperCase()
  );
/**
 * Decode a percent-encoded message which may have been converted to uppercase
 * for transmission in alphanumeric mode. As uppercase letters are percent
 * encoded, all other letters are assumed to be intended for lower case display.
 */
const decodeMessage = (message) => decodeURIComponent(message.toLowerCase());
```

</details>

<details>

<summary><strong>Payment URI Test Vectors</strong></summary>

The following examples demonstrate well-formed payment URIs.

| Description                                                          | Parameters                                                                                     | URI                                                                                                                                                          |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| v0 address                                                           |                                                                                                | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl`                                                                                                     |
| v0 address, token support                                            | `t`                                                                                            | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t`                                                                                                   |
| v0 address, token support, web-safe URI                              | `t`                                                                                            | `bitcoin:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t`                                                                                                       |
| v0 address, token support, backwards-compatible message              | `t`, `message` of `test`                                                                       | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t&message=test`                                                                                      |
| v0 address, token support, message                                   | `t`, `m` of `Tip for Alice at Venue`                                                           | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t&m=%54ip%20for%20%41lice%20at%20%56enue`                                                            |
| v0 address, token support, message, expiration                       | `t`, `m` of `Tip for Bob at Venue`, expiration of `2022-11-15T12:00:00.000Z`                   | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t&e=1668513600&m=%54ip%20for%20%42ob%20at%20%56enue`                                                 |
| v2 address (implicit token support)                                  |                                                                                                | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v`                                                                                                     |
| v2 address (implicit token support), web-safe URI                    |                                                                                                | `bitcoin:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v`                                                                                                         |
| v0 address, satoshis, expiration                                     | `s=123400`, expiration of `2023-05-15T12:00:00.000Z`                                           | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?s=123400&e=1684152000`                                                                               |
| Payment Protocol-Only                                                | `r=example.com/pay/1234`                                                                       | `bitcoincash:?r=example.com/pay/1234`                                                                                                                        |
| Payment Protocol-Only, web-safe URI                                  | `r=example.com/pay/1234`                                                                       | `bitcoin:?r=example.com/pay/1234`                                                                                                                            |
| Payment Protocol; falls back to v0 address, BIP21 amount             | `r=example.com/pay/1234`, `amount=.001234`                                                     | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?amount=.001234&r=example.com/pay/1234`                                                               |
| Payment Protocol; falls back to v0 address, satoshis, and expiration | `r=example.com/pay/1234`, `s=123400`, expiration of `2023-05-15T12:00:00.000Z`                 | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?s=123400&e=1684152000&r=example.com/pay/1234`                                                        |
| Payment Protocol; falls back to v0 address, token support, message   | `r=example.com/tip/1234`, `t`, `m` of `Tip for Bob at Venue`                                   | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?r=example.com/tip/1234&t&m=%54ip%20for%20%42ob%20at%20%56enue`                                       |
| v0 address, BIP21-compatible amount and message                      | `amount=1` (`1.00000000 BCH`), `message` of `Test at ACME (10% Friends & Family Discount)`     | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?amount=1&message=%54est%20at%20%41%43%4D%45%20%2810%25%20%46riends%20%26%20%46amily%20%44iscount%29` |
| v0 address, satoshi amount and message                               | `s=12345` (`0.00012345 BCH`), `m` of `Multi\nLine\nMessage: T̶̀est`                              | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?s=12345&m=%4Dulti%0A%4Cine%0A%4Dessage%3A%20%54%CD%80%CC%B6est`                                      |
| v2 address, 10,000 FT request, minimum satoshis                      | `c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428`,`f=10000`                 | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10000`                          |
| v2 address, 10,000 FT request, 1000 satoshis                         | `c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428`, `f=10000`, `s=1000`      | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10000&s=1000`                   |
| v2 address, 10,000 FT request, zero-byte NFT, minimum satoshis       | `c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428`, `f=10000`, `n`           | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10000&n`                        |
| v2 address, 10,000 FT request, zero-byte NFT, 1000 satoshis          | `c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428`, `f=10000`, `n`           | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10000&n&s=1000`                 |
| v2 address, NFT commitment, minimum satoshis                         | `c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428`, NFT commitment of `0x01` | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&&n=01`                            |
| v0 address, unknown optional parameter                               | `custom=value`                                                                                 | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?custom=value`                                                                                        |

The following examples demonstrate malformed payment URIs. For each, implementations should consider returning the error message, `This payment request uses an invalid or unknown encoding. ${error}`, with each `error` indicated below:

| Error                                                             | URI                                                                                                                                           |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| No address or request URL is provided.                            | `bitcoincash:?amount=1`                                                                                                                       |
| No address or request URL is provided.                            | `bitcoincash:?s=1000`                                                                                                                         |
| Token-accepting request requires a token-aware address.           | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t&amount=1`                                                                           |
| Token-accepting request requires a token-aware address.           | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t&s=1000`                                                                             |
| Token-accepting request requires a token-aware address.           | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t&c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10000`         |
| Expiring request uses the deprecated `amount` parameter.          | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?amount=1&e=1684152000`                                                                |
| Both `s` and `amount` are set.                                    | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?s=123456&amount=.00123456`                                                            |
| Both `m` and `message` are set.                                   | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?m=abc&message=abc`                                                                    |
| Token request is missing a fungible amount and/or NFT commitment. | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428`                   |
| Token request pays to an address without token support.           | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10`              |
| Token request pays to an address without token support.           | `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t&c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10`            |
| Token request uses the ambiguous `amount` parameter.              | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10&amount=.0001` |
| Token request is missing a token category.                        | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?f=10`                                                                                 |
| Token request is missing a token category.                        | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?n`                                                                                    |
| Token request is missing a token category.                        | `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?n=00`                                                                                 |

</details>

#### Payment URI Alphanumeric Mode

Payment URI Alphanumeric Mode is defined to enable high-density encoding of payment URIs in [QR codes](https://en.wikipedia.org/wiki/QR_code). To convert a payment URI to alphanumeric mode:

1. Convert all characters to uppercase.
2. Convert reserved characters to their alphanumeric mode equivalent:
   1. `?` becomes `:`,
   2. `=` becomes `-`, and
   3. `&` becomes `+`.
3. Optionally, instances of `%20` (space) and `%24` (`$`) may be decoded within `m`/`message` values for maximum compression.

Use of alphanumeric mode is recommended for all QR-encoded payment URIs requesting tokens and for use cases in which BIP21 backwards-compatibility is not required.

<details>

<summary><strong>Alphanumeric Mode Test Vectors</strong></summary>

| URI                                                                                                                                                  | Alphanumeric Mode                                                                                                                      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl`                                                                                             | `BITCOINCASH:QR7FZMEP8G7H7YMFXY74LGC0V950J3R2959LHTXXSL`                                                                               |
| `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?t`                                                                                           | `BITCOINCASH:QR7FZMEP8G7H7YMFXY74LGC0V950J3R2959LHTXXSL:T`                                                                             |
| `bitcoincash:?r=example.com/pay/1234`                                                                                                                | `BITCOINCASH::R-EXAMPLE.COM/PAY/1234`                                                                                                  |
| `bitcoin:?r=example.com/pay/1234`                                                                                                                    | `BITCOIN::R-EXAMPLE.COM/PAY/1234`                                                                                                      |
| `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?s=1234`                                                                                      | `BITCOINCASH:QR7FZMEP8G7H7YMFXY74LGC0V950J3R2959LHTXXSL:SAT-1234`                                                                      |
| `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v?c=0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428&f=10000`                  | `BITCOINCASH:ZR7FZMEP8G7H7YMFXY74LGC0V950J3R295Z4Y4GQ0V:C-0AFD5F9AD130D043F627FAD3B422AB17CFB5FF0FC69E4782EEA7BD0853948428+F-10000`    |
| `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl?s=1000&m=%54est%20at%20%41%43%4D%45%20%2810%25%20%46riends%20%26%20%46amily%20%44iscount%29` | `BITCOINCASH:QR7FZMEP8G7H7YMFXY74LGC0V950J3R2959LHTXXSL:S-1000+M-%54EST AT %41%43%4D%45 %2810%25 %46RIENDS %26 %46AMILY %44ISCOUNT%29` |

</details>

### Payment Protocol

A JSON-based payment protocol is defined for Bitcoin Cash and CashToken payments; this protocol is fully compatible with and extends the multi-asset [JSON Payment Protocol v2](https://github.com/bitpay/jsonPaymentProtocol/blob/master/v2/specification.md).

_Work in progress_:

- Support [BIP72](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki) and [BIP73](https://github.com/bitcoin/bips/blob/master/bip-0073.mediawiki)-style GET requests:
  - `Accept` = `application/bitcoincash-payment-request+0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428,application/bitcoincash-payment-request,application/payment-options`
    - User preference: category `0afd5f9ad130d043f627fad3b422ab17cfb5ff0fc69e4782eea7bd0853948428`, BCH, or falls back to `application/payment-options`
  - `x-paypro-version` = `2`
- `application/payment-options`:
  - `currency` is either `"BCH"` or hex-encoded CashToken `category`
  - `network` may be `main`, `chip`, `test4`, or a BCMR split ID
- `paypro` extension for BCMR identities, uses [Key Distribution](https://github.com/bitpay/jsonPaymentProtocol/blob/master/v2/specification.md#key-distribution) format, omit `owner` (superseded by identity `name`).

## Rationale

### Selection of URI Parameters

The URI parameters specified by this proposal reflect changes in the marketplace since [BIP21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki) was initially introduced:

- **Less-than-expected demand within URI parameter namespace** – BIP21 anticipated that many new, application-specific parameters would be added by specific wallets and implementations. This growth in URI parameter usage did not materialize in the ensuing decade; new use cases instead focused on changes or expansions to various payment protocols.
- **Increased importance of brevity** – both scannable codes and Near Field Communication (NFC) became widely adopted by nearly every modern computing device; QR codes became a primary feature of mainstream cryptocurrency usage (especially as camera performance improved and costs fell), and globally, NFC became a common interface for in-person payments. In both cases, saving bytes within payment URIs became more important: additional bytes can require increased QR code size or larger NFC tag storage capacity.

Given these changes in the marketplace, this proposal aims to maximize URI efficiency in the long-term by defining an upgrade path from the loosely BIP21-based standard used by today's Bitcoin Cash ecosystem. All new URI parameters use a single identifying character, the `message` parameter may be optionally shortened to `m`, `amount` is [replaced by `s`](#inclusion-of-s-parameter) in token use cases (and optionally in other cases), and an [alphanumeric mode for QR codes](#inclusion-of-alphanumeric-mode) is introduced.

### CashAddress Version Requirements

While this proposal allows backwards-compatible payment URIs to indicate support for receiving CashTokens by appending `?t`/`&t`, payment URIs requesting specific payments in CashTokens are required to include a CashToken-supporting address (e.g. [version `2` or `3` CashAddresses](https://cashtokens.org/docs/spec/chip#cashaddress-token-support)). This requirement prevents pathological cases in which a wallet without token support quietly ignores token-related parameters and misleads users into making incorrect payments.

To illustrate, consider a payment URI requesting a $1 stablecoin payment: a token-unaware wallet might quietly ignore unknown parameters (`c`, `f`, `s`, etc.), and instead simply pre-fill the address in a payment view. Unaware of the issue, some users might manually enter "$1", causing the wallet to compute an equivalent value payment in BCH rather than the requested stablecoin. If the user sends this payment, the receiver will not only have received a different asset than expected (at an exchange rate chosen by the user's wallet), but refunding this payment will incur support and operational costs.

To prevent such errors, token-requesting payment URIs are required to use CashToken-supporting addresses.

### Inclusion of `s` parameter

This proposal requires use of the new `s` (satoshis) parameter rather than the existing BIP21 `amount` parameter for payment URIs requesting CashTokens; this policy is valuable for payment error prevention, privacy, and encoding efficiency.

The word "amount" can be reasonably interpreted as either an amount of Bitcoin Cash or a fungible token amount, so usage of the `amount` parameter in newer, token-specific applications would increase the risk of misinterpretation and software bugs.

Additionally, the whole-amount rounding bias of the `amount` parameter is incompatible with good wallet privacy practices: rounded values make transaction clustering analysis easier, so well-designed wallets should almost never send round amounts. For this reason, it can be concluded that **privacy-preserving amount values are more efficiently described in terms of satoshis**. Amounts greater than 1 BCH save at least one character (a decimal), and amounts smaller than 1 BCH may drop multiple placeholder `0` digits; these savings are in addition to the 5 characters saved by the shorter parameter name.

Note, existing applications may continue to use the BIP21 `amount` parameter indefinitely; this requirement applies only to new applications creating token-specific payment URIs.

### Exclusion of `label` parameter

The [BIP21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki) `label` parameter is not commonly acknowledged by modern wallets as it is easily used in attacks that mislead novice users. Additionally, labelling individual addresses encourages poor security and privacy practices like widespread address publication and address reuse.

This proposal considers `label` to be deprecated; it can be safely ignored by implementations. [Bitcoin Cash Metadata Registries](https://cashtokens.org/docs/bcmr/chip) are a superior solution for transmitting authenticated addresses and address-like information, and the `m` parameter (or equivalent, BIP21-compatible `message` parameter) is generally a safer choice for all other use cases in which `label` might have previously been used.

### Behavior of `r` parameter

This proposal differs from [BIP72](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki) in that the `r` parameter does not support alternatives to the `HTTPS` protocol scheme. This is the established behavior of many BIP72 implementations, but lack of clarity in the initial specification has prevented many payment URI generators from omitting the `https://` prefix, increasing URI length and the resulting complexity of QR-encoded URIs.

Note that this clarification does not inhibit the use of alternative protocols where a special Top-Level Domain (TLD) is established to indicate use of the protocol. For example, the [Tor](<https://en.wikipedia.org/wiki/Tor_(network)>) network uses the [`.onion` TLD](https://en.wikipedia.org/wiki/.onion) in Tor addresses. Additionally, if future standards require indication of request URIs using non-`HTTPS` protocol schemes, an alternative parameter name can be assigned for that use.

### Exclusion of fee rate parameter

This proposal does not include a parameter for conveying a minimum required network fee rate (e.g. `k`, a minimum fee rate per kilobyte). While a fee rate parameter might help merchants reduce payment errors during unusual network activity, the additional specification, implementation, and testing complexity outweigh any potential value.

On Bitcoin Cash, the minimum required network fee rate is a relay policy not enforced by block validation consensus. (As of 2023, the fee rate is conventionally set to 1000 satoshis per kilobyte.) In practice, this minimum required fee rate can only rise in cases where network transaction throughput exceeds the prevailing consensus block size limit for a sustained period. As the prevailing limit is set to a significant multiple of expected peak usage, such network conditions only occur during attempted network-wide denial of service attacks in which an attacker creates enough fee-paying transactions to crowd out real users over multiple blocks. Under these circumstances, wallets of real users should begin to temporarily increase their selected fee rate to have their transactions prioritized over the attacker; to continue impacting real network usage, the attacker must match or exceed these higher fee rates, geometrically increasing the cost of the attack at each step.

Given this background, including a minimum fee rate parameter in the [proposed URI scheme](#bitcoin-cash-payment-uris) is unlikely to meaningfully reduce payment errors during attacks: wallet software must already be capable of adapting fee rates to network conditions, so any hinting in URI requests simply duplicates this expected capability with greater delay (the URI's fee rate is necessarily computed before the reading wallet's fee rate). Additionally, the specified [payment protocol](#payment-protocol) enables merchants to specify a minimum fee rate, so applications with stricter settlement requirements can always specify a payment protocol request URL (`r` parameter) without fallback URI parameters.

### Inclusion of Alphanumeric Mode

This proposal defines a [payment URI alphanumeric mode](#payment-uri-alphanumeric-mode) which is specifically optimized for encoding efficiency within alphanumeric-encoded QR codes.

Alternatively, this proposal could standardize a "binary mode", where URI contents are instead packed into a binary-encoded QR code. However, such a mode would require significantly more specification and implementation complexity, and the binary mode would only reduce QR code complexity by a negligible amount beyond the proposed alphanumeric mode:

- CashAddresses already include carefully-designed, application-specific encoding and error correction:
  - **Encoding** – mixed case and several easily confusable characters are omitted. This is important in edge cases where a user might be required to manually type or read an address aloud.
  - **Error detection** – applications can detect up to 6 random errors or up to 8 character bursts of errors within most addresses. Error detection in binary-encoded QR codes would be less efficient and more opaque to end users.
- CashAddresses encode information at 5 bits per character; alphanumeric-encoded QR codes coincidentally encode information at 5.5 bits per character (for similar reasons), so only minimal theoretical efficiency gains are possible from encoding address information using QR code binary encoding. These minimal gains would require significant additional development and maintenance across all implementations.
- The density of other URI parameters could be increased via binary encoding, but this would again require significant increases in implementation complexity, and gains would only apply to a relatively small subset of real-world use cases:
  - Payment Request URLs (`r`) are already very densely-packed by alphanumeric mode.
  - Messages (`m` or `message`) with primarily lowercase characters are very densely-packed by alphanumeric mode.
  - Token categories (`c`) and NFT commitment requests (`n`) only appear in more specialized URIs; they could be more efficiently packed either by a binary mode or by another encoding within alphanumeric mode (like bech32), but this would have negligible impact on QR code complexity<sup>1</sup>.
  - Numeric parameters (`f` and `s`) occupy a minimal percentage of total URI length, so encoding savings are unlikely to be worth additional implementation complexity.

<details>
<summary>Notes</summary>

1. For example, `07a70ec6e0a325991e829daea5de1be1bb71e1ef4a04931cdddf567d8f60f676` might be encoded as `q7nsa3hq5vjej85znkh2thsmuxahrc00fgzfx8xamat8mrmq7emq` (`libauth.encodeBech32(libauth.regroupBits({bin: libauth.hexToBin('07a70ec6e0a325991e829daea5de1be1bb71e1ef4a04931cdddf567d8f60f676'), sourceWordLength: 8, resultWordLength: 5 }))`), saving 12 alphanumeric characters, but any URI with `c` (category) defined must also define an address and either `f` (fungible amount) or `n` (NFT commitment). (Note, if a request URL – `r` – were defined, none of the aforementioned parameters are required to be encoded in the URI itself, so again, density is not improved.) Altogether, we can expect the minimum overhead in this URI to be 113 characters (prefix and address: 54, `c` parameter and bech32 encoded category: 55, additional `f` or `n` parameter overhead: 4). At this length, the 12 character reduction does not reduce QR code complexity at all: regardless of bech32 encoding, both QR codes would be 53x53 modules. (at `Quartile` QR code quality – 25% error correction).

</details>

### Support for `bitcoin` URI scheme

This proposal recommends that wallets support the `bitcoin` URI scheme in addition to the `bitcoincash` scheme; the `bitcoin` URI receives [special treatment by web browsers](https://html.spec.whatwg.org/#safelisted-scheme), so wallets that register support for `bitcoin` may be activatable in some contexts where browser policy might interfere with the `bitcoincash` URI scheme. Additionally, it's common for multi-coin Bitcoin Cash wallets to also support BTC, so these wallets are easily capable of distinguishing and using both URI schemes.

Finally, most wallets can expect little overlap in real usage between BCH and BTC usage of `bitcoin` URIs. Much of the BTC ecosystem has transitioned from [BIP21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki) to alternative payment information sharing strategies (often for secondary networks); for example, [BOLT11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) recommends the `lightning` URI scheme and considers the `bitcoin` scheme to be a fallback measure. If meaningful usage overlap is expected – and the wallet does not also support BTC – the wallet may simply offer users the option to copy or re-open the `bitcoin` URI in a different wallet.

### Design of Payment URI Error Behavior

This proposal strictly defines expected error behavior in parsing of payment URIs – unexpected conflicts are never ignored. This behavior minimizes subtle differences between implementations and maximizes flexibility for future standards: future standards can safely replace error behaviors without risk that non-supporting clients will partially-misinterpret the new standard.

### Exclusion of Web Payment Request API

To reduce scope, this proposal does not attempt to standardize the use of Bitcoin Cash or CashTokens in the [Web Payment Request API](https://developer.mozilla.org/en-US/docs/Web/API/Payment_Request_API); future CHIPs might provide usage recommendations.

## Implementations

The following software is known to support Bitcoin Cash Metadata Registries:

_(pending initial implementations)_

## Feedback & Reviews

- [CHIP-BCMR Issues](https://github.com/bitjson/chip-uris/issues)
- [`BIP21 Query Parameters for CashTokens` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/bip21-query-parameters-for-cashtokens/1015)

## Acknowledgements

Thank you to [Jonas Lundqvist](https://github.com/jonas-lundqvist), [Tom Zander](https://github.com/zander), [Yashasvi S. Ranawat](https://github.com/yashasvi-ranawat), [Joemar Taganna](https://github.com/joemarct), [Mathieu Geukens](https://github.com/mr-zwets), and [bitcoincashautist](https://github.com/A60AB5450353F40E) for reviewing and contributing ideas leading to this proposal, providing feedback, and promoting consensus among stakeholders.

## Changelog

This section summarizes the evolution of this document.

- **Draft v1.0.0**
  - Initial publication
  - Dropped support for `req-` parameter prefix ([#1](https://github.com/bitjson/chip-paypro/issues/1))
  - Restricted use of `t` to reasonable cases ([#2](https://github.com/bitjson/chip-paypro/issues/2))
  - Defined `e` (expiration) parameter ([#6](https://github.com/bitjson/chip-paypro/issues/6))

## Copyright

This document is placed in the public domain.
