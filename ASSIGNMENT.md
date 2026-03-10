# 02244 Logic for Security - Project on Security Protocols

- **Hand-out:** Feb 9, 2026
- **Hand-in:** via DTU Learn until Mar 16, 2026 noon

## General Requirements

- Groups of up to 3 students allowed
- Each report must indicate which students are part of the group
- Reports must be divided into sections, each section must have **one group member designated as author**
  - Must reflect fair distribution in report writing
  - Any section without author marking will count as not submitted
- Report must indicate resources used:
  - Text books, research papers
  - Information found on the web
  - **Use of any AI tools**
  - Detailed suggestions from teachers
  - Results of discussions/cooperation with other students
- **Page Limit: 15 pages**
- Submit: Report + AnB files + Lab logbook (AnB and logbook don't count toward page limit)
- **Presentation:** Mar 16 - groups present protocols designed

---

## Open Authorization Scenario

**Setting:**
- **A** = User with online resources (photos stored at server B)
- **B** = Photo storage server  
- **P** = Photo book printing service
- **IdP** = Identity Provider (like MitID)

**Goal:** A wants P to access A's photos at B (without downloading and re-uploading)

**Constraints:**
- A does NOT want P to access other resources (e.g., email)
- Only read-access to photos (maybe specific folder)
- **A does NOT have a public/private key pair**
- A has a **password** (shared secret with IdP) - NOT usable as encryption key
- Everyone except A has public/private key pairs
- Everyone including A knows IdP's public key
- IdP knows all public keys
- **IdP is assumed honest** - all parties trust IdP
- Statements signed by IdP are accepted as truth

---

## Development & Lab Logbook

**Required:** Gradually develop protocol, document each step

For each version, document:
- What changed from previous version? What's the idea?
- Modeling considerations (assumptions, goals)
- Problems (attacks found, modeling limitations, OFMC runtime issues)
- Discussions/thoughts of group members, teacher input

---

## Weekly Tasks

### Week 2: Dolev and Yao
- Make informal outline (natural language/diagrams) without OFMC constraints
- Try first AnB version with simplifications (e.g., A has public keys)
- Model data access simply

**Special Task:** Static analysis - what can passive intruder learn (only observing, not sending)?

---

### Week 3: Lazy Intruder
- Make protocol more comprehensive
- A must NOT have secure key initially - rely on IdP
- A has shared password with IdP (NOT for encryption)
- May assume A initially knows all public keys

**Special Task:** Use lazy intruder technique to show role P is executable
- Instantiate P with intruder, others with honest agents
- Show intruder can play P (send every outgoing message when knowing incoming)
- Follow exactly the lazy intruder method from lecture

---

### Week 4: Typing and Implementation
- Use formats to replace all concatenations
- Make protocol **type-flaw resistant**
- Remove requirement that A knows all public keys initially
- Create **separate protocol** for A to ask IdP about public keys

**Special Task:** Show protocols are type-flaw resistant
- For any two messages in SMP (not variables), if they have unifier, they have same type

---

### Week 5: Channels
**Special Task:** Use TLS channels (server-side authenticated only)
- Use pseudonymous channel notation in OFMC
- Replace as much cryptography as possible with pseudonymous channels

---

### Week 6: Privacy
- Catch up and write report

**Special Task:** Verify protocol is secure even if password is **guessable secret**

---

### Week 7: Abstraction
- **Present solution in class**

---

## Hand-in Checklist

### Report (max 15 pages, individualized sections):
- [ ] Description of protocol development (including OFMC attacks encountered)
- [ ] Description of final AnB protocol (initial knowledge, goals explained)
- [ ] Assumptions/simplifications made, remaining problems
- [ ] Answer to special task for EACH week:
  - [ ] Week 2: Static analysis
  - [ ] Week 3: Lazy intruder execution
  - [ ] Week 4: Type-flaw resistance proof
  - [ ] Week 5: TLS channel variant
  - [ ] Week 6: Guessable password verification

### Attachments:
- [ ] Lab Logbook
- [ ] AnB file(s)
