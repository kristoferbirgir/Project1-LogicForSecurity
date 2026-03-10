# Protocol Development Logbook

## Project Overview

**Scenario:** Open Authorization (OAuth-like)
- **A** = Alice (user who owns photos)
- **B** = Photo storage server
- **P** = Photo printing service
- **IdP** = Identity Provider (like MitID)

**Goal:** A wants P to access photos stored at B without:
- Giving P access to other resources (e.g., email)
- Sharing A's password with P
- Downloading and re-uploading photos

**Assignment Constraints:**
- A does NOT have a public/private key pair (simplified in Week 2)
- A has a password shared with IdP (NOT for encryption)
- Everyone except A has public/private key pairs
- Everyone knows IdP's public key
- IdP is honest and trusted by all parties

---

## Week 2 – Dolev-Yao: Barebones Protocol

### Informal Design (Before AnB)

**High-level idea:** Similar to "Login with Google" - A authorizes P through a trusted identity provider.

**Protocol sketch in natural language:**
```
1. A tells IdP: "I want to authorize P to access my photos at B"
2. IdP creates a signed authorization token
3. A forwards this token to P
4. P presents the token to B
5. B verifies the token and gives photos to P
```

**Design diagram:**
```
    A                   IdP                  P                   B
    |                    |                   |                   |
    |-- auth request --->|                   |                   |
    |<-- signed token ---|                   |                   |
    |------- forward token ---------------->|                   |
    |                    |                   |-- present token ->|
    |                    |                   |<---- photos ------|
```

**Key design questions:**
- How does B verify the token? → Check IdP's signature
- How are photos protected in transit? → Encrypt to P
- What prevents token reuse? → (Not addressed in v1 - needs nonces)

---

### Version: week2_v1.AnB

**Simplifications made (allowed in Week 2):**
- A has public/private key pair (will be removed in Week 3)
- All parties know all public keys
- No password authentication yet

**AnB Protocol:**
```
Protocol: PhotoAuthorization_v1

Types:
  Agent A, B, P, IdP;
  Number Na;
  Function pk, photos;

Knowledge:
  A: A, B, P, IdP, pk(A), pk(B), pk(P), pk(IdP), inv(pk(A)), photos(A);
  B: A, B, P, IdP, pk(A), pk(B), pk(P), pk(IdP), inv(pk(B)), photos(A);
  P: A, B, P, IdP, pk(A), pk(B), pk(P), pk(IdP), inv(pk(P));
  IdP: A, B, P, IdP, pk(A), pk(B), pk(P), pk(IdP), inv(pk(IdP));

Actions:
  A -> IdP: A, P, B
  IdP -> A: {A, P, B}(inv(pk(IdP)))
  A -> P: {A, P, B}(inv(pk(IdP)))
  P -> B: {A, P, B}(inv(pk(IdP)))
  B -> P: {photos(A)}(pk(P))

Goals:
  B authenticates IdP on A, P, B
  photos(A) secret between A, B, P
```

**Message flow:**
```
1. A → IdP  : A, P, B                    (plaintext request)
2. IdP → A  : {A, P, B}_inv(pk(IdP))     (signed token)
3. A → P    : {A, P, B}_inv(pk(IdP))     (forward token)
4. P → B    : {A, P, B}_inv(pk(IdP))     (present token)
5. B → P    : {photos(A)}_pk(P)          (encrypted photos)
```

**Modeling considerations:**
- `photos(A)` models "A's photos" as a function application
- Token is signed by IdP (provides authenticity)
- Photos encrypted to P (provides confidentiality)
- Authentication goal: B should verify IdP signed the token

---

### Week 2 Special Task: Static Analysis (Passive Intruder)

**Question:** If the intruder ONLY observes network traffic (does not inject or modify messages), what can they learn according to the Dolev-Yao model?

**Messages observable on the network:**
```
Message 1: A, P, B                         (plaintext)
Message 2: {A, P, B}_inv(pk(IdP))          (signed, readable)
Message 3: {A, P, B}_inv(pk(IdP))          (same token)
Message 4: {A, P, B}_inv(pk(IdP))          (same token)
Message 5: {photos(A)}_pk(P)               (encrypted)
```

**Dolev-Yao analysis - What passive intruder can derive:**

| Data | Learnable? | Reasoning |
|------|------------|-----------|
| A (user identity) | ✅ Yes | Sent in plaintext in message 1 |
| P (service identity) | ✅ Yes | Sent in plaintext in message 1 |
| B (server identity) | ✅ Yes | Sent in plaintext in message 1 |
| Authorization token content | ✅ Yes | Signature does not encrypt - content {A,P,B} is readable |
| IdP's signature | ✅ Yes | Can see the signature (but cannot forge without inv(pk(IdP))) |
| photos(A) | ❌ No | Encrypted with pk(P), intruder lacks inv(pk(P)) |

**Derivation rules applied:**
- Projection: From (A, P, B), derive A, P, B individually
- Signature opening: From {M}_inv(pk(IdP)) and pk(IdP), can read M (but not forge)
- Decryption blocked: {photos(A)}_pk(P) requires inv(pk(P)) which intruder doesn't have

**Conclusion:**
- **What passive intruder learns:** User A is authorizing service P to access server B
- **What remains secret:** The actual photo content
- **Privacy concern:** Metadata leak - the fact that A uses service P is revealed to anyone observing the network

---

### OFMC Verification (Active Intruder)

**Command:** `ofmc anb/week2_v1.AnB`

**Result:** ATTACK FOUND

**Attack trace:**
```
i -> (B,1): {A,P,B}_inv(pk(i))      
(B,1) -> i: {photos(A)}_(pk(P))    

secret leaked: photos(A)
```

**Attack explanation:**
1. Intruder creates a forged token `{A,P,B}_inv(pk(i))` signed with their own key
2. Intruder sends this to B
3. B accepts the token (doesn't verify signer is IdP)
4. B sends photos encrypted to P
5. Intruder obtains the encrypted photos

**Root cause:** 
B accepts ANY valid signature, not specifically one from IdP. In the Dolev-Yao model, the intruder has their own key pair and can sign anything.

**Problems identified:**
1. No mechanism for B to verify the signer is specifically IdP
2. No nonces - vulnerable to replay attacks
3. No freshness guarantees

---

### Week 2 Summary

**Completed:**
- [x] Informal protocol design (natural language + diagram)
- [x] First AnB protocol (week2_v1.AnB)
- [x] Static analysis (passive intruder) - Special Task
- [x] OFMC verification showing attack

**Key findings:**
- Passive intruder: Can learn metadata (who authorizes whom) but not photos
- Active intruder: Can forge tokens and obtain photos

**Next steps for Week 3:**
- Remove A's key pair (use password authentication with IdP)
- Fix signature verification so B only accepts IdP signatures
- Add nonces for freshness

---

## Week 3 – Lazy Intruder

### Version: week3_v1.AnB

**Changes from Week 2:**
- A no longer has public/private key pair
- A authenticates to IdP using password (shared secret)
- Password NOT used as encryption key

**TODO:**
- [ ] Implement password-based authentication
- [ ] Special Task: Lazy intruder execution (show P role is executable)

---

## Week 4 – Typing and Key Lookup

**TODO:**
- [ ] Use formats for type-flaw resistance
- [ ] Create separate key lookup protocol
- [ ] Special Task: Show protocols are type-flaw resistant

---

## Week 5 – Channels

**TODO:**
- [ ] Replace cryptography with pseudonymous channels (TLS)
- [ ] Special Task: TLS channel variant

---

## Week 6 – Privacy

**TODO:**
- [ ] Verify protocol secure with guessable password
- [ ] Special Task: Guessable password verification
