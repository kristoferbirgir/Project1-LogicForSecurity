# Protocol Development Logbook

## Project Overview

**Scenario:** OAuth-like photo authorization
- **A** = Alice (user who owns photos)
- **B** = Photo storage server
- **P** = Photo printing service  
- **IdP** = Identity Provider (like MitID)

**Goal:** A wants P to access photos stored at B, BUT:
- P must NOT see A's password
- P must ONLY get access to photos (not everything)
- B must trust authorization from IdP

**Security Goals:**
1. Attacker cannot read photos
2. Attacker cannot impersonate A
3. Attacker cannot reuse old tokens (replay attack)
4. Password must not leak
5. P cannot access more than authorized

---

## Week 2 – Barebones Protocol

### Version: week2_v1.AnB

**Message Flow:**
```
1. A → IdP  : A, P, B                           (request authorization)
2. IdP → A  : {A, P, B, token(A,P,B)}_inv(pk(IdP))  (signed token)
3. A → P    : {A, P, B, token(A,P,B)}_inv(pk(IdP))  (forward token)
4. P → B    : {A, P, B, token(A,P,B)}_inv(pk(IdP))  (present token)
5. B → P    : {photos}_pk(P)                    (send photos encrypted)
```

**Idea:**
Initial simple protocol where A authorizes P to access photos from B.
IdP signs a token that grants P access to A's photos at B.

**Modeling considerations:**
- Public keys assumed known by all parties
- Token is signed by IdP (not encrypted)
- Photos encrypted to P

**Known Vulnerabilities (intentional):**
- No nonces → potential replay attacks
- No freshness guarantee → old tokens could be reused
- No binding between session and token

**OFMC Result:**
[x] Run completed - **ATTACK FOUND**

**Attack Description:**
```
ATTACK TRACE:
i -> (B,1): {A,P,B}_inv(pk(i))      # Intruder sends forged token with OWN signature
(B,1) -> i: {photos(A)}_(pk(P))     # B sends photos encrypted to P (but intruder intercepts)
```

**Problems Found:**
1. **Signature verification flaw**: B accepts ANY signed token, not specifically IdP-signed
2. The intruder can create `{A,P,B}_inv(pk(i))` using their own key
3. B doesn't verify the signer is actually IdP

**Next Steps for Week 3:**
- B must verify token is signed specifically by IdP
- Add nonces for freshness
- Consider encrypting the token so only intended recipient can use it

---

## Week 3 – Adding Freshness

### Version: week3_v1.AnB

**Idea:**
Add nonces to prevent replay attacks.

**Changes:**
(to be implemented after Week 2 analysis)

**Problems:**
(to be filled after OFMC run)

---

## Week 4 – Refined Protocol

### Version: week4_v1.AnB

**Idea:**
Further refinements based on Week 3 attacks.

**Changes:**
(to be implemented)

**Problems:**
(to be filled after OFMC run)