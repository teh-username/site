---
title: Time-based One-Time Password (TOTP) Primer
date: 2019-11-17 20:47:55
tags:
  - totp
  - security
  - 2fa
  - otp
---

The Time-based One-Time Password algorithm (TOTP) has always been an interesting concept for me as it allows two systems to agree on a set of numbers that change over time even while network partitioned! This opens up a broader set of devices that can serve as a second factor of authentication (something you have) aside from the traditional username-password combo (something you know).

In this episode, we'll take a peek under the hood of the TOTP algorithm and implement a version of it using JavaScript. Spoiler alert: See it in action on [https://www.2faas.dev/](https://www.2faas.dev/) and [GitHub](https://github.com/teh-username/2faas.dev).

We'll be emulating how [Google Authenticator](https://github.com/google/google-authenticator) does the OTP computation so we'll tweak some of the parts to abide by some [assumptions](https://github.com/google/google-authenticator/wiki/Key-Uri-Format).

## Preliminaries

Before diving into the specifics of TOTP, we'll have to work our way through some of the prior art that it utilizes. The "timeline" looks something like:

[RFC 2104 (HMAC)](https://tools.ietf.org/html/rfc2104) -> [RFC 4226 (HOTP)](https://tools.ietf.org/html/rfc4226) -> [RFC 6238 (TOTP)](https://tools.ietf.org/html/rfc6238)

### MAC and HMAC

[Message Authentication Code or MAC](https://en.wikipedia.org/wiki/Message_authentication_code) are bits of information that are sent along with a message that enables us to verify whether the message hasn't been tampered with. HMAC (Hash-based MAC) is a variant of MAC that utilizes, well, hashes to perform its job. [RFC 2104](https://tools.ietf.org/html/rfc2104) defines it as:

`H(K XOR opad, H(K XOR ipad, message))`

where:

* H - The hash function of your choice (e.g. SHA-1)
* K - Secret key shared between the 2 parties
* opad - The byte `0x5C` repeated
* ipad - The byte `0x36` repeated

I'm not gonna pretend to be an expert on HMAC so let me leave you this [Computerphile video](https://www.youtube.com/watch?v=wlSG3pEiQdc) about it.

To implement HMAC, we'll utilize the [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API) that should be present on most modern browsers.

We'll first have to ["import"](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/importKey) a key that specifies the shared secret and the type of hashing algorithm we are going to use (for reasons we'll see later, we'll go with the "SHA-1" algorithm). We will then ["sign"](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/sign) the "message" using the derived key.

We can implement it as follows:

```js
crypto.subtle.importKey(
  'raw',
  secret,
  { name: 'HMAC', hash: {name: 'SHA-1'} },
  false,
  ['sign']
).then((key) => {
  return crypto.subtle.sign('HMAC', key, message)
})
```

### HOTP

Up the chain is the HMAC-based One-Time Password algorithm (HOTP) which, as the name implies, is an OTP implementation using HMAC. Since we are now in the realm of OTP generation, we'll be replacing the "message" with some arbitrary value (called a counter) that changes over time and is known (or can be derived) by both parties. [RFC 4226](https://tools.ietf.org/html/rfc4226) explains in full the details about HOTP and can be summarized as:

`HOTP(K, C) = Truncate(HMAC-SHA-1(K, C))`

where:

* Truncate - Computes the OTP from the HMAC output
* HMAC-SHA-1 - HMAC but using the SHA-1 hash function
* K - Shared key between the two parties
* C - Originally a message, but is now an increasing counter value known only to the two parties

The result we get from HMAC-SHA-1 is usually a 20-byte value, which is nowhere close to being usable as an OTP. This is where the `Truncate` function comes in. Given the result from HMAC-SHA-1, it basically extracts `N` (common is 6) digits dynamically as our OTP.

We can then implement HOTP as follows:

```js
// https://tools.ietf.org/html/rfc4226#section-5.1
// Counter is expected to be an 8-byte value. Convert and pad if needed.
let formatCounter = (counter) => {
  let binStr = ('0'.repeat(64) + counter.toString(2)).slice(-64);
  let intArr = [];

  for (let i = 0; i < 8; i++) {
    intArr[i] = parseInt(binStr.slice(i * 8, i * 8 + 8), 2);
  }

  return Uint8Array.from(intArr).buffer;
};

// https://tools.ietf.org/html/rfc4226#section-5.4
let truncate = (buffer) => {
  let offset = buffer[buffer.length - 1] & 0xf;
  return (
    ((buffer[offset] & 0x7f) << 24) |
    ((buffer[offset + 1] & 0xff) << 16) |
    ((buffer[offset + 2] & 0xff) << 8) |
    (buffer[offset + 3] & 0xff)
  );
};

crypto.subtle.importKey(
  'raw',
  secret,
  { name: 'HMAC', hash: {name: 'SHA-1'} },
  false,
  ['sign']
).then((key) => {
  return crypto.subtle.sign('HMAC', key, formatCounter(counter))
}).then((result) => {
  // Make sure we keep any leading zeroes
  return ('000000' + (truncate(new Uint8Array(result)) % 10 ** 6 )).slice(-6)
});
```

The RFC also includes [test values](https://tools.ietf.org/html/rfc4226#page-32) that we can use to verify whether our implementation is correct.

### TOTP

The TOTP algorithm is an extension of the HOTP where instead of using a counter, we utilize time instead (Unix time to be more specific). [RFC 6238](https://tools.ietf.org/html/rfc6238) defines TOTP as follows:

`TOTP = HOTP(K, T)`

where:

* K = Shared key between the two parties
* T = (Current Unix time - T0) / X
  * T0 = Time to start the count (Default: 0)
  * X = Time step in seconds or how long the computed OTP is valid for (Default: 30s)

Implementing TOTP simply involves creating a wrapper on top of the HOTP function such as:

```js
let counter = Math.floor(Date.now() / 30000);
return computeHOTP(secret, counter);
```

### à la Google Authenticator

One tweak we'll be adding to our implementation would be expecting the [secret to being encoded in Base32](https://github.com/google/google-authenticator/wiki/Key-Uri-Format#secret), so we'll have to include a [decoder](https://git.coolaj86.com/coolaj86/unibabel.js/src/branch/master/unibabel.base32.js#L82) in our code. This way, our implementation and Google Authenticator should arrive at the same values.

Our final implementation is now:

```js
/*
  https://github.com/google/google-authenticator/wiki/Key-Uri-Format
  Assumptions (based from Google Authenticator):
    Algorithm: SHA1
    Digits: 6
    Period: 30s
*/
let computeHOTP = (secret, counter) => {
  // https://tools.ietf.org/html/rfc4226#section-5.1
  let formatCounter = (counter) => {
    let binStr = ('0'.repeat(64) + counter.toString(2)).slice(-64);
    let intArr = [];

    for (let i = 0; i < 8; i++) {
      intArr[i] = parseInt(binStr.slice(i * 8, i * 8 + 8), 2);
    }

    return Uint8Array.from(intArr).buffer;
  };

  // https://tools.ietf.org/html/rfc4226#section-5.4
  let truncate = (buffer) => {
    let offset = buffer[buffer.length - 1] & 0xf;
    return (
      ((buffer[offset] & 0x7f) << 24) |
      ((buffer[offset + 1] & 0xff) << 16) |
      ((buffer[offset + 2] & 0xff) << 8) |
      (buffer[offset + 3] & 0xff)
    );
  };

  return crypto.subtle.importKey(
    'raw',
    base32ToBuffer(secret),
    { name: 'HMAC', hash: {name: 'SHA-1'} },
    false,
    ['sign']
  ).then((key) => {
    return crypto.subtle.sign('HMAC', key, formatCounter(counter))
  }).then((result) => {
    return ('000000' + (truncate(new Uint8Array(result)) % 10 ** 6 )).slice(-6)
  });
};

let computeTOTP = (secret) => {
  let counter = Math.floor(Date.now() / 30000);
  return computeHOTP(secret, counter);
}
```

### Verification

To verify everything is working, scan the following QR code with your Google Authenticator App:

{% asset_img totp_qr.png Sample TOTP QR %}

Or if you are on mobile, enter the following secret:

```
IRUWIIDZN52SAZLWMVZCA2DFMFZCA5DIMUQHI4TBM5SWI6JAN5TCARDBOJ2GQICQNRQWO5LFNFZSAVDIMUQFO2LTMU7Q
```

Then visit [https://www.2faas.dev/](https://www.2faas.dev/) and enter the secret above. The OTP generated should be the same as what Google Authenticator should show!

## Resources

* [How To Implement Google Authenticator Two Factor Auth in JavaScript](https://hackernoon.com/how-to-implement-google-authenticator-two-factor-auth-in-javascript-091wy3vh3)
* [Generating 2FA One-Time Passwords in JS Using Web Crypto API](https://dev.to/al_khovansky/generating-2fa-one-time-passwords-in-js-using-web-crypto-api-1hfo)
* [2FA QR code generator](https://stefansundin.github.io/2fa-qr/)
* [Browser Authenticator](https://rootprojects.org/authenticator/)
