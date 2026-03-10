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
- Password is NOT used as encryption key
- A encrypts authentication request to IdP using pk(IdP)

**AnB Protocol:**
```
Protocol: PhotoAuthorization_v2

Types:
  Agent A, B, P, IdP;
  Number Na;
  Function pk, photos, pwd;

Knowledge:
  A: A, B, P, IdP, pk(B), pk(P), pk(IdP), pwd(A), photos(A);
  B: A, B, P, IdP, pk(B), pk(P), pk(IdP), inv(pk(B)), photos(A);
  P: A, B, P, IdP, pk(B), pk(P), pk(IdP), inv(pk(P));
  IdP: A, B, P, IdP, pk(B), pk(P), pk(IdP), inv(pk(IdP)), pwd(A);

Actions:
  A -> IdP: {A, P, B, pwd(A)}(pk(IdP))
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
1. A → IdP  : {A, P, B, pwd(A)}_(pk(IdP))   (encrypted auth request)
2. IdP → A  : {A, P, B}_inv(pk(IdP))        (signed token)
3. A → P    : {A, P, B}_inv(pk(IdP))        (forward token)
4. P → B    : {A, P, B}_inv(pk(IdP))        (present token)
5. B → P    : {photos(A)}_pk(P)             (encrypted photos)
```

**Modeling considerations:**
- `pwd(A)` models A's password as a function (password of A)
- Password sent encrypted to IdP (confidential but not used as key)
- IdP is assumed honest (assignment constraint)

---

### OFMC Verification

**Command:** `ofmc anb/week3_v1.AnB`

**Result:** ATTACK FOUND (same as Week 2)

**Attack trace:**
```
i -> (B,1): {A,P,B}_inv(pk(i))
(B,1) -> i: {photos(A)}_(pk(P))

secret leaked: photos(A)
```

**Analysis:**
The signature forgery attack from Week 2 still exists. Adding password authentication between A and IdP does NOT fix the core vulnerability - B still accepts any signed token.

**Key observation:**
- Password authentication: ✅ A proves identity to IdP
- Signature verification: ❌ B still cannot verify signer is IdP

**Note on password secrecy:**
If we add `pwd(A) secret between A, IdP` as a goal, OFMC finds an attack where intruder plays IdP. This violates the "IdP is honest" assumption from the assignment, so we exclude this attack from our threat model.

---

### Week 3 Special Task: Lazy Intruder Execution

**Goal:** Show that role P is executable by the intruder using the lazy intruder technique.

**Setup:**
- P = intruder (i)
- A, B, IdP = honest agents

**P's role in the protocol:**
```
Input  (step 3): Receive {A, P, B}(inv(pk(IdP))) from A
Output (step 4): Send {A, P, B}(inv(pk(IdP))) to B
Input  (step 5): Receive {photos(A)}(pk(P)) from B
```

**Lazy Intruder Analysis:**

**Step 0: Initial Intruder Knowledge (IK₀)**
```
IK₀ = {A, B, i, IdP, pk(A), pk(B), pk(i), pk(IdP), inv(pk(i))}
```
(All agent names, public keys, and intruder's private key)

**Step 1-2: Honest execution between A and IdP**
```
A → IdP: {A, i, B, pwd(A)}_(pk(IdP))
IdP → A: {A, i, B}_inv(pk(IdP))
```
(Intruder observes but these are encrypted/signed - cannot extract pwd(A))

**Step 3: A sends token to P (intruder)**
```
A → i: {A, i, B}_inv(pk(IdP))
```
Intruder knowledge after step 3:
```
IK₃ = IK₀ ∪ {{A, i, B}_inv(pk(IdP))}
```

**Step 4: Intruder must send token to B**

*Constraint:* Intruder must produce `{A, i, B}_inv(pk(IdP))`

*Lazy Intruder Derivation:*
```
Required: {A, i, B}_inv(pk(IdP))
From IK₃: {A, i, B}_inv(pk(IdP)) ∈ IK₃  ✓
```
**Constraint satisfied** - intruder directly has the required message.

**Step 5: B sends photos to P (intruder)**
```
B → i: {photos(A)}_(pk(i))
```
Intruder knowledge after step 5:
```
IK₅ = IK₃ ∪ {{photos(A)}_(pk(i))}
```

*Decryption:*
```
From {photos(A)}_(pk(i)) and inv(pk(i)) ∈ IK₀:
Derive: photos(A)  ✓
```

**Conclusion:**
**Role P is executable by the intruder.**

The intruder can successfully play role P because:
1. The token received in step 3 can be forwarded directly in step 4 (no transformation needed)
2. Photos received in step 5 are encrypted with pk(i), which intruder can decrypt

This demonstrates that any party who receives the authorization token can present it to B - the protocol does not bind the token to a specific recipient.

---

### Week 3 Summary

**Completed:**
- [x] Protocol with password authentication (week3_v1.AnB)
- [x] A has no key pair, uses password with IdP
- [x] OFMC verification (signature attack still present)
- [x] Lazy intruder special task (P role is executable)

**Key findings:**
1. Password authentication added but doesn't fix signature forgery
2. Role P is trivially executable - just forward the token
3. Token is not bound to specific recipient P

**Problems remaining:**
- Signature forgery attack (B accepts any signature)
- No freshness/nonces
- Token not bound to P

**Next steps for Week 4:**
- Add formats for type-flaw resistance
- Create separate key lookup protocol
- Address signature verification issue

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
