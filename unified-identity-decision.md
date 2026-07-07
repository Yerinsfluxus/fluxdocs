# Unified Identity — Decision Doc

**Question:** should Riverly's Personal and SME apps run on one unified identity system (as they do today), or on fully separate accounts (like WEMA's ALAT and ALAT Business)?

**Verdict:** unified, with the per-product credential separation we just shipped. This document lays out the case honestly — trade-offs included — so the team can decide with full information.

---

## What "unified identity" means, concretely

A single `identities` row per human. That identity owns one or more `productMemberships` (Personal, SME, Corporate). Each product membership has:

- Its own credential (`ProductMembership.PasswordHash`) — Personal is a 6-digit PIN, SME is an 8+ char complex password. Formats can't collide.
- Its own contact (`ProductMembership.PhoneNumber` + `Email`) — different business phone from Personal is supported.
- Its own product-scoped JWT after the caller proves the product's credential.

Login goes through `/identity/login` with a `productType` parameter. Switching between products (e.g. after login) goes through `/identity/switch-product`, which requires the target product's credential on every cross-product move.

The alternative under discussion is **fully separate accounts** — two completely disconnected systems, one for Personal, one for SME. Different apps, different databases (or at least different root tables), no shared identity concept. This is what WEMA does.

---

## The four questions that matter

### 1. Is unified as secure as fully separate?

**Yes, after this week's work.** The specific attack that unified used to make easier — leak Personal PIN → attacker reaches SME because they shared a credential — is fully closed by three changes we shipped:

1. Personal and SME hashes are now on separate rows (`ProductMembership.PasswordHash`). A Personal PIN reset physically cannot overwrite SME's password.
2. `/identity/login` with `productType: Sme` verifies against the SME membership hash only. A leaked Personal PIN → SME login returns "Invalid credentials." Confirmed on staging (test 1e in the validation matrix).
3. `/identity/switch-product` requires the target product's credential on every cross-product move. A leaked Personal PIN → Personal login → try to switch to SME → server refuses without the SME password. Confirmed on staging (test 3b + 3c).

Same in reverse for a leaked SME password.

Compare to fully separate:
- Fully separate makes the leak-Personal-then-reach-SME attack impossible by construction (different systems, no shared token). Same real-world outcome as our current setup.
- Fully separate does NOT make phishing, SIM swap, credential stuffing, or session hijacking any harder. Those are the actual attack vectors banks lose money to. Neither model protects against them any better than the other.

**Net:** on the security dimension that matters — protecting a wallet when one credential leaks — unified with per-product credentials matches fully separate. On the security dimensions that matter more (fraud correlation, sanctions monitoring, fast lock-down on a compromised human), unified is better. See #3.

### 2. What does unified give us that fully separate can't?

**a. One human, one BVN check.** Nigerian regulation identifies individuals by BVN. Femi as an individual has one BVN. If Femi's Personal account was verified by BVN and later he opens SME (as the SME's owner-director), we already know he's a verified real person. Fully separate would ask him to re-verify.

Note: SME KYC is business-scoped (KYB via CAC) not individual-scoped (BVN). The individual-BVN dedup only applies to the *owner* linkage, not the business itself. This is a smaller win than "one KYC per person" — but it's a real one, because the owner is always a person with a BVN.

**b. Fraud correlation across an individual's products.** If Femi's Personal shows suspicious activity (unusual login geo, structuring, mule pattern), we can proactively flag any SME he owns from the same dashboard. Fully separate treats them as unrelated humans — investigators would have to manually correlate.

**c. Easier compliance dashboard.** Regulators (CBN, NFIU) increasingly ask for a customer-360 view. Unified gives us that natively; separate accounts force a manual reconciliation step every reporting cycle.

**d. Product-add without full re-onboarding.** A Personal user who wants to add SME just needs KYB on the business — Riverly already knows who they are as a person. Separate accounts would require full identity re-verification.

**e. Future features that need a shared identity.** Cross-product transfers (Personal → SME wallet), unified statements, single customer support view. Separate accounts block all of these.

None of a–e come "for free" on the separate model — they'd need custom middleware that ends up rebuilding a unified identity anyway.

### 3. What does fully separate give us that unified can't?

Being fair to the other side:

**a. Simpler code paths.** Two systems, no shared identity, no dual-table lookup, no per-product credential resolution. Less surface area for the specific class of bugs the unified system had (which we spent this week fixing).

**b. Perceived clean separation for enterprise SME customers.** Some business owners *feel* safer knowing their business is on a different system. This is UX perception, not real security — but perception matters.

**c. Independent product velocity.** SME can ship an auth change without a per-product-credential-plan-level design doc. Unified requires more discipline.

**d. Blast radius on a bug.** A bug in unified auth affects both products. A bug in Personal's separate auth doesn't touch SME's separate auth.

None of these are safety concerns — they're engineering complexity trade-offs. Complexity we chose to take on because a–e in section 2 aren't achievable the separate way.

### 4. Have we introduced complexity we can't maintain?

**Honest answer: unified requires discipline we didn't have before.** The initial implementation had the shared-credential bug precisely because we treated identity as the credential source without asking whether products should share it. That mistake is now paid for and won't recur — the plan doc and code enforce per-membership credentials as an invariant.

Going forward, unified takes more BE thought per auth-touching feature. In exchange for that, we get the compliance and correlation benefits in section 2. That's a fair trade for a bank.

If we were to reverse this decision now, we'd:
- Duplicate every user's identity row into two disconnected systems
- Lose the BVN/owner linkage between a person and their SME
- Break every existing user's login (they signed up on unified)
- Rebuild switch-product, cross-product KYC checks, and fraud correlation from scratch — as separate-account middleware

The migration cost is significantly higher than the ongoing discipline cost.

---

## What's actually working right now (verified on staging)

Full validation matrix — 16 scenarios covering login, switch, change, forgot, legacy `/auth/*` mirror, and Femi-shape edge case — all pass. Highlights:

- Personal PIN typed at SME login → refused. Same reverse. Credential separation is real.
- Cross-product switch without target password → structured `stepUpRequired` refusal, no token.
- Cross-product switch with correct password → succeeds. With wrong password → `Invalid password.`
- Rate limit fires at attempt 11/min → 429.
- Product-scope token used against `/switch-product` → 403.
- Legacy `/auth/login` still works with the same PIN — the two-way credential mirror is in sync.
- Femi's account (multi-product, membership hashes not yet backfilled) correctly returns `credentialSetupRequired: true, nextStep: "ForgotPasscode"` on new-client Personal login. This is the one-time migration flow — every existing multi-product user runs forgot-passcode once per product on Tolu's new build, sets their own PIN/password, and they're on the per-product model permanently.

---

## Known design choices (not gaps — flag these to the team so they're conscious calls, not surprises)

1. **Refresh token drops the `authenticated_product` claim.** After a refresh, the next `/switch-product` call asks for the password again. Defensible — the claim represents a fresh credential proof, and a refresh isn't one — but it's a UX call, not a security requirement. Can be revisited.

2. **`/passcode/set` doesn't mint the claim.** A user who just set their Personal PIN and then calls `/switch-product` Personal will be asked to type the PIN they just set. Minor friction. Fixable if we want.

3. **`SwitchProduct` rate limit is identity-wide, not per-target-product.** An SME wrong-password spree burns the same identity's rate budget as Personal switches from that identity. Not a security concern; would be cleaner as per-target-product buckets in a follow-up.

4. **Corporate is deliberately out of v1.** Corporate keeps its own `/corporate/*` login. A future PR migrates it in properly.

5. **No automated test project yet.** All verification is manual curl against staging + DB assertions. Standing up an xUnit + Testcontainers-postgres project is a follow-up worth doing; not blocking this release.

6. **Legacy `/auth/*` endpoints are still live** with a two-way credential mirror. Necessary during rollout so users on the old app build keep working. Once every mobile install is on the new build, we can drop `/auth/*` in a follow-up PR.

None of these are open security holes.

---

## Recommendation

Ship the unified identity + per-product credentials + switch-product step-up model to production. It matches the modern fintech industry direction, meets Nigerian regulatory requirements, closes the specific bug that made unified less safe than separate in v0, and gives us the compliance and cross-product features that a Nigerian banking app needs to scale.

The trade-off — more BE discipline per auth feature — is worth it and is now codified in the two plan docs (`per-product-credentials-plan.md`, `switch-product-step-up-plan.md`) so future changes don't recreate the v0 mistake.

If the team disagrees, the concrete question to answer is: which of the five benefits in section 2 can we live without? Because those are the things fully separate can't give us.
