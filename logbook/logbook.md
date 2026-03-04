# Protocol Development Logbook

## Week 2 – Barebones Protocol

### Version: week2_v1.anb

**Idea**
Initial simple protocol where A authorizes P to access photos from B.

**Changes**
First protocol attempt.

**Modeling considerations**
- Simplified model
- Public keys assumed known

**Problems**
Not yet verified in OFMC.

---

## Week 3 – Identity Provider Authentication

### Version: week3_v1.anb

**Idea**
Introduce identity provider for authentication.

**Changes**
A authenticates via IdP instead of having keys.

**Problems**
Need to test lazy intruder execution.