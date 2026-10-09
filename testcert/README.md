# BankID test credentials — checked in on purpose

Everything here is **published by BankID for anyone to use against their test environment**, passphrase included. It
is committed so that anyone who wants to make a real BankID call has it in one place.

⛔ **None of it authenticates anything real.** The client certificate is issued by `Testbank A AB (publ)`, a fictional
bank, and is accepted only by `appapi2.test.bankid.com`. Pointing it at `appapi2.bankid.com` fails at the handshake.
There is no production credential in this repository and none should ever be added to it — a real RP certificate
comes from a Swedish bank under an agreement, belongs in `osy secret set`, and is the one thing this directory must
never be used as a precedent for.

⚠ **A secret scanner will flag `FPTestcert5_20240610.p12`,** because it holds a private key and the passphrase is two
lines below. That alert is correct about the shape and wrong about the risk. If you are triaging one: this is
[BankID's own published test certificate](https://developers.bankid.com/test-portal/test-information), downloadable by
anyone, and rotating it would mean asking BankID to rotate a file they hand out deliberately.

## The files

| file | what it is |
|---|---|
| `FPTestcert5_20240610.p12` | **The one to use.** RP client certificate + private key, PKCS#12, AES-256-CBC. Same format BankID Keygen produces for production. |
| `FPTestcert5_20240610.pem` | The same pair in PEM — certificate first, then the encrypted key. |
| `FPTestcert5_20240610-legacy.pfx.pfx` | The same pair under `pbeWithSHA1And40BitRC2-CBC`, for tooling too old for the above. The doubled extension is how BankID ships it. |
| `test-bankid-ssl-root-ca-v1.pem` | **`CN=Test BankID SSL Root CA v1 Test`** — the SERVER root, extracted from the *Relying Party Guidelines* §8. See below; this is the file that is genuinely hard to find. |
| `BANKID-README.sv.md` | BankID's own README from the archive, unedited (Swedish). |

**Passphrase for all three client formats: `qwerty123`** — published in the same guidelines section.

## The server root is the part that is hard to get, and easy to get wrong

`appapi2.test.bankid.com` serves **one leaf, no intermediates**, issued directly by a root that is in no public trust
store and that the host does not send. So the call fails on TLS before anything BankID-shaped happens, and the error
(`tls 20`, *unable to get local issuer certificate*) says nothing about BankID.

The root is not on the developer portal. It is a PEM block inside the *Relying Party Guidelines* PDF, §8 — the same
section that gives the passphrase above.

⛔ **A search readily turns up the WRONG root:** `CN=BankID SSL Root Certification Authority TEST`, issued
2008 and expired in 2019. A plausible name, a real BankID certificate, and useless here. **One command tells you
which you are holding**, and it is worth running on any root before trusting it:

```console
$ openssl s_client -connect appapi2.test.bankid.com:443 \
    -servername appapi2.test.bankid.com -CAfile test-bankid-ssl-root-ca-v1.pem </dev/null | grep "Verify return"
Verify return code: 0 (ok)
```

## Trying it again

A real `/auth`, straight from this directory — no app, no platform, just curl:

```console
$ curl --cert-type P12 --cert FPTestcert5_20240610.p12:qwerty123 \
       --cacert test-bankid-ssl-root-ca-v1.pem \
       -H "Content-Type: application/json" -d '{"endUserIp":"192.168.0.1"}' \
       https://appapi2.test.bankid.com/rp/v6.0/auth
{"orderRef":"…","autoStartToken":"…","qrStartToken":"…","qrStartSecret":"…"}
```

That is the shape `Osysharp.BankId` produces, and it proves the kit's wire types against something other than its own
stubs. Through the kit instead, the same two files become the two grants the package declares:

```console
$ osy secret set BankIdCert --file FPTestcert5_20240610.p12 --base64
$ osy secret set BankIdCertPassword                      # qwerty123
$ osy secret set BankIdCa   --file test-bankid-ssl-root-ca-v1.pem
```

⚑ **Granting `serverCa` NARROWS trust rather than widening it** — the package then trusts BankID's root and *not* the
public authorities, so a compromised public CA cannot mint a certificate these calls would accept.

## Completing a flow

Completing a flow needs a **test BankID on a phone**, which is a separate thing from the RP certificate: the test
build of the BankID app plus an issued test identity, from
[BankID's test portal](https://developers.bankid.com/test-portal/test-information). Without one you can start an
order and watch it sit in `pending` — enough to prove the wire, not enough to exercise `collect` to completion or the
signature that comes back with it.
