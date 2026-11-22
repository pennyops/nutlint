# nutlint

[![check-nuts-vectors](https://github.com/pennyops/nutlint/actions/workflows/vectors.yml/badge.svg)](https://github.com/pennyops/nutlint/actions/workflows/vectors.yml)

Independent, dependency-free checker for the Cashu protocol test vectors in
[cashubtc/nuts](https://github.com/cashubtc/nuts) (`tests/*.md`).

It parses the markdown test-vector files directly and re-derives every value
with a small standalone implementation written from the spec text (pure
Python 3 standard library: secp256k1 math, BIP340 Schnorr, BIP32/BIP39, CBOR).
No wallet or mint codebase is imported, so a passing run means the published
vectors agree with the spec prose, and a failing run points at either a vector
or a spec ambiguity worth fixing.

Built from scratch for [Nut November 2026](https://nutnovember.org) (#NutNovember):
the whole project (first commit 2026-11-19, MIT) and the spec pull requests it led to
(cashubtc/nuts #484-#489, #491) were made during November. Nothing existed before.

Formerly named nutcheck; renamed so it is not confused with [nutcheck.org](https://nutcheck.org), an unrelated
#NutNovember project that grades live Cashu mints. This repo checks the
*specification* (test vectors and prose examples in cashubtc/nuts) offline;
it never talks to a mint unless you pass `--fetch`.

Use it to:

- sanity-check vector changes in a NUT pull request before merging;
- port Cashu to a new language: the modules are short, readable reference code;
- learn how each NUT computes its values, step by step.

## Usage

```
git clone --depth 1 https://github.com/cashubtc/nuts
python3 -m nutlint ./nuts
```

Or install it as a command (no dependencies):

```
pip install git+https://github.com/pennyops/nutlint@v0.1.2   # or without @v0.1.2 for latest main
nutlint ./nuts
```

Sample output (excerpt):

```
PASS  NUT-12  hash_e
PASS  NUT-12  nonce vector: A == a*G
PASS  NUT-12  nonce vector: C_ == a*B_
PASS  NUT-12  deterministic e
PASS  NUT-12  deterministic s
PASS  NUT-12  deterministic proof verifies
PASS  NUT-12  BlindSignature DLEQ valid
PASS  NUT-12  Proof DLEQ valid (reblinded)
...
WARN  NUT-18  Complete Payment Request: vector uses standard base64 ('+' or '/'), spec says base64_urlsafe
WARN  NUT-18  HTTP Transport Payment Request: vector uses standard base64 ('+' or '/'), spec says base64_urlsafe
WARN  NUT-26  nut10.d is not a hex hash/pubkey: a]0e66820b... (vector typo; JSON and TLV agree)
WARN  NUT-20  20.md example mint request: signature does not verify for the quoted pubkey
WARN  NUT-25  25.md example mint request: quote has a pubkey but the request has no signature
WARN  NUT-05  05.md:191 example melt quote (PENDING) is missing request
WARN  NUT-05  05.md:210 example melt quote (PAID) is missing method
WARN  PROSE   11.md:250 example point 02e2aeb97f47690e... is not on secp256k1
WARN  PROSE   14.md:141 example point 02e2aeb97f47690e... is not on secp256k1
WARN  PROSE   29.md:180 example point 0208657b2917f469... is not on secp256k1
WARN  PROSE   29.md:181 example point 037dc0b5e712a39e... is not on secp256k1
243 checks, 243 passed, 0 failed
```

`WARN` lines are spec problems that do not break a vector: example values in the prose that disagree with the spec's own rules or code (bad example points, signatures that do not verify, missing fields, wrong base64 alphabet). Each one above has an open fix upstream (see Coverage). They do not change the exit code.

Exit code 0 = all checks pass, 1 = at least one vector disagrees.

## Inspect a token (`nutlint token`)

The same independent code also works on real tokens. Paste a `cashuA`/`cashuB` token to decode it and run
basic checks (amounts, points on the curve, duplicate secrets, NUT-10 secret kind). With mint keys it also
verifies each NUT-12 DLEQ proof offline, so a receiver can check that the proofs were really signed by that
mint's keyset without contacting the mint about the token itself:

```
nutlint token cashuB...                    # decode + structural checks, fully offline
nutlint token cashuB... --keys keys.json   # verify DLEQ against a saved /v1/keys response
nutlint token cashuB... --fetch            # download the public keyset from the token's mint, then verify
nutlint token creqA...                     # decode a NUT-18 payment request
```

```
version : cashuB (V4 CBOR)
mint    : https://mint.example
unit    : sat
proofs  : 2, total 10 sat
ok    [0]      8 keyset 00ad268c4d1f5826 secret=random DLEQ valid
FAIL  [1]      2 keyset 00ad268c4d1f5826 secret=random DLEQ INVALID  (DLEQ INVALID)
```

`tests/test_inspect.py` (run in CI) builds a token with DLEQ proofs from a known test key, checks that it
passes, then flips one bit of a proof and checks that it is rejected. The tool never spends, swaps, or
sends the token anywhere; `--fetch` only downloads the mint's public keys.

## Use as a GitHub Action

Run nutlint in CI on a checkout of the spec (for example on every pull request to cashubtc/nuts or a fork):

```yaml
- uses: actions/checkout@v4
- uses: pennyops/nutlint@v0.1.2
  with:
    path: .   # the spec checkout
```

## Check a spec pull request before merging

nutlint runs on any checkout of the spec, so a reviewer can test a proposed change to the vectors or examples:

```sh
git clone https://github.com/cashubtc/nuts && cd nuts
git fetch origin pull/488/head:pr488 && git checkout pr488   # any PR number
cd .. && python3 -m nutlint ./nuts
```

This also works on the proposed Keysets v3 branch ([cashubtc/nuts#443](https://github.com/cashubtc/nuts/pull/443)): nutlint reads superseded NUTs from `legacy/` and re-derives the new NUT-28 slot-map vectors (slot0/slot1 blinding scalars and keys). On Nov 19, 2026, #443 and the PRs stacked on it passed 174/174 (re-checked Nov 20 on heads updated overnight and again on the Nov 20 afternoon heads, e.g. #443 at 10d8a69: still 174/174; #490 passes 243/243). Re-checked Nov 22 on the latest heads (#443 cb1190a, #445 e7c3bea, #457 29993c5, #464 611640b): 174/174 each.

A PR that edits a test vector should still end with `0 failed`; a new `FAIL` line points at the vector that no longer matches the code path it describes.

## Coverage

| NUT | What is checked |
|-----|-----------------|
| 00 | hash_to_curve, blinded messages, blinded signatures; TokenV3 encode/decode and prefix rejection; TokenV4 CBOR decode + byte-exact re-encode; raw `crawB` binary token |
| 01 | keyset validation: reject truncated / uncompressed keys, accept u64 amounts up to 2^63 |
| 02 | keyset ID v1 (`00`) and v2 (`01`, unit / input_fee_ppk / final_expiry) |
| 08 | no vector file: the worked fee-return example in `08.md` (blank-output count, change amount, power-of-2 split) and the `max(ceil(log2(fee_reserve)), 1)` rule (enough outputs for any change up to the reserve) |
| 11 | BIP340 signatures; SIG_INPUTS spendability (n_sigs multisig, locktime, refund keys); SIG_ALL `msg_to_sign` hashes and full swap/melt request verification (identical conditions, first-input witness, P2PK + NUT-14 HTLC preimage, multisig, refund after locktime) |
| 12 | DLEQ `hash_e`, deterministic nonce (exact e, s), BlindSignature and Proof (reblinded) verification |
| 13 | BIP39 seed, legacy BIP32 derivation (v1), HMAC-SHA256 KDF (v2), P2PK key derivation |
| 18 | payment requests: `creqA` + CBOR decode equals the JSON, byte-exact CBOR re-encode (incl. CBOR `undefined`); WARNs on non-urlsafe base64 in vectors |
| 20 | quote-locking key derivation (m/129373'/20'/...), `msg_to_sign` bytes and hash, BIP340 mint-request signature, tamper rejection |
| 26 | `creqb` payment requests: own Bech32m (checksum const + HRP), TLV decode equals the JSON (unit compact `sat`, transports with NIP-19 npub/nprofile pubkey + relay tags, NUT-10 locks, `mp`, `sm`), byte-exact Bech32m re-encode |
| 27 | Nostr mint-backup key derivation (SHA256(seed \|\| domain separator), x-only pubkey) |
| 28 | P2BK: ECDH Zx symmetry, blinding scalars r0..r10, blinded keys P', skStd/skNeg, parity rule (chosen key really signs for P'), example proof slots |
| 29 | batch mint: offline request-shape rules (empty / duplicate quotes, signatures length), per-quote NUT-20 signature over the full outputs array |
| 14 | no vector file: the hash / preimage example pair in `14.md` (SHA256(preimage) == Secret.data) |
| prose 11 | the signed P2PK proof quoted in `11.md`: witness BIP340 signature verifies over SHA256(Proof.secret) |
| prose | every compressed point (`02`/`03` + 64 hex) quoted in the `NN.md` spec text is checked to be on secp256k1; invalid example points are WARNs |

Planned: more NUTs as vectors are published.

Findings so far (status as of Nov 19, 2026):

| nutlint output | Spec PR | Status |
|---|---|---|
| NUT-18 vectors use standard, not urlsafe, base64 | [#484](https://github.com/cashubtc/nuts/pull/484) | closed as duplicate; same fix merged in [#475](https://github.com/cashubtc/nuts/pull/475) |
| NUT-02 prose vs code: `final_expiry: 0` in the keyset ID v2 preimage | [#485](https://github.com/cashubtc/nuts/pull/485) | open |
| 4 example points in NUT-11/14/29 not on secp256k1 | [#486](https://github.com/cashubtc/nuts/pull/486) | open |
| NUT-20 example mint request signature does not verify | [#487](https://github.com/cashubtc/nuts/pull/487) | open |
| NUT-04 signature order not stated; NUT-23/25 example signatures out of output order | [#488](https://github.com/cashubtc/nuts/pull/488) | open |
| NUT-05 async melt example missing common quote fields | [#489](https://github.com/cashubtc/nuts/pull/489) | open |
| NUT-10 key-only tag example vs NUT-11 (key-only `pubkeys` is malformed) | [#491](https://github.com/cashubtc/nuts/pull/491) | open |
| NUT-26 HTLC vector `nut10.d` typo | [#477](https://github.com/cashubtc/nuts/pull/477) (not mine) | open; nutlint WARN clears on that branch |

## Is the checker itself trustworthy? (mutation self-test)

A checker that always says PASS is useless, so nutlint ships a mutation test:

```
python3 -m nutlint.mutate ./nuts [--per-file N] [--seed S]
```

It flips one hex digit inside random long hex values of each vector file,
re-runs the checks on the mutated text, and reports how many corruptions were
caught. Current score against cashubtc/nuts HEAD (seed 1, 8 per file):
67/79 (85%). Uncaught locations are listed by file and line. They are
values the vectors only show as examples and that nothing is derived from,
e.g. NUT-01 keyset maps (only the key format is specified), the NUT-29
keyset `id` (not part of the signed message), or NUT-11 secret nonces in
examples that come without a signature to check against.

## Warning

The crypto here is NOT constant time and is meant only for checking public
test vectors. Never use it with real keys or funds.

## License

MIT
