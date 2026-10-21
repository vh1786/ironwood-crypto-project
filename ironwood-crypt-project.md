# Ironwood: proved cryptography for Ada, to replace libcrypto where no certificate is required

*Project document, 21 October 2026. "Ironwood" is a working name, to be
checked before publication.*

> **Status of every claim in this document.**
> - **Measured** (21 October 2026, Linux x86-64): the Alire index and each
>   crate's version, origin and licence (`alr show`); gnatprove FSF 16.1.0
>   and OpenSSL 3.0.13 installed.
> - **Read**: FIPS 186-5 (digital signatures).
> - **The crates' claims** ("SPARK-proved", "proven against a tested
>   specification") are their authors'. Ironwood adopts nothing without
>   replaying the proof itself (§6).
> - **[to verify]**: export-control rules for publishing cryptographic
>   source, the availability and licences of test-vector suites, the
>   constant-time tooling, and anything about GNAT intrinsics not measured
>   here.

---

## 1. Summary

An Ada application that needs cryptography links OpenSSL's libcrypto
today: hundreds of thousands of lines of C reached through `pragma
Import`, `System.Address` handles, manual frees and error queues. For
**TLS** and for **certified** deployments (FIPS 140-3, Common Criteria),
that remains the right choice: a validated module is a validated module.

For everything else, the application's **own constructions** (sealing
data at rest, signing and checking its own files, deriving keys,
authenticating with HMAC, hashing passwords), Ironwood offers a single
proved SPARK library:

- **one API** over the primitives such an application uses: hashes, MACs,
  KDFs, AEADs, key agreement, signatures, password hashing, random;
- **built first from what exists**: the Alire index already holds
  SPARK-proved crates for several of them. Ironwood pins them, replays
  their proofs, audits them for constant time, and wraps them;
- **writing what is missing**: AES-GCM, ChaCha20-Poly1305 (IETF), P-384,
  RSA verification, Argon2id, HMAC and HKDF over SHA-384/512;
- **a provider interface**, so that one application chooses, per
  deployment profile, between Ironwood and a certified OpenSSL provider.
  That is the crypto-agility a product with several security levels needs.

---

## 2. The problem

### 2.1 What libcrypto costs an Ada application

- **An unproved attack surface in the middle of a proved program.** A
  SPARK application's guarantees stop at every `Import`.
- **An unsafe interface**: opaque handles as `System.Address`, ownership
  rules in the C documentation, errors read from a thread-local queue.
  Each binding is hand-written and reviewed by its user alone.
- **A large dependency for small needs**: an application that seals a
  file with an AEAD and checks an Ed25519 signature pulls a library built
  for every protocol in existence.
- **Version churn**: OpenSSL's major versions have changed APIs (the 3.0
  provider model), and legacy algorithms move to a separate provider that
  must be loaded explicitly.

### 2.2 What already exists in Ada (measured)

| crate | version | licence | what | claim |
|---|---|---|---|---|
| `sparknacl` | 4.0.1 | BSD-3-Clause | TweetNaCl re-implemented: X25519, Ed25519, XSalsa20, Poly1305, SHA-512 **[to confirm the exact set]** | "Verified SPARK 2014" |
| `hmac_ada` | 0.2.0 | Apache-2.0 | HMAC (RFC 2104) | "SPARK-proved" |
| `hkdf_ada` | 0.1.0 | Apache-2.0 | HKDF (RFC 5869) | "HKDF_SHA256 proved at Level 2" |
| `blake2b` | 0.1.1 | BSD-3-Clause | BLAKE2b (RFC 7693) | "in SPARK, proven against a tested specification" |
| `libkeccak` | 3.0.0 | BSD-3-Clause | SHA-3, SHAKE and related | SPARK; silver level claimed in its README |
| `p256_ada` | — | Apache-2.0 | NIST P-256 / ECDSA (ES256) | "with SPARK flow analysis" (flow only, not proof) |
| `bigints` | 0.2.0 | MIT | constant-time big integers | "Experimental SPARK" |
| `sha2` | 2.0.0 | ISC | SHA-2 | Ada, no SPARK claim |
| `chacha20` | 0.1.0 | MIT | ChaCha20 | Ada, no SPARK claim |
| `ml_kem` | 1.0.0 | **GPL-3.0-only** | ML-KEM (FIPS 203) | "SPARK-proved" |
| `ml_dsa` | 1.0.0 | **GPL-3.0-only** | ML-DSA (FIPS 204) | "SPARK-proved" |
| `blake3` | — | **GPL-3.0-only** | BLAKE3 | "SPARK-proved" |
| `cose_ada` | — | Apache-2.0 | COSE (RFC 9052/9053) | — |
| `system_random`, `rand_sys` | — | ISC; Apache-2.0 WITH LLVM-exception | system entropy | — |

**Not in the index** (searched): AES in any mode, GCM, P-384, RSA,
Argon2, a DRBG, a constant-time test harness.

The material is there, scattered across crates by different authors,
with different proof levels, APIs and licences. **Ironwood's first job is
assembly and assurance, not reinvention.**

### 2.3 Why not just use one of them

No single crate covers an application's set. Their proof claims range from
flow analysis to full functional proof. Constant time is claimed by some
and examined by none here. Three use GPL-3.0, which a proprietary user
cannot link. An application team that assembles them alone repeats the
same audit every other team repeats.

---

## 3. Goals and non-goals

### Goals

1. **One API** (§5) for the primitives in §4, with typed keys, nonces and
   tags (no `String` of secrets), and results rather than exceptions.
2. **Every primitive proved free of run-time errors**, with no
   justification left unexplained; functional proof where it is
   tractable (SPARKNaCl's arithmetic shows it can be done).
3. **Constant time by design and by measurement** for every operation on
   secrets (§7).
4. **Secrets wiped** when they go out of scope, in a way the compiler
   cannot elide (§7.3).
5. **Test vectors from the standards and from independent suites**, and
   differential testing against OpenSSL on every build (§8).
6. **A provider interface** so that an application picks Ironwood or a
   certified OpenSSL provider per deployment, without changing its code.
7. **Licence-clean**: every adopted crate under a permissive licence; the
   GPL-3.0 crates are not linked (§9.2).

### Non-goals

- **TLS.** An application keeps a TLS library. Ironwood offers no TLS
  stack.
- **A FIPS 140-3 or CC certificate.** A small project cannot carry a
  validation. Users who need one use the OpenSSL FIPS provider through
  the provider interface (§5.4). Ironwood's design keeps a validation
  possible later (§11), but never claims one.
- **Every algorithm OpenSSL has.** Only those an application's own
  constructions and its interoperability need (§4). Legacy algorithms are
  confined (§4.3).
- **Hardware security modules.** PKCS#11 is a separate binding; the
  provider interface can wrap it.

---

## 4. The primitives

### 4.1 Core set (milestones M1-M4)

| family | primitive | source | work |
|---|---|---|---|
| hash | SHA-256, SHA-384, SHA-512 | `sparknacl`'s SHA-512; SHA-256 and SHA-384 written, or `sha2` proved | proof, constant-time review |
| hash | SHA-3, SHAKE | `libkeccak` | replay proof |
| hash | BLAKE2b | `blake2b` | replay proof |
| MAC | HMAC-SHA-256/384/512 | `hmac_ada`, generalised | extend beyond SHA-256 |
| KDF | HKDF-SHA-256/384/512 | `hkdf_ada`, generalised | extend; SHA-384 matters for some national profiles |
| password hash | **Argon2id** (RFC 9106) | written on `blake2b` | new: memory-hard fill, bounded memory, proved indexing |
| AEAD | **ChaCha20-Poly1305** (RFC 8439, IETF nonce) | ChaCha20 core written in SPARK (the `chacha20` crate as a reference); Poly1305 from `sparknacl` | new composition, proof, vectors |
| AEAD | **AES-256-GCM, AES-128-GCM** | written | new: constant-time AES without lookup tables (bitsliced), and GHASH; the hardest item (§7.2) |
| key agreement | X25519 | `sparknacl` | wrap |
| key agreement | ECDH P-256 / P-384 | `p256_ada` for P-256 (to be proved, it is flow-only); P-384 written | new for P-384; proof for both |
| signature | Ed25519 | `sparknacl` | wrap |
| signature | ECDSA P-256 / P-384 (FIPS 186-5, read) | `p256_ada`; P-384 written | as above; deterministic nonces (RFC 6979) |
| signature | RSA verification (PKCS#1 v1.5, PSS) | written on `bigints` | new; verification only: verification needs no secret, so constant time is not at stake |
| random | system entropy (`getrandom`) | written, thin | no user-space DRBG: the kernel's is the source |

### 4.2 Post-quantum (M6)

ML-KEM (FIPS 203) and ML-DSA (FIPS 204) exist as SPARK crates, under
**GPL-3.0-only**. Three routes, for the maintainers to decide:

1. ask the author for a permissive licence (the same author publishes
   `hmac_ada`, `hkdf_ada` and `p256_ada` under Apache-2.0);
2. write clean-room implementations from FIPS 203/204;
3. ship them as an optional, separately licensed module.

Hybrid use (X25519 + ML-KEM, Ed25519 + ML-DSA) is the shape national
guidance recommends during the transition **[to verify against the ANSSI
and BSI guidance]**.

### 4.3 Legacy, confined (M5)

Some protocols an application must still *speak* use broken algorithms:
SNMPv3's HMAC-MD5-96 and HMAC-SHA1-96, DES-CBC and 3DES, SHA-1
signatures in old certificates. Ironwood offers them **only** in a
`Ironwood.Legacy` hierarchy:

- not reachable from the main API;
- each subprogram named for the protocol that needs it
  (`SNMPv3_HMAC_MD5_96`), so that a reviewer sees why it is there;
- a build switch removes the hierarchy entirely.

---

## 5. The API

### 5.1 Typed secrets

```ada
type Key_256   is limited private;     -- no copy, no "=", wiped on finalisation
type Nonce_96  is private;             -- public, but its own type
type Tag_128   is private;

procedure Import (K : out Key_256; From : in out Byte_Array)
  with Post => Is_Wiped (From);        -- the caller's copy wiped after import
```

Keys are `limited`: no assignment, so no accidental copy left on the
stack. Equality on tags and MACs is a constant-time function, never `=`.

### 5.2 Results, not exceptions

```ada
type Verdict is (Ok, Authentication_Failed, Invalid_Input, Entropy_Failed);

procedure Open (K : Key_256; N : Nonce_96; AAD, Cipher : Byte_Array;
                T : Tag_128; Plain : out Byte_Array; V : out Verdict)
  with Pre  => Plain'Length = Cipher'Length,
       Post => (if V /= Ok then Is_Zero (Plain));  -- nothing released on failure
```

The postcondition says what users get wrong most: **on a failed
authentication, no plaintext is released**.

### 5.3 Misuse resistance

- Nonces for AES-GCM and ChaCha20-Poly1305 drawn by the library by
  default (`Seal_Random_Nonce`), with the explicit-nonce form available
  and documented as the dangerous one.
- Signatures: Ed25519 and deterministic ECDSA only (RFC 6979), so that no
  signature depends on the quality of a random source at signing time.
- Password hashing: parameters as a record with floors (memory, passes)
  enforced by a precondition.

### 5.4 The provider interface

```ada
type Provider is limited interface;
procedure Seal (P : Provider; ...) is abstract;
...
package Ironwood.Providers.SPARK   -- the library itself
package Ironwood.Providers.OpenSSL -- a binding to libcrypto 3 (default or FIPS provider)
```

An application holds a `Provider'Class` chosen at start-up from its
configuration: "proved" for a civil deployment, "OpenSSL FIPS" for one
that must show a certificate. **The same test vectors run against both
providers**, which also tests the binding.

---

## 6. Assurance of the adopted crates

Each adopted crate goes through the same gate before it enters a
release:

1. **Pinned** to a commit (the Alire origin above), vendored, and its
   licence recorded.
2. **Proof replayed** with the project's gnatprove version (FSF 16.1.0
   measured here) at the project's level. The result is written in
   `docs/assurance/<crate>.md`: checks, proved, justified and why. A claim
   that does not replay is reported to the crate's author and the crate
   waits.
3. **Read** against the rules of §7: no secret-dependent branch or index,
   wiping, no allocation.
4. **Tested** with the vectors of §8.
5. **Upstream first**: fixes go to the crate's author as pull requests;
   Ironwood keeps a patch only while one is pending.

This gate is the project's main value to the ecosystem: one audit,
published, instead of every user repeating it.

---

## 7. Side channels and secrets

### 7.1 Constant time

SPARK proves the absence of run-time errors, **not** timing behaviour. The
rules are:

- no branch on a secret; selection by masking;
- no array index computed from a secret (it is a cache-timing channel);
- no early exit in comparisons of tags, MACs or keys;
- division and modulo on secrets avoided (some processors take variable
  time).

How they are held:

- a **review checklist** per unit, signed in the assurance file;
- **a measurement**: statistical timing tests in the style of dudect, on
  each operation with fixed and random secrets **[to verify the tool and
  its licence]**; a Valgrind-based taint check that flags branches and
  memory indices depending on secret memory, in the style of ctgrind
  **[to verify]**;
- **the generated assembly** of the hot loops inspected at the release's
  optimisation level: the compiler may turn a masked selection back into
  a branch.

### 7.2 AES without tables

A table-based AES leaks its key through the cache. Ironwood's AES is
**bitsliced** (constant time in software, slower). When the processor has
AES instructions, an accelerated path uses them through GCC builtins
**[to verify how GNAT reaches them: an `Intrinsic` import or a C shim]**.
Both paths give the same answers on every vector, and a test proves the
switch is taken.

### 7.3 Wiping

Secrets are wiped when they go out of scope, by a `Wipe` procedure on a
`Volatile_Full_Access` view, so that the compiler cannot remove the
store as dead **[to verify the aspect's guarantee in the GNAT reference
manual]**. `limited` key types with finalisation call it. Stack copies
are avoided by passing keys by reference (`limited` types are).

### 7.4 Fault attacks

These are out of scope for a software library. Recomputing a signature
and comparing it before release (the classic defence against fault
attacks on RSA-CRT and ECDSA) is offered as an option.

---

## 8. Testing

| source | for | status |
|---|---|---|
| the standards' own examples (RFC 8439, RFC 9106, RFC 5869, RFC 7693, FIPS 186-5, FIPS 197 for AES, SP 800-38D for GCM) | every primitive | the RFCs to read; FIPS 186-5 read |
| NIST's validation vectors (CAVP, now ACVP) | AES-GCM, SHA, HMAC, ECDSA, RSA | **[to verify the download form and terms]** |
| an independent edge-case suite (Wycheproof) | AEAD, ECDH, ECDSA, RSA edge cases: invalid curve points, malleable signatures, wrong tag lengths | **[to verify availability and licence]** |
| **differential against OpenSSL** 3.0.13 (installed) | every primitive both offer, on random inputs, in CI | measured available |
| fuzzing (the companion project Harrow) | every parser of keys, signatures and encodings | — |
| both providers (§5.4) | the same vectors through Ironwood and through the OpenSSL binding | — |

---

## 9. Licence, export control, governance

### 9.1 Licence

Apache-2.0: it matches the main adopted crates (`hmac_ada`, `hkdf_ada`,
`p256_ada`) and gives users a patent grant. BSD-3 and ISC code is
compatible inside it, with notices kept.

### 9.2 The GPL crates

`ml_kem`, `ml_dsa` and `blake3` are GPL-3.0-only. They are **not linked**
by Ironwood's default build. §4.2 lists the three routes for
post-quantum.

### 9.3 Export control

Publishing cryptographic source code is treated differently by different
jurisdictions. Some exempt publicly available source; some ask for a
notification **[to verify for the publisher's jurisdiction and for the
hosting service's]**. Settled before the first public release, and
written in `docs/export.md`.

### 9.4 Governance

- A `SECURITY.md` with a private channel and a disclosure policy.
- Every release states, per primitive: the proof level replayed, the
  constant-time measurement result, the vectors passed.
- Two maintainers review every change to a primitive.

---

## 10. Roadmap

| milestone | content | done when |
|---|---|---|
| **M0** reading and licences | the RFCs and FIPS documents of §8 read; vector suites and their terms checked; export rules settled (§9.3); the crates' authors contacted | each [to verify] point answered and the document amended |
| **M1** the assurance gate | §6 applied to `sparknacl`, `hmac_ada`, `hkdf_ada`, `blake2b`, `libkeccak` | each crate's proof replayed and its assurance file published |
| **M2** the API and the easy primitives | typed keys and wiping (§5.1, §7.3), results, the SHA-2 family, HMAC and HKDF over SHA-256/384/512, X25519, Ed25519, random | vectors and the OpenSSL differential pass; a constant-time measurement for each secret operation |
| **M3** ChaCha20-Poly1305 and Argon2id | §4.1 | RFC 8439 and RFC 9106 vectors; proof; timing measured |
| **M4** AES-GCM, P-256/P-384, ECDSA, RSA verification | the hard ones | NIST and independent vectors pass; bitsliced AES measured constant time; the accelerated path equal on every vector |
| **M5** the provider interface and legacy | §5.4, §4.3 | the same suite passes through both providers |
| **M6** post-quantum | one of §4.2's routes | FIPS 203/204 vectors pass |
| **M7** publication | crate, documentation, assurance reports | an outside review of at least the AEADs and signatures, by someone not on the project |

---

## 11. Risks and open questions

| risk | answer |
|---|---|
| A crate's proof does not replay | report upstream; the crate waits, or Ironwood writes the primitive |
| Constant time cannot be proved, only measured | measure on every release and publish the results; inspect the hot loops' assembly |
| Bitsliced AES is slow | the accelerated path when the processor allows it; ChaCha20-Poly1305 as the recommended AEAD |
| Users assume a certificate | the README and every API page say "not validated", and the provider interface points to the certified path |
| A cryptographic defect | the disclosure policy, two reviewers per change, outside review before M7 |
| The GPL crates for post-quantum | §4.2's routes; a decision before M6 |
| A validation one day | the boundaries kept clean (one module, a self-test at start-up, a known-answer test per algorithm) so that a CMVP or ANSSI evaluation remains possible, without promising one |

Open questions for the maintainers:

1. Which of §4.2's routes for post-quantum?
2. Is Argon2id published as its own crate as well (it has users beyond
   Ironwood's)?
3. Hosting and export: in which jurisdiction is the project published?

---

## 12. Repository layout

```
ironwood/
  alire.toml
  ironwood.gpr
  vendor/                         -- adopted crates, pinned, with LICENSE files
  src/
    ironwood.ads                  -- Byte_Array, Verdict, typed secrets
    ironwood-wipe.ad[sb]
    ironwood-hash-*.ad[sb]        -- SHA-2, SHA-3, BLAKE2b
    ironwood-mac-hmac.ad[sb]
    ironwood-kdf-hkdf.ad[sb]
    ironwood-kdf-argon2id.ad[sb]
    ironwood-aead-chacha20_poly1305.ad[sb]
    ironwood-aead-aes_gcm.ad[sb]  (+ aes-bitsliced, aes-accelerated)
    ironwood-ecc-x25519.ad[sb]  ironwood-ecc-p256.ad[sb]  ironwood-ecc-p384.ad[sb]
    ironwood-sign-ed25519.ad[sb]  ironwood-sign-ecdsa.ad[sb]  ironwood-sign-rsa_verify.ad[sb]
    ironwood-random.ad[sb]
    ironwood-providers.ads  ironwood-providers-spark.ad[sb]  ironwood-providers-openssl.ad[sb]
    ironwood-legacy-*.ad[sb]      -- behind a build switch
  tests/
    vectors/                      -- with their source and terms
    differential/                 -- against OpenSSL
    timing/                       -- dudect-style measurements
  fuzz/                           -- Harrow targets
  proof/
  docs/
    api.md  assurance/<crate>.md  constant-time.md  providers.md  export.md  SECURITY.md
```

---

## Appendix A — measurements behind this document

21 October 2026:

```bash
alr search --crates | grep -i -E 'nacl|hmac|hkdf|sha|blake|keccak|chacha|aes|gcm|p256|p384|rsa|argon|kem|dsa|random'
#   sparknacl, hmac_ada, hkdf_ada, sha2, hmac, blake2b, blake2s, blake3, libkeccak,
#   chacha20, p256_ada, bigints, ml_kem, ml_dsa, adare_dilithium5, system_random, rand_sys;
#   nothing for AES, GCM, P-384, RSA, Argon2
alr show <crate>      # versions, origins and licences as in §2.2
gnatprove --version   # FSF 16.1.0, Why3 1.8.2
openssl version       # OpenSSL 3.0.13
```
