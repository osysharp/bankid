# Osysharp.BankId — Swedish BankID, in about 170 lines of Osy#

Authenticate and sign with [BankID](https://www.bankid.com) — animated QR for another device, autostart for the same
one — from classes written entirely in Osy# over the public `Http.*` surface.

## Use case

Anything in Sweden that has to know who someone actually is: signing in to a service, approving a payment, signing a
contract or a form, an admin action that needs a real identity behind it. BankID is the identity layer nearly every
Swedish adult already has.

It has no user interface of its own — you draw the QR and write the page — so there is nothing to screenshot.

## Mutual TLS

BankID authenticates the **client** with a certificate and will not talk to anyone without one — as do most bank,
insurer and government APIs in the Nordics. The package declares it with a `clientCert` grant, and the app grants it:

```osy
// app.osy
app Bidrag {
  model "**/*.osy";
  use Osysharp.BankId@0 {
    egress     "appapi2.test.bankid.com";
    clientCert "BankIdCert";
  }
}

app.Secrets = [ new Secret("BankIdCert"), new Secret("BankIdCertPassword") ];
```
```console
$ osy secret set BankIdCert --file FPTestcert.p12 --base64
$ osy secret set BankIdCertPassword qwerty123
```

⭐ **The kit is granted the identity, not the key.** `clientCert` is deliberately not `secret`: the platform
authenticates as that certificate on the kit's behalf, and nothing in this package can read a byte of it. There is no
`ApiKey` property on the client — a test asserts the absence of an `Authorization` header, because a kit that grew
one would pass every other test in the file.

The certificate is never installed on the machine. It lives in your app's secret store and is loaded into memory per
call.

## Use

```osy
using Osysharp.BankId;

BankIdClient Bank() {
  return new BankIdClient { BaseUrl = "https://appapi2.test.bankid.com/rp/v6.0" };
}

string StartLogin() {
  // The END USER's address, as the platform received the request — never the server's own.
  var order = Bank().Auth(Request.ClientAddress, "Sign in to Bidrag.");
  // Keep OrderRef and QrStartSecret SERVER-SIDE. The page gets one QR string per second, and nothing else —
  // `Qr.Encode(frame)` turns it into rows of modules to draw.
  return order.OrderRef ?? "";
}

string Frame(string qrStartToken, string qrStartSecret, int secondsElapsed) {
  return Bank().QrData(qrStartToken, qrStartSecret, secondsElapsed);
}

string Finish(string orderRef) {
  var r = Bank().Collect(orderRef);
  if (r.Status == "complete") { return r.Completion?.User?.PersonalNumber ?? ""; }
  if (r.Status == "pending")  { return ""; }          // keep polling, about every two seconds
  throw new ValidationException($"BankID order failed: {r.HintCode}");
}
```

`BaseUrl` **defaults to production**. Point it at the test host while developing. Defaulting to test would be the
worse mistake — an app that shipped still pointing there would authenticate nobody while looking like it worked.

## The animated QR is the part worth having a kit for

The QR is not static. It is recomputed **every second** from a secret, so a screenshot of it is worthless a second
later. That is what stops the oldest attack on this flow: a phishing page showing the victim a QR from the attacker's
own login.

```
bankid.<qrStartToken>.<seconds>.<HMAC-SHA256(qrStartSecret, seconds)>
```

Get it subtly wrong and everything still *looks* right, because the frame at `t=0` is the one you photograph while
testing. So the kit is checked against **BankID's own published vector** — their token, their secret, their expected
output — which is the only kind of test that can catch an algorithm being confidently wrong.

⛔ **Compute it server-side and send only the result.** `qrStartSecret` must never reach the browser: anyone holding
it can generate every future frame for that order. A test asserts the secret does not appear in the frame.

## Four things that bite everyone

| | |
|---|---|
| **`endUserIp` is the END USER's IP** | Not your server's. It works perfectly in development, where they are the same machine, and silently destroys BankID's fraud signals in production. Refused here if missing, naming the distinction. `Request.ClientAddress` is the right value. |
| **v6.0 takes no personal number** | You cannot start an order for a named person any more. The user is identified by scanning the QR or by autostart on their own device. Code written against v5 is not missing a parameter — the flow changed. |
| **`userVisibleData` is base64** | The kit encodes it. Passing already-encoded text puts a screen of base64 in front of the user, which you only discover by looking at a phone. |
| **Cancel abandoned orders** | A live order blocks the next one with `alreadyInProgress`, so a page the user navigated away from locks them out for minutes — and the error names a transaction they cannot see. |

`alreadyInProgress` and `unauthorized` both get their real cause spelled into the failure message, because both point
somewhere other than the call that failed. `unauthorized` is almost always a test certificate against production, or
the reverse.

## Getting a test certificate

BankID publishes the RP test certificate openly — no agreement, no cost. Get the current file and its password from
the [technical guides](https://www.bankid.com/en/utvecklare/guider); the filename and API version both move, so take
them from there rather than from a blog. Test host is `https://appapi2.test.bankid.com/rp/v6.0`.

To complete a flow you also need a **test BankID** on a phone, which is a separate thing from the RP certificate —
the test build of the app plus an issued test BankID. Production needs an RP agreement through a Swedish bank.

## Scope

The kit covers `auth`, `sign`, `collect` and `cancel`, the animated QR and the autostart link. Two things are outside it:

- **`/phone/auth` and `/phone/sign`** — orders initiated over the telephone.
- **Verifying the returned signature** — the kit hands you `signature` and `ocspResponse`; store them, because they
  cannot be recovered from BankID later. Validating the XML signature is the app's step.

## Verified against the real test environment

Both of these were checked against BankID's test environment with a real certificate.

**The test host DOES need its CA trusted, and this is the one to trust.** `appapi2.test.bankid.com` serves a single
leaf — no intermediates — issued directly by `CN=Test BankID SSL Root CA v1 Test`, which is in no public trust store.
So `serverCa "BankIdCa"` is not optional for the test environment; without it the handshake fails before anything
BankID-shaped happens, and the error is about TLS rather than about your integration.

The root is published in BankID's *Relying Party Guidelines* §8 (the same section that gives the `qwerty123`
passphrase), as a PEM you extract from the PDF. ⚠ **Do not take it from a search result.** A search readily turns up
`CN=BankID SSL Root Certification Authority TEST` — a *different* root, issued 2008 and expired in 2019, which is a
plausible-looking wrong answer. **The check that settles it costs one command**, and it is the right
one to run on any root before trusting it:

```console
$ openssl s_client -connect appapi2.test.bankid.com:443 \
    -servername appapi2.test.bankid.com -CAfile <candidate>.pem </dev/null | grep "Verify return"
Verify return code: 0 (ok)
```

⭐ **Both files are checked in, under [`testcert/`](testcert/)** — BankID publishes them for anyone, passphrase and
all. That directory's README has the two commands below ready
to paste. It is in this repository for you to use, and NOT in the package an app installs: `package.osy` globs
`model/*.osy` and `tests`, so nothing under `testcert/` is in the release archive or reaches an app.

**And a real `/auth` went through.** Client certificate `FP Testcert 5`, the root above, no other trust:

```
POST https://appapi2.test.bankid.com/rp/v6.0/auth   {"endUserIp":"192.168.0.1"}
HTTP 200
{"orderRef":"01a0df65-…","autoStartToken":"b09f7d75-…","qrStartToken":"90ab5fbc-…","qrStartSecret":"d590c1b9-…"}
```

Every field the kit declares on the auth response came back populated — which is the one thing a stubbed test
structurally cannot prove. A completed flow — `collect` after a person approves — needs a test BankID on a phone, and
is outside this check.

## Source and tests

Everything is in `model/` — two files, no C# anywhere. `tests/` is the producer's own project, including
BankID's published QR vector. It does not ship, and `osy publish` runs it and refuses a version that fails.

