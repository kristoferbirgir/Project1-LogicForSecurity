# Protocol Development Logbook

## Project Overview

**Scenario:** Open Authorization (OAuth-like)
- **A** = Alice (user who owns photos) — uppercase, variable
- **B** = Photo storage server — uppercase, variable
- **P** = Photo printing service — uppercase, variable
- **idp** = Identity Provider (like MitID) — **lowercase, honest constant** (intruder cannot instantiate)

**Goal:** A wants P to access photos stored at B without:
- Giving P access to other resources (e.g., email)
- Sharing A's password with P
- Downloading and re-uploading photos

**Assignment Constraints:**
- A does NOT have a public/private key pair (simplified in Week 2)
- A has a password `pw(A,idp)` shared with idp (NOT for encryption)
- Everyone except A has public/private key pairs
- Everyone knows idp's public key
- idp is honest and trusted by all parties (lowercase constant enforces this)

---

## Week 2 – Dolev-Yao: Barebones Protocol

### Informal Design (Before AnB)

**High-level idea:** Similar to "Login with Google" - A authorizes P through a trusted identity provider.

**Protocol sketch in natural language:**
```
1. A tells idp: "I want to authorize P to access my photos at B"
2. idp creates a signed authorization token
3. A forwards this token to P
4. P presents the token to B (encrypted to B)
5. B verifies the token and gives photos to P (encrypted to P)
```

**Design diagram:**
```
    A                   idp                  P                   B
    |                    |                   |                   |
    |-- auth request --->|                   |                   |
    |<-- signed token ---|                   |                   |
    |------- forward token ---------------->|                   |
    |                    |                   |-- present token ->|
    |                    |                   |<---- photos ------|
```

---

### Version: week2_v1.AnB

**Simplifications made (allowed in Week 2):**
- A has public/private key pair (temporary, removed in Week 3)
- All parties know all public keys
- No password authentication yet
- idp is lowercase (honest constant)

**AnB Protocol:**
```
Protocol: PhotoAuthorization_v1

Types:
  Agent A, B, P, idp;
  Function pk, photos;

Knowledge:
  A: A, B, P, idp, pk(A), pk(B), pk(P), pk(idp), inv(pk(A)), photos(A);
  B: A, B, P, idp, pk(A), pk(B), pk(P), pk(idp), inv(pk(B)), photos(A);
  P: A, B, P, idp, pk(A), pk(B), pk(P), pk(idp), inv(pk(P));
  idp: A, B, P, idp, pk(A), pk(B), pk(P), pk(idp), inv(pk(idp));

Actions:
  A -> idp: {A, P, B}(pk(idp))
  idp -> A: {A, P, B}(inv(pk(idp)))
  A -> P: {A, P, B}(inv(pk(idp)))
  P -> B: {{A, P, B}(inv(pk(idp)))}(pk(B))
  B -> P: {photos(A)}(pk(P))

Goals:
  B authenticates idp on A, P, B
  photos(A) secret between A, B, P
```

---

### OFMC Verification (Active Intruder)

**Command:** `ofmc anb/week2_v1.AnB`

**Result:** ATTACK FOUND (secrets)

**Attack explanation:**
The intruder plays A's role (A is an uppercase variable). Since idp is lowercase (honest),
the intruder cannot forge idp's signature. Instead, the intruder sends a legitimate
request to idp and manipulates the token flow so that B sends photos encrypted to a key
the intruder controls.

**Key insight:** With lowercase idp, signature forgery is impossible. The attacks are about
the intruder instantiating the variable agent roles (A, B, P).

---

### Week 2 Special Task: Static Analysis (Passive Intruder)

**Question:** If the intruder ONLY observes network traffic (passive), what can they learn?

**Messages observable on the network:**
```
Message 1: {A, P, B}(pk(idp))              (encrypted - unreadable)
Message 2: {A, P, B}(inv(pk(idp)))          (signed - content readable)
Message 3: {A, P, B}(inv(pk(idp)))          (same token)
Message 4: {{token}}(pk(B))                 (encrypted - unreadable)
Message 5: {photos(A)}(pk(P))               (encrypted - unreadable)
```

| Data | Learnable? | Reasoning |
|------|------------|-----------|
| A, P, B (identities) | Yes | Readable from signed token (msg 2) |
| photos(A) | No | Encrypted with pk(P), intruder lacks inv(pk(P)) |
| Token content | Yes | Signature does not encrypt |

**Conclusion:**
- **Passive intruder learns:** Metadata only (who authorizes whom)
- **Remains secret:** Actual photo content
- Active attacks require message injection and role manipulation

---

## Week 3 – Lazy Intruder

### Version: week3_v1.AnB

**Changes from Week 2:**
- A no longer has public/private key pair
- A authenticates to idp using shared password `pw(A,idp)`
- Password is NOT used as encryption key
- A encrypts authentication request to idp using pk(idp)
- P now encrypts the token to B: `{{token}}(pk(B))`

**AnB Protocol:**
```
Protocol: PhotoAuthorization_v2

Types:
  Agent A, B, P, idp;
  Function pk, photos, pw;

Knowledge:
  A: A, B, P, idp, pk(B), pk(P), pk(idp), pw(A,idp), photos(A);
  B: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(B)), photos(A);
  P: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(P));
  idp: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(idp)), pw(A,idp);

Actions:
  A -> idp: {A, P, B, pw(A,idp)}(pk(idp))
  idp -> A: {A, P, B}(inv(pk(idp)))
  A -> P: {A, P, B}(inv(pk(idp)))
  P -> B: {{A, P, B}(inv(pk(idp)))}(pk(B))
  B -> P: {photos(A)}(pk(P))

Goals:
  B authenticates idp on A, P, B
  photos(A) secret between A, B, P
```

---

### OFMC Verification

**Command:** `ofmc anb/week3_v1.AnB`

**Result:** ATTACK FOUND (secrets)

**Analysis:**
Same pattern as Week 2 — the intruder can play A's role and manipulate the protocol
to leak photos. The password authentication between A and idp does not prevent this
because A, B, P are all uppercase variables that the intruder can instantiate.

---

### Week 3 Special Task: Lazy Intruder Execution

**Goal:** Show that role P is executable by the intruder using constraint solving (lazy intruder).

**Setup:**
- Intruder plays P
- A, B, idp = honest agents

**P's role in the protocol:**
```
Input  (step 3): Receive token = {A, P, B}(inv(pk(idp))) from A
Output (step 4): Send {{A, P, B}(inv(pk(idp)))}(pk(B)) to B
Input  (step 5): Receive {photos(A)}(pk(P)) from B
```

**Formal Constraint Solving (Lecture Style):**

**Initial intruder knowledge IK₀:**
```
IK₀ = {A, B, i, idp, pk(B), pk(i), pk(idp), inv(pk(i))}
```

**After intercepting message 3 (A → P):**
```
IK₁ = IK₀ ∪ {{A, i, B}(inv(pk(idp)))}
```

**Constraint:** Intruder must produce `{{A, i, B}(inv(pk(idp)))}(pk(B))`

**Resolution:**
1. **Axiom:** `{A, i, B}(inv(pk(idp)))` ∈ IK₁ (intercepted token)
2. **Axiom:** `pk(B)` ∈ IK₀ (public key)
3. **Compose (encryption):** From (1) and (2), compose `{{A, i, B}(inv(pk(idp)))}(pk(B))` ✓

**After step 5:** B sends `{photos(A)}(pk(i))` encrypted to intruder's key.
```
IK₂ = IK₁ ∪ {{photos(A)}(pk(i))}
```
4. **Axiom:** `{photos(A)}(pk(i))` ∈ IK₂
5. **Axiom:** `inv(pk(i))` ∈ IK₀
6. **Decompose (decryption):** From (4) and (5), derive `photos(A)` ✓

**Conclusion:** Role P is executable by the intruder. All outgoing messages can be
constructed from IK via Axiom and Compose rules. The token received in step 3 can
be forwarded with encryption in step 4.

---

### Week 3 Summary

**Completed:**
- [x] Protocol with password authentication (week3_v1.AnB)
- [x] A has no key pair, uses password with idp
- [x] OFMC verification (secrets attack — intruder plays A)
- [x] Lazy intruder special task (P role is executable)

**Key findings:**
1. Password authentication added but doesn't prevent role manipulation
2. Role P is trivially executable — forward the token and decrypt photos
3. Token is not bound to specific recipient P

**Next steps for Week 4:**
- Add formats for type-flaw resistance
- Create separate key lookup protocol
- Address signature verification issue

---

## Week 4 – Typing and Key Lookup

### Changes from Week 3

1. **Format tags f1–f4** — Explicit constants tagging each message type for type-flaw resistance
2. **Key Lookup Protocol** — Separate protocol for A to get public keys from idp
3. **P encrypts token to B** — `{f3, P, token}(pk(B))` prevents B from accepting raw tokens
4. **photos(A) removed from A's knowledge** — A never possesses the photos

---

### Key Lookup Protocol: key_lookup.AnB

**Purpose:** A asks idp for the public key of B

**Protocol:**
```
Protocol: KeyLookup

Types:
  Agent A, B, idp;
  Function pk, pw, f5;

Knowledge:
  A: A, B, idp, pk(idp), pw(A,idp), f5;
  idp: A, B, idp, pk(idp), pk(B), inv(pk(idp)), pw(A,idp), f5;

Actions:
  A -> idp: {f5, A, B, pw(A,idp)}(pk(idp))
  idp -> A: {f5, A, B, pk(B)}(inv(pk(idp)))

Goals:
  A authenticates idp on f5, A, B, pk(B)
```

**OFMC Result:** ATTACK FOUND (strong_auth)
- This is a replay attack: the intruder replays a single response from idp to two sessions of A. Binding the response to A prevents the broader unbounded-replay issue, but a session-level replay is still possible without nonces.
- Under our threat model, this is acceptable for the key-lookup subprotocol.

---

### Main Protocol: week4_v1.AnB

**Protocol with format tags:**
```
Protocol: PhotoAuthorization_v3

Types:
  Agent A, B, P, idp;
  Function pk, photos, pw, f1, f2, f3, f4;

Knowledge:
  A: A, B, P, idp, pk(B), pk(P), pk(idp), pw(A,idp), f1, f2, f3, f4;
  B: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(B)), photos(A), f1, f2, f3, f4;
  P: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(P)), f1, f2, f3, f4;
  idp: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(idp)), pw(A,idp), f1, f2, f3, f4;

Actions:
  A -> idp: {f1, A, P, B, pw(A,idp)}(pk(idp))
  idp -> A: {f2, A, P, B}(inv(pk(idp)))
  A -> P: {f2, A, P, B}(inv(pk(idp)))
  P -> B: {f3, P, {f2, A, P, B}(inv(pk(idp)))}(pk(B))
  B -> P: {f4, B, photos(A)}(pk(P))

Goals:
  B authenticates idp on f2, A, P, B
  photos(A) secret between B, P
```

**Format tags (tuple elements, not function applications):**
| Tag | Message | Purpose |
|-----|---------|---------|
| f1 | `{f1, A, P, B, pw(A,idp)}(pk(idp))` | Auth request (5-tuple) |
| f2 | `{f2, A, P, B}(inv(pk(idp)))` | Signed token (4-tuple) |
| f3 | `{f3, P, {token}(inv(pk(idp)))}(pk(B))` | Photo request (nested) |
| f4 | `{f4, B, photos(A)}(pk(P))` | Photo response (3-tuple) |

**Important:** Format tags are placed as tuple elements (e.g., `{f1, A, P, B, ...}`) NOT as function applications (e.g., `f1(A,P,B)`). Function applications create opaque terms in OFMC that receivers cannot decompose.

---

### OFMC Verification

**Command:** `ofmc anb/week4_v1.AnB`

**Result:** ATTACK FOUND (secrets)

**Analysis:**
The intruder plays the role of A (with i as the instantiation), so idp honestly issues
a token for `f2, i, P, B`. B then sends `photos(i)` to P. Since the intruder IS i,
obtaining photos(i) is not a realistic attack — the intruder gets their own photos.
Format tags prevent type-flaw confusion but do not prevent role manipulation where
the intruder legitimately participates in the protocol as an honest agent A.

---

### Week 4 Special Task: Type-Flaw Resistance Proof

**Definition:** A protocol is type-flaw resistant if for any two messages M₁, M₂ in the
sub-messages of the protocol (SMP), if M₁ and M₂ are not variables and have a unifier σ,
then type(M₁) = type(M₂).

**Sub-Messages of Protocol (SMP):**

From message 1: `{f1, A, P, B, pw(A,idp)}(pk(idp))`
```
SMP₁ = {f1, A, P, B, pw(A,idp), pk(idp), {f1,A,P,B,pw(A,idp)}(pk(idp))}
```

From message 2: `{f2, A, P, B}(inv(pk(idp)))`
```
SMP₂ = {f2, A, P, B, inv(pk(idp)), {f2,A,P,B}(inv(pk(idp)))}
```

From message 4: `{f3, P, {f2,A,P,B}(inv(pk(idp)))}(pk(B))`
```
SMP₄ = {f3, P, {f2,A,P,B}(inv(pk(idp))), pk(B), outer}
```

From message 5: `{f4, B, photos(A)}(pk(P))`
```
SMP₅ = {f4, B, photos(A), pk(P), {f4,B,photos(A)}(pk(P))}
```

**Unification check — format tags ensure distinctness:**

1. **Msg 1 vs Msg 2:** Different leading tag (f1 ≠ f2) and different arity (5-tuple vs 4-tuple)
   → **No unifier** ✓

2. **Msg 1 vs Msg 4 outer:** f1 ≠ f3 and 5-tuple vs 3-tuple
   → **No unifier** ✓

3. **Msg 1 vs Msg 5:** f1 ≠ f4 and 5-tuple vs 3-tuple
   → **No unifier** ✓

4. **Msg 2 vs Msg 4 outer:** f2 ≠ f3 and 4-tuple vs 3-tuple
   → **No unifier** ✓

5. **Msg 2 vs Msg 5:** f2 ≠ f4 and 4-tuple vs 3-tuple
   → **No unifier** ✓

6. **Msg 4 outer vs Msg 5:** f3 ≠ f4. Second component (P vs B) and third component
   (signed token vs photos(A)) also differ structurally.
   → **No unifier** ✓

7. **Encrypted messages use different keys:** pk(idp), pk(B), pk(P), inv(pk(idp))
   → Different keys prevent cross-message confusion ✓

**Conclusion: Protocol is type-flaw resistant.**
The distinct format tags (f1, f2, f3, f4) ensure that no two message plaintexts from
different protocol steps can unify. Even without the tags, different tuple arities and key
usage would prevent most confusion, but the tags provide a clean, systematic guarantee.

---

### Week 4 Summary

**Completed:**
- [x] Key lookup protocol (key_lookup.AnB) with format tag f5
- [x] Main protocol with format tags f1–f4 (week4_v1.AnB)
- [x] OFMC verification (secrets attack — intruder plays A)
- [x] Type-flaw resistance proof (Special Task)

**Key findings:**
1. Format tags as tuple elements prevent type flaws between all message pairs
2. OFMC attack is role manipulation (intruder plays A), not a type flaw
3. photos(A) secret between B, P only (A never receives photos)

---

## Week 5 – Channels (Pseudonymous TLS)

### Changes from Week 4

1. **Replaced encryption with pseudonymous channels** — `[A] *->* idp` notation (TLS server-side auth)
2. **A has no keys** — Only needs password; TLS channels handle confidentiality/authentication
3. **Server-to-server uses mutual TLS** — `P *->* B` (both have certificates)
4. **Guessable secret goal added** — `pw(A,idp) guessable secret between A, idp`

---

### Channel Notation Reference

| Notation | Meaning | TLS Equivalent |
|----------|---------|----------------|
| `A -> B: M` | Insecure (Dolev-Yao) | No TLS |
| `A *-> B: M` | Confidential only | Encrypted, no sender auth |
| `A ->* B: M` | Authenticated only | Signed, not encrypted |
| `A *->* B: M` | Secure (both) | Mutual TLS |
| `[A] *->* B: M` | Pseudonymous secure | TLS server-side auth (A anonymous, B authenticated) |
| `B *->* [A]: M` | Reply on pseudonymous | Response on same TLS session |

---

### Main Protocol: week5_v1.AnB

**Protocol with pseudonymous channels:**
```
Protocol: PhotoAuthorization_v4

Types:
  Agent A, B, P, idp;
  Function pk, photos, pw, f1, f2, f3, f4;

Knowledge:
  A: A, B, P, idp, pw(A,idp), f1, f2, f3, f4;
  B: A, B, P, idp, pk(B), pk(idp), inv(pk(B)), photos(A), f1, f2, f3, f4;
  P: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(P)), f1, f2, f3, f4;
  idp: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(idp)), pw(A,idp), f1, f2, f3, f4;

Actions:
  [A] *->* idp: f1, A, P, B, pw(A,idp)
  idp *->* [A]: {f2, A, P, B}(inv(pk(idp)))
  [A] *->* P: {f2, A, P, B}(inv(pk(idp)))
  P *->* B: f3, P, {f2, A, P, B}(inv(pk(idp)))
  B *->* P: f4, B, photos(A)

Goals:
  B authenticates idp on f2, A, P, B
  photos(A) secret between B, P
  pw(A,idp) guessable secret between A, idp
```

**Key design decisions:**
- `[A] *->* idp` — A is pseudonymous (no client certificate), idp is authenticated (has cert)
- Token `{f2, A, P, B}(inv(pk(idp)))` still needs idp's signature because B verifies it offline (no direct channel to idp)
- `P *->* B` and `B *->* P` are mutual TLS (both servers have certificates)
- A knows no public keys — channels abstract the TLS infrastructure

---

### Special Task: Crypto Comparison (week5_v1_tls.AnB)

**Purpose:** Same protocol *without* channels, kept for side-by-side comparison. This is essentially the Week 4 protocol with format tags, showing what the channel version replaces.

**Protocol (cryptographic version for comparison):**
```
Protocol: PhotoAuthorization_v4_crypto

Types:
  Agent A, B, P, idp;
  Function pk, photos, pw, f1, f2, f3, f4;

Knowledge:
  A: A, B, P, idp, pk(B), pk(P), pk(idp), pw(A,idp), f1, f2, f3, f4;
  B: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(B)), photos(A), f1, f2, f3, f4;
  P: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(P)), f1, f2, f3, f4;
  idp: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(idp)), pw(A,idp), f1, f2, f3, f4;

Actions:
  A -> idp: {f1, A, P, B, pw(A,idp)}(pk(idp))
  idp -> A: {f2, A, P, B}(inv(pk(idp)))
  A -> P: {f2, A, P, B}(inv(pk(idp)))
  P -> B: {f3, P, {f2, A, P, B}(inv(pk(idp)))}(pk(B))
  B -> P: {f4, B, photos(A)}(pk(P))

Goals:
  B authenticates idp on f2, A, P, B
  photos(A) secret between B, P
```

---

### OFMC Verification

**week5_v1.AnB:** ATTACK FOUND (secrets)
- Same pattern as Week 4: intruder plays A=i, idp issues token for i, B sends photos(i)
- Goal `guesswhat` NOT triggered → password is safe from offline guessing ✓

**week5_v1_tls.AnB:** ATTACK FOUND (secrets)
- Same role-manipulation attack as the crypto version

**Analysis:**
Pseudonymous channels replace explicit encryption but the fundamental attack pattern is
the same: the intruder can legitimately play A and request their own photos. The important
result is that `pw(A,idp) guessable secret` is NOT violated — the password stays protected.

---

### Cryptography Replacement Analysis

| Original (crypto) | Channel Version | Replaced? |
|--------------------|-----------------|-----------|
| `{M}(pk(idp))` | `[A] *->* idp: M` | ✓ Yes |
| `{M}(pk(B))` | `P *->* B: M` | ✓ Yes |
| `{M}(pk(P))` | `B *->* P: M` | ✓ Yes |
| `{M}(inv(pk(idp)))` | Cannot replace | ✗ No |

**Token signature cannot be replaced** because:
1. B needs to verify idp's approval offline
2. B has no direct TLS session with idp
3. The token travels through A and P before reaching B
4. Signature is the only mechanism for B to verify origin

---

### Week 5 Summary

**Completed:**
- [x] week5_v1.AnB with pseudonymous channels
- [x] week5_v1_tls.AnB crypto comparison version (Special Task)
- [x] OFMC verification (secrets attack — password safe from guessing)
- [x] Analysis of what can/cannot be replaced by channels

**Key findings:**
1. `[A] *->* idp` models TLS server-side auth (A is pseudonymous, idp authenticated)
2. Password guessing is prevented by pseudonymous channels
3. Token signature must remain — B verifies offline without channel to idp

---

## Week 6 – Privacy (Guessable Passwords)

### Changes from Week 5

1. **Guessable secret goal verified** — `pw(A,idp) guessable secret between A, idp`
2. **Created insecure variant** — Hash-based challenge-response demonstrating offline guessing

---

### What is a Guessable Secret?

A guessable secret is a value (like a password) that:
- Has low entropy (dictionary word, short, predictable)
- Can be subject to **offline dictionary attacks**

**Offline attack:** Intruder observes protocol messages, then tries guessed passwords
offline to see if a guess produces the observed message.

**OFMC syntax:** `pw(A,idp) guessable secret between A, idp`

---

### Main Protocol: week6_v1.AnB

**Protocol (same as Week 5 — pseudonymous channels with guessable secret goal):**
```
Protocol: PhotoAuthorization_v5

Types:
  Agent A, B, P, idp;
  Function pk, photos, pw, f1, f2, f3, f4;

Knowledge:
  A: A, B, P, idp, pw(A,idp), f1, f2, f3, f4;
  B: A, B, P, idp, pk(B), pk(idp), inv(pk(B)), photos(A), f1, f2, f3, f4;
  P: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(P)), f1, f2, f3, f4;
  idp: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(idp)), pw(A,idp), f1, f2, f3, f4;

Actions:
  [A] *->* idp: f1, A, P, B, pw(A,idp)
  idp *->* [A]: {f2, A, P, B}(inv(pk(idp)))
  [A] *->* P: {f2, A, P, B}(inv(pk(idp)))
  P *->* B: f3, P, {f2, A, P, B}(inv(pk(idp)))
  B *->* P: f4, B, photos(A)

Goals:
  B authenticates idp on f2, A, P, B
  photos(A) secret between B, P
  pw(A,idp) guessable secret between A, idp
```

**OFMC Result:** ATTACK FOUND (secrets)
- Goal `guesswhat` NOT triggered → **password is safe from offline guessing** ✓
- Only the role-manipulation attack (intruder plays A, gets own photos)
- Pseudonymous channels prevent the intruder from observing the password

---

### Special Task: Demonstrating Offline Guessing Attack

**Why pk-encryption is NOT a good insecure example:**
Encrypting the password as `{pw(A,idp)}(pk(idp))` does NOT allow offline guessing —
the intruder cannot decrypt this ciphertext (only idp has `inv(pk(idp))`), so the intruder
has no way to verify whether a guessed password is correct.

**Hash-based challenge-response IS vulnerable:**
If idp sends a nonce NB and A responds with `h(pw(A,idp), NB)`, the intruder observes
both NB and `h(pw(A,idp), NB)` on the wire. The intruder can then guess passwords
and recompute `h(guessedPW, NB)` to check if it matches the observed hash.

**Insecure version (week6_insecure.AnB):**
```
Protocol: PhotoAuthorization_v5_insecure

Types:
  Agent A, B, P, idp;
  Number NB;
  Function pk, photos, pw, h, f2, f3, f4;

Knowledge:
  A: A, B, P, idp, pk(B), pk(P), pk(idp), pw(A,idp), h, f2, f3, f4;
  B: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(B)), photos(A), h, f2, f3, f4;
  P: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(P)), h, f2, f3, f4;
  idp: A, B, P, idp, pk(B), pk(P), pk(idp), inv(pk(idp)), pw(A,idp), h, f2, f3, f4;

Actions:
  A -> idp: A, P, B
  idp -> A: NB
  A -> idp: h(pw(A,idp), NB)
  idp -> A: {f2, A, P, B}(inv(pk(idp)))
  A -> P: {f2, A, P, B}(inv(pk(idp)))
  P -> B: {f3, P, {f2, A, P, B}(inv(pk(idp)))}(pk(B))
  B -> P: {f4, B, photos(A)}(pk(P))

Goals:
  B authenticates idp on f2, A, P, B
  photos(A) secret between B, P
  pw(A,idp) guessable secret between A, idp
```

**OFMC Result:** ATTACK FOUND (`guesswhat`) ✓

OFMC reports: `i can produce secret h(guessPW, x310)` — confirming the intruder
can verify password guesses offline by recomputing the hash with the observed nonce.

**Attack mechanism:**
1. Intruder intercepts `A -> idp: A, P, B` and `idp -> A: NB` (nonce visible)
2. Intruder intercepts `A -> idp: h(pw(A,idp), NB)` (hash visible)
3. Intruder guesses password `guessPW`, computes `h(guessPW, NB)`
4. If `h(guessPW, NB) = h(pw(A,idp), NB)` → password recovered

---

### Why Secure Channels Prevent Guessing

| Protocol variant | Intruder sees | Guessing possible? |
|------------------|---------------|-------------------|
| `A -> idp: h(pw, NB)` | Hash + nonce | ✓ Yes (offline) |
| `[A] *->* idp: pw` | Nothing (TLS) | ✗ No |

**Pseudonymous channels** (`[A] *->* idp`) abstract TLS with server-side authentication:
- The password travels inside an encrypted TLS session
- The intruder sees no ciphertext containing the password
- No material available for offline guessing

---

### Week 6 Summary

**Completed:**
- [x] week6_v1.AnB — secure version with guessable password goal
- [x] week6_insecure.AnB — hash-based challenge-response demonstrating guessing (Special Task)
- [x] OFMC verification: secure version protects password, insecure triggers `guesswhat`

**Key findings:**
1. Hash-based challenge-response (`h(pw, NB)`) enables offline guessing — intruder verifies guesses
2. Pseudonymous channels prevent guessing — no password-derived material on the wire
3. pk-encryption of password does NOT enable guessing (intruder cannot decrypt)
4. Our final protocol protects passwords because A→idp uses `[A] *->* idp`

---

## Final Protocol

The final protocol (`anb/final/photo_auth_final.AnB`) is the Week 6 secure version
(`week6_v1.AnB`). No further changes were needed after Week 6 — the pseudonymous
channel design with format tags and guessable-password protection represents our
complete solution.

The key lookup subprotocol (`anb/final/key_lookup.AnB`) is analyzed separately and
uses format tag f5 to avoid confusion with the main protocol's f1–f4 tags.
