# SecretPass — Implementation Plan

**Version:** 1.2 (Draft)
**Companion to:** SecretPass Requirements Document v1.3
**Last updated:** 14 August 2026

> **On estimates:** this plan carries no effort or duration estimates, by decision. The build is intended to move fast, and phase-level week counts would only invite planning against numbers nobody intends to hold to. What the plan does carry is **ordering**, which matters more — see Section 5. The only durations recorded anywhere are other people's clocks (D-U-N-S issuance, penetration-test scheduling, store review, Google Play's 14-day testing rule), because those are not compressible by working faster.

---

## 1. Tech Stack Summary

| Layer | Choice | Notes |
|---|---|---|
| Client apps (Web, Android; **iOS post-v1**) | **Flutter** | Single codebase. Web is built first, Android second (Requirements 2) |
| **Android autofill service** | **Native Kotlin** (`AutofillService`) | Flutter cannot provide this — the OS instantiates a native Android component outside the Flutter engine. Platform work, not a Flutter screen (Requirements 4.3.2) |
| Browser extension | **JS/TS, Manifest V3** | Separate small codebase — Flutter can't produce a browser extension |
| **Marketing site** | **Static HTML/CSS** (any static generator) | Deliberately *not* the Flutter web build. Canvas rendering is unindexable, and the SEO channel in Requirements 6.1 depends on this being ordinary HTML. Also keeps Terms, Privacy Policy, contact, activation and resend pages reachable without loading the app |
| Backend / sync | **Supabase** (Postgres + Auth + Storage) | Low starting cost, self-hostable later; server only ever stores ciphertext |
| Local on-device storage | **SQLite** (`sqflite` / `drift`) | Stores the same one-blob-per-item ciphertext the server holds (see Section 3) — never field-level plaintext columns. Search decrypts into memory and filters there; do not try to make encrypted items SQL-queryable |
| Cryptography | **libsodium** (`sodium_libs` in Flutter, `libsodium.js` in the extension) | One crypto library across both codebases avoids subtle cross-language incompatibilities |
| Key derivation | **Argon2id** (via libsodium) | Derives the vault encryption key from the master password |
| Subscription billing | **Paddle** (merchant of record) | Single processor, web checkout only. Handles global VAT/GST registration and filing so SecretPass doesn't have to |
| TOTP generation | **RFC 6238 implementation** (e.g., `otp` package in Dart, `otplib` in JS) | Standard algorithm, no custom crypto |
| CSV parsing | `csv` (Dart) | Handles quoting/escaping correctly — don't split on commas by hand |
| Admin console | **Flutter web** (separate build target) or a small React app | Separate deployment from the user-facing app; never bundled into the mobile builds |

---

## 2. Architecture

```
┌──────────────┐   ┌─────────────────┐   ┌─────────────────┐
│   Web App    │   │   Android App   │   │    Browser      │
│  (Flutter)   │   │    (Flutter)    │   │ Extension (JS)  │
│              │   │  + AutofillSvc  │   │                 │
│              │   │     (Kotlin)    │   │                 │
└──────┬───────┘   └────────┬────────┘   └────────┬────────┘
       │                    │                     │
· · · ·│· · · · · · · · · · │ · · · · · · · · · · │· · · · · · ·
       │    CLIENT SIDE — plaintext, vault key, all crypto
       │                    │                     │
       │    SERVER SIDE — ciphertext only, never the key
       └────────────────────┼─────────────────────┘
                            │
              ┌─────────────▼────────────┐
              │        Supabase          │
              │  (Auth + Postgres +      │
              │   encrypted blob storage)│
              └──────────────────────────┘

              (iOS app + ASCredentialProviderExtension: post-v1,
               joins the client row unchanged)
```

Everything above the dotted line encrypts and decrypts locally — the browser extension and the Android autofill service included. Supabase's job is limited to authenticating the user and storing/syncing opaque encrypted blobs; it never has the vault key and cannot derive it.

Note that the Android autofill service sits **inside** the client boundary but **outside** the Flutter engine: it is a native component the OS calls directly, and it is a second entry point into the vault. Requirements 4.3.3 governs how it behaves against a locked vault, and that rule is a security boundary, not a UX preference.

### 2.1 Zero-knowledge flow

**One password, two independent derivations.** There is a single master password (Requirements 4.5 — this reverses the two-password design in v1.1 of this plan). It never leaves the device. Two separate values are derived from it client-side, and only one of them is ever transmitted:

```
                    master password
                          │
          ┌───────────────┴───────────────┐
          │                               │
   Argon2id(pw, salt_vault)        Argon2id(pw, salt_auth)
          │                               │
     wrapping key                   auth value
          │                               │
   unwraps the vault key          sent to Supabase Auth
   (never leaves device)          (server hashes it again
                                   and stores that)
```

The two derivations **must use distinct salts and distinct context**, so that possession of the auth value gives no advantage in deriving the wrapping key. Get this wrong — same salt, or one derived from the other in a reversible way — and the server is holding material related to the vault key, which is the exact property the whole design exists to prevent.

Rules that follow, and which the cryptographic design document (Requirements 5.1.1) must state explicitly because a reviewer will look for them first:

- The raw master password is **never** transmitted, logged, persisted, or placed in a crash report. Only the auth value leaves the device.
- The auth value is a credential, not a key. Supabase applies its own password hashing to it on receipt; treat a leak of the auth-value database as equivalent to a leak of any password hash database — bad, but not a vault compromise.
- Deriving the auth value costs a full KDF run, so **derive it once per sign-in and cache the session token**, not the password. Re-deriving on every request would put the password back in memory repeatedly for no benefit.
- **There is no server-side password reset.** Nothing the server holds can regenerate the wrapping key, so a reset flow that restored account access without the recovery key would either be impossible or a backdoor. Do not build a "forgot password" email flow that appears to work; the only recovery is 2.2.

**Key hierarchy — two levels, not one.** The master password does *not* encrypt vault items. It derives a wrapping key, which encrypts the vault key, which encrypts the items. That indirection is what makes both the recovery key (2.2) and master-password changes possible. Getting this wrong — collapsing it to one level — is the single most expensive mistake available here, because it is only discovered when the recovery path is built and found not to work.

1. At activation, the client generates a **random vault key** (`crypto_secretbox_keygen`). This key, and only this key, ever encrypts vault data.
2. The user sets a master password. Argon2id derives a **wrapping key** from it, using a per-user random salt stored server-side — the salt is not secret, only the derived key is. A second Argon2id run over the same password with a *different* salt produces the auth value described above.
3. The vault key is encrypted ("wrapped") under that wrapping key. The wrapped copy is stored server-side; the vault key itself is never stored anywhere in the clear.
4. Vault items are encrypted locally with the **vault key** using libsodium's `crypto_secretbox` (XChaCha20-Poly1305), one ciphertext blob per item. Only ciphertext is ever sent to Supabase.
5. On unlock, the client re-derives the wrapping key from the master password + salt, unwraps the vault key, and holds it in memory only while the vault is unlocked (see Requirements 4.7).
6. On a new device: same master password, same salt, same derivation — unwraps the same vault key, which decrypts the synced blobs.

Two consequences worth stating, because they are the reason for the indirection:
- **Changing the master password re-wraps one key.** Derive a new wrapping key, re-encrypt the vault key under it, upload. Vault items are untouched — no bulk re-encryption, no risk of a partial rewrite corrupting a vault. Under the single-password model a master-password change must **also** re-derive the auth value and update the Supabase credential, and the two updates have to succeed or fail together — a vault re-wrap that lands while the auth update fails leaves an account that cannot sign in to reach its own newly-wrapped vault. Sequence it so the recoverable failure is the one that happens.
- **The recovery key is a second, independent wrap of the same vault key** (2.2), which is only possible because the vault key is a distinct object rather than a direct output of the master password.

### 2.2 Recovery key (the only backup path)

The vault's actual encryption key is generated once at signup and then wrapped (encrypted) twice, independently:

1. With a key derived from the master password — the daily unlock path.
2. With the **recovery key** — a high-entropy secret shown to the user once at setup, to be stored offline.

Both wrapped copies live server-side as ciphertext; the server can open neither. Losing the master password means unwrapping copy 2 with the recovery key, then re-wrapping under a new master password.

Losing both means the vault is gone. There is no third wrap, no escrowed copy, and no admin override — see Requirements 4.9 for why the admin console must not be able to provide one.

Trusted-contact emergency access was considered and dropped: it would have required per-user keypairs, an invitation flow, time-delayed approvals, and notification infrastructure — a significant amount of security-critical surface for a v1.

---

## 3. Data Model (Supabase / Postgres)

| Table | Key columns | Notes |
|---|---|---|
| `users` | `id`, `email`, `kdf_salt_vault`, `kdf_salt_auth`, `created_at`, `is_admin`, `entitlement_override`, `refunds_issued` | Managed largely by Supabase Auth. **Two salts, not one** — the wrapping-key and auth-value derivations must not share one (2.1). Neither is secret. `is_admin` grants console access only — it must never appear in any RLS policy governing vault reads. `entitlement_override` marks the admin's and the store review account permanently entitled without a `subscriptions` row, and also excludes them from the dormancy deletion job. `refunds_issued` enforces the one-automatic-refund-per-customer rule at the database level |
| `wrapped_keys` | `id`, `user_id`, `wrap_type`, `wrapped_vault_key`, `nonce`, `updated_at` | The two independent wraps of the vault key (2.1, 2.2): `wrap_type` is `master_password` or `recovery_key`. Both are ciphertext the server cannot open. A master-password change rewrites one row here and touches no vault items |
| `vault_items` | `id`, `user_id`, `ciphertext`, `nonce`, `updated_at`, `deleted_at` | `ciphertext` holds the entire encrypted item (title, username, password, notes, TOTP secret) as one encrypted JSON blob — server never sees field-level structure, and neither does local SQLite |
| `folders` | `id`, `user_id`, `ciphertext`, `nonce` | Folder/tag names are also encrypted, not just item contents |
| `subscriptions` | `id`, `user_id`, `paddle_subscription_id`, `status`, `created_at`, `renews_at`, `grace_ends_at`, `lapsed_at` | Mirrors Paddle state; `status` drives the active/past_due/grace/lapsed/suspended machine. `created_at` is what the 30-day refund window is measured from |
| `activation_tokens` | `id`, `user_id`, `token_hash`, `expires_at`, `consumed_at` | Single-use, 7-day activation links. Store a hash, never the raw token |
| `tos_agreements` | `id`, `user_id`, `tos_version`, `agreed_at`, `consent_point` | Both consent captures per Requirements 4.11 — `consent_point` is `checkout` or `activation`. Which version was agreed to and when is unanswerable retroactively, so record it at the time |
| `admin_audit_log` | `id`, `admin_user_id`, `action`, `target_user_id`, `metadata`, `timestamp` | Append-only: grant `INSERT` and `SELECT` to the console role, never `UPDATE` or `DELETE`. Must never contain vault ciphertext or anything derived from it |
| `sync_log` | `id`, `user_id`, `item_id`, `op`, `timestamp` | Supports conflict resolution / last-write-wins with a changelog for debugging |

Row-Level Security (RLS) policies in Supabase ensure a user can only ever read/write rows where `user_id` matches their authenticated session.

---

## 4. Build Phases

### Phase 0 — Foundations

- **Start the organization registration on day one.** It is the longest-lead item in the whole plan and nothing else in the build depends on it, which is precisely why it gets forgotten until it is the only thing standing between a finished app and a store listing. The chain is: form the legal entity → apply for the D-U-N-S number (issued in one to two weeks, and not compressible) → complete Google Play organization verification. Registering as an organization rather than an individual also exempts the Play account from the closed-testing gate described in Section 6.3. See that section for the full reasoning.
- Repo setup, CI/CD skeleton, Flutter project scaffolding for **Web and Android** targets.
- **Turn on the automated security tooling now** (see Phase 5a). Retrofitting secret scanning after a key has already been committed to history is a much worse day.
- Supabase project setup: Auth, Postgres schema, RLS policies.
- Integrate libsodium in Flutter; validate Argon2id + XChaCha20-Poly1305 round-trip.
- **Validate the full two-level key hierarchy (2.1) before building anything on top of it:** generate a vault key, wrap it under a master-password-derived key *and* under a recovery key, unwrap via both paths, and confirm a master-password change re-wraps without touching item ciphertext. Cheap now, a rewrite later.
- **Validate the single-password split derivation in the same session** (2.1): confirm the wrapping key and the auth value are independently derived under distinct salts, that the auth value round-trips through Supabase Auth as a credential, and that a master-password change updates both the wrap and the auth credential atomically.
- **Decide the autofill/auto-lock resolution now** (Requirements 4.3.3), before either the lock model or the autofill service exists. Phase 2.4 and Phase 2.6 both depend on the answer, and discovering the conflict once both are built means rewriting one of them.
- Measure Argon2id on real mid-range hardware and record the parameters and timings — Requirements 5.2 budgets unlock against measured numbers, not guesses. Note that the single-password model runs the KDF twice at sign-in (wrapping key and auth value); measure the combined cost, not one derivation.

### Phase 1 — Core vault MVP
- Master password setup + unlock flow (single password, both derivations — 2.1).
- Local encrypted SQLite store.
- Vault item CRUD (create/read/update/delete), search, folders/tags.
- Sync engine: push/pull encrypted blobs to/from Supabase, basic last-write-wins conflict resolution.
- Ship on Web first — fastest iteration loop, and no store review standing between a build and a user — then Android.

### Phase 2 — Generator & TOTP
- Password generator screen + inline generator (when creating/editing an item).
- TOTP secret storage (manual entry + QR scan) and live code display.

### Phase 2.4 — Biometric unlock
- Use `local_auth` for the platform prompt, and store the wrapped vault key in the **Android Keystore** with biometric-gated access — do not hand-roll key storage. Put this behind a platform-agnostic interface so the iOS Keychain / Secure Enclave implementation lands post-v1 as an implementation rather than a refactor.
- Set the key invalidation flag (`setInvalidatedByBiometricEnrollment`) so enrolling a new fingerprint or face on the device invalidates the stored key. This is one line of configuration and it's the difference between biometrics being a convenience and being a hole.
- Enforce master-password re-entry on reboot, after inactivity, after failed biometric attempts, and before export, recovery-key display, master-password change, or account deletion.
- **Auto-lock (Requirements 4.7):** 5-minute default, user-configurable 1–60 minutes, no "never". Lock immediately on app background, device lock, and restart regardless of the setting. Locking must zero the vault key and decrypted items in memory — a lock that only swaps the visible route leaves the plaintext vault sitting in a Dart object for anyone who can attach a debugger or trigger a heap dump. Test that explicitly; it is the kind of thing that silently regresses.

### Phase 2.5 — CSV import/export, Terms, account deletion
- CSV import with a column-mapping step; parse with a real CSV library, validate before committing, all-or-nothing write.
- CSV export gated behind master-password re-authentication, with an unmissable plaintext warning in the UI.
- Terms of Service screen in the app (Settings), and the **activation-time** consent modal and checkbox. The *checkout-time* consent lives on the marketing site and is built in Phase 4 — the two consent points ship in different phases, so neither should assume the other exists. Write both agreements to `tos_agreements` with the version and timestamp.
- Self-service account deletion (Requirements 4.13): master-password re-auth, an explicit irreversibility warning, a CSV export offered from the confirmation screen itself, and a Paddle subscription cancellation as part of the same flow. Required by both app stores, so it is not optional polish.

### Phase 2.6 — Android autofill service

Native Kotlin, outside the Flutter engine — the OS instantiates and calls this component itself (Requirements 4.3.2). Plan it as platform work. This is the phase that makes the Android app a password manager rather than a list users copy out of, and it has no Flutter analogue to lean on.

- Implement `AutofillService`: parse the incoming `AssistStructure`, identify credential fields, return datasets.
- **Digital Asset Links verification before any dataset is returned.** Resolve the requesting package's signing certificate and check it against the target domain's `assetlinks.json`. No verified association means no automatic fill — fall back to explicit per-app user confirmation and persist that decision. Never match on package name or app label; both are attacker-chosen.
- Wire the service to the lock model settled in Phase 0. Against a locked vault the fill request triggers a biometric or master-password prompt scoped to that single request, the key is not retained afterwards, and servicing a fill does not extend the inactivity timer (Requirements 4.3.3).
- Onboarding prompt to set SecretPass as the device autofill provider. It is off by default and users will not find that setting on their own — an autofill implementation nobody enables is indistinguishable from one that was never built.
- Save-new-credential capture from other apps.
- Keep vault lookup logic cleanly separated from the `AutofillService` plumbing so Credential Manager and passkeys are an addition later rather than a rewrite.
- **Test against a deliberately built look-alike app**, not by code inspection. This is the extension's look-alike-domain bug class in mobile form, and it is the most severe thing this phase can ship wrong.

### Phase 3 — Browser extension, Chrome only
- Manifest V3 extension: auth against the same Supabase project.
- Chrome only at launch. Keep the code free of Chrome-specific APIs where a standard equivalent exists, so Edge and Firefox are a port rather than a rewrite later.
- Form detection, autofill, save-new-credential prompt.
- Reuse the same libsodium.js-based crypto so the extension reads/writes the identical ciphertext format as the apps.

### Phase 4 — Subscription & entitlements

The largest phase after the vault core, and the one most easily mistaken for a small one. What follows is a payment integration, a five-state lifecycle, a provisioning path, a refund system with abuse controls, and six or so transactional emails — none of which can be half-built, because every one of them is a path where money and vault access meet. Expect a meaningful share of the work to be testing webhooks against Paddle's sandbox rather than writing code, and note that sandbox round-trips run on Paddle's timing rather than yours.

- Integrate Paddle checkout on the marketing website, including the **checkout-time** Terms consent checkbox that blocks progress to payment (Requirements 4.11).
- Paddle webhook → Supabase `subscriptions` table; entitlement is a single boolean per user.
- Apps check that boolean at sign-in and on sync. No billing SDK ships in the mobile or web app at all.
- Ensure the mobile builds contain no purchase flow — no subscribe button, no checkout link, and no pricing shown to anyone who isn't already subscribed (see Requirements 4.5). This is a Play review rejection risk now and an App Review rejection risk when iOS ships. Showing an existing subscriber their own plan, price, renewal date, and refund amount is account management and is permitted; the rule is about initiating or advertising a purchase, not about displaying numbers.
- **Provision the store review account** (Requirements 4.5). Purchase-first provisioning means a reviewer cannot create an account, and a submission whose sign-in screen is a dead end is rejected for reviewer inaccessibility — a wasted review cycle every time it happens. Create one permanently entitled account via `entitlement_override`, exclude it from the dormancy job alongside the admin's, and record its credentials in the Play Console's App access section. Re-verify it signs in from a clean device before every submission.
- **Purchase-first provisioning:** Paddle webhook creates the account row and issues a single-use activation token (7-day expiry). Provision on the webhook, never on the browser redirect.
- Activation page: consumes the token, then walks the user through master password → recovery key confirmation. Two secrets, not three (Requirements 4.5). The same page serves the bootstrap admin, so there is one activation code path in the product and the operator exercises it before any customer does.
- Public, unlimited resend-activation page keyed on checkout email — this doubles as the mistyped-email recovery path.
- Subscription state machine: `active` → `past_due` (Paddle dunning) → `grace` (7 days, still fully functional, reminders at day 1 and 5) → `lapsed`.
- Refund → `lapsed`. Chargeback → `suspended`, blocked from resubscribing pending review.
- Dormancy job: warn at 90/97/104 days lapsed, then delete the vault.
- Implement lapse behaviour per Requirements 4.5: vault stays readable locally and CSV export stays available; sync, extension, and new-item creation suspend. **Never** make a lapsed user's existing vault unreadable.
- **Self-service cancel + automatic refund.** In-app cancel calls your backend, which decides from `subscriptions.created_at` whether the purchase is inside the 30-day window, then calls Paddle's cancel and (if eligible) refund endpoints directly. No admin queue in the normal path.
- Make the refund call **idempotent** — key it on the subscription ID and store the result. A double-tapped button or a retried request must not issue two refunds.
- State transitions come from the Paddle webhook, not from the API response: mark the account lapsed when the refund event arrives, so a settled refund and the account state can't diverge.
- Failed refund calls: retry with backoff, then raise into the admin console's "needs attention" queue. This is the only path where a human touches a refund.
- Enforce one automatic refund per customer at the database level (`refunds_issued` counter on the user); a second request routes to manual review rather than auto-approving.
- Confirmation email on both cancellation and refund, stating the amount and what happens to their vault.

### Phase 4.4 — Contact page
- Public form on the marketing site; no authentication required.
- Server-side handler holds the destination address in an environment variable — never in client code, page source, or a form `action` attribute pointing at a mail address.
- Rate limit per IP; honeypot field; validate and sanitise before relaying.
- Relay through a transactional email provider with a support sender address, so replies don't expose the destination inbox.
- Auto-acknowledgement email on submit.
- Include the standing notice that support never asks for a master password, recovery key, or vault contents (Requirements 4.8).

### Phase 4.5 — Admin console
- Separate web app, separate deployment, separate auth from the user-facing product.
- First-launch bootstrap: read **`SECRETPASS_ADMIN_EMAIL` only** — there is no `SECRETPASS_ADMIN_PASSWORD` under the single-password model, and there must not be one. Create the unactivated admin if no admin exists, mark it `is_admin` and `entitlement_override`, issue a standard activation token, then permanently disable the bootstrap path. The admin then activates through the ordinary customer flow (Requirements 4.9). Force 2FA enrollment before the console becomes usable.
- Build admin queries against non-vault tables only. The console's database role should have **no read access to the `vault_items` ciphertext columns at all** — enforce the "admin cannot read vaults" rule in Postgres permissions, not just in application code, so a bug in the console can't become a data breach.
- Append-only audit log of every admin action. Log console actions only — never vault contents or anything derived from them.

**The admin is also a vault user (Requirements 4.9)**

The operator holds one account that carries both roles: a normal vault, and console access. The implementation trick is to keep the two roles on separate connections rather than separate accounts.

- **`is_admin` is a console flag, not a data-access flag.** It gates entry to the console app and nothing else. It must never appear in an RLS policy governing `vault_items`, `folders`, or `sync_log` — the moment an admin flag can widen a vault query, the zero-knowledge property is gone. Vault RLS stays exactly `user_id = auth.uid()` for everyone, admin included.
- **Two database roles, one human.** The client apps connect as the ordinary authenticated user role and read the admin's own vault rows through standard RLS. The console connects as a separate role with no `SELECT` on any ciphertext column. Same person, same login, different connection — and the console role physically cannot return vault data even if someone writes a query asking for it.
- The bootstrap env var carries an **email address and nothing else**. Vault setup (master password → recovery key confirmation) happens through the normal activation flow. This is strictly better than the previous design, which put a password in the environment: anything in an env var has been in shell history, a deploy log, and probably a `.env` file, and under the single-password model there is no server-settable password for it to carry anyway.
- Entitlement: set `entitlement_override` on the admin's user row. Verify the admin's account is excluded from the past_due/grace/lapsed state machine and, critically, from the 90-day dormancy deletion job — an operator who stops using their own vault for three months should not have it deleted by a cron job.
- Console entry requires a fresh 2FA challenge independent of the app session. Unlocking the vault must not open the console; an open console must not unlock the vault.
- **Add an automated test for this specifically:** authenticate as the admin, then attempt to read another user's `vault_items` through PostgREST, and attempt to read *any* `vault_items` ciphertext through the console's role. Both must fail. This extends the existing admin-boundary test in 5a, and it is the test that catches the exact regression this feature makes possible — someone "fixing" admin vault access by widening a policy instead of switching roles.

### Phase 5 — Security testing & launch prep (do not skip, and do not run concurrently with shipping)

Security testing is not one task at the end — the automated half runs from Phase 0 onward, and this phase is where the manual half happens.

**5a. Set up in Phase 0, running continuously**
- SAST in CI: Semgrep or CodeQL on the Flutter app, extension, and backend functions.
- Dependency scanning: Dependabot or Snyk, with the build failing on known-vulnerable packages.
- Secret scanning: gitleaks or trufflehog on every commit and across git history — the admin bootstrap variables make this more than theoretical.
- Cross-implementation crypto test vectors: a fixture file of plaintext/key/ciphertext triples that both the Dart and JS implementations must pass. Add it the day the extension work starts.
- RLS authorization tests: automated tests that authenticate as user A and attempt to read user B's rows through PostgREST directly, not just through your app code.
- Admin-boundary test: assert the console database role gets a permission error on the vault ciphertext columns. Make it fail the build. Since the admin is also a vault user, extend this to assert that an admin-authenticated session can read its **own** vault rows and gets nothing for any other user's — the dual role must not widen RLS.

**5b. Prepare the review package**
- Write the threat model and the cryptographic design document. Reviewers cost less and find more when they aren't reverse-engineering your intent.
- Freeze scope. Don't ship features during the review window.

**5c. Manual testing — mostly other people's clocks, so book early**
- **Third-party penetration test.** Reputable firms schedule weeks out and that lead time is theirs, not something a fast build can compress. Book it well before you need it. Scope: sync API, auth and session handling, activation and recovery flows, the Paddle webhook path, the admin console, the extension, and the Android autofill service.
- **Independent cryptographic review** by someone who didn't write the code. This is the single highest-leverage spend on the whole project. Point them at the split derivation in 2.1 first — the single-password model puts a value derived from the master password on the wire, and whether that value is safely separated from the wrapping key is the first thing a reviewer should be asked to confirm.
- **Mobile testing** against OWASP MASVS/MASTG, Android only for v1 — MobSF for the automated pass, manual work for Keystore usage, biometric key invalidation, background-snapshot leakage, clipboard, and logging.
- **Extension review**: content-script isolation, message passing, and autofill domain matching. Verify the extension will not fill credentials into a look-alike domain — this is the highest-severity bug class an extension can ship.
- **Autofill review, Android**: Digital Asset Links verification against a purpose-built look-alike app, the per-app confirmation fallback, and the lock-boundary behaviour from Requirements 4.3.3 — that a locked vault prompts, that the key is not retained after the fill, and that servicing a fill does not extend the inactivity timer.
- **Business-logic testing**: refund abuse, entitlement bypass, activation-token reuse, subscription-state races.
- **Self-testing you can do meanwhile**: OWASP ZAP or Burp against your own API; verify rate limiting on auth, activation, resend, and contact endpoints; confirm clipboard auto-clear; confirm the vault re-locks as specified.

**5d. Remediate and re-test**
- Fix critical and high findings. Medium findings get an owner and a date.
- **Re-test by the original tester** — self-certifying a fix is how fixes turn out to be incomplete.
- Retain reports and remediation evidence.

**5e. Before you announce**
- Publish a responsible disclosure policy with a security contact. Researchers will find things; give them a route that isn't the contact form.
- Write the incident response plan: who's notified, how users are told, what the disclosure timeline is. Writing it during an incident is too late.
- **Play Store submission package**: listing copy (stating that accounts are created on the website, per Requirements 6.1), phone and tablet screenshots, feature graphic, icon, content rating questionnaire, target audience declaration, Data safety form, privacy policy URL, and working App access credentials for reviewers. Password managers get extra scrutiny, so expect review cycles rather than one clean pass. Apple's review joins this list when iOS ships — its reputation for scrutiny on credential-storage apps is part of why iOS is sequenced second rather than run in parallel.
- **Chrome Web Store submission**: a password manager requires broad host permissions, which is the most heavily scrutinised review category. Write the permission justifications before submitting rather than in response to a rejection.

**After launch:** scanning stays on in CI; repeat the penetration test annually and after any change to the cryptographic design, the sync protocol, or the admin console. Google raises the required Play target API level annually — miss it and the app stops being served to new users, so it is a standing obligation on the most security-sensitive code in the product, not a one-off.

---

## 5.1 Sequencing — what this plan asserts instead of durations

This plan carries no effort estimates. What it does assert is **order**, and the order is not negotiable in four places:

1. **The key hierarchy is validated in Phase 0**, before anything depends on it. Collapsing the two levels into one is the single most expensive mistake available, and it surfaces only when the recovery path is built.
2. **The split derivation is validated in Phase 0**, in the same session, for the same reason — the single-password model is only safe if the auth value and the wrapping key are genuinely independent.
3. **The autofill/auto-lock resolution is decided in Phase 0**, before either Phase 2.4 or Phase 2.6 is built. Deciding it afterwards means rewriting one of them.
4. **Security review (Phase 5) is a gate, not a parallel track.** Freeze scope during the review window; shipping features into an active audit invalidates the audit.

**Three things are not compressible by working faster, because they run on other people's clocks.** Start them early and let them run alongside the build:

- **Organization registration → D-U-N-S issuance → Play verification.** Begin in Phase 0 (see 6.3).
- **Penetration test and cryptographic review scheduling.** Reputable firms book out, and the post-remediation re-test in 5d is a second appointment, not a follow-up email.
- **Store review cycles**, including rejection round-trips — and, if the Play account were ever registered to an individual rather than an organization, the 12-tester/14-day closed testing gate on top (6.3).

Everything else is yours to compress. These are not, which is exactly why they start first.

---

## 5. Key Risks

| Risk | Mitigation |
|---|---|
| Crypto implementation bugs | Use libsodium exclusively; never hand-roll primitives; get an external review before launch |
| Collapsing the two-level key hierarchy into one | The master password must derive a *wrapping* key, never the item-encryption key (2.1). Flattening it silently breaks the recovery key and turns a master-password change into a full vault re-encryption — and it is typically discovered only when the recovery path is built. Write the wrap/unwrap round-trip test in Phase 0, before any vault code depends on it |
| **Auth value and wrapping key not properly separated** (single-password model) | Distinct salts and distinct context for the two Argon2id runs (2.1). Neither derivable from the other. This is the first thing to put in front of the independent cryptographic reviewer, because it is the one place the new password model touches the zero-knowledge boundary |
| **A "forgot password" flow gets built because it looks missing** | There is no server-side reset under the single-password model, and one that appears to work would be a backdoor. Document the absence as deliberate so a future contributor doesn't "fix" it |
| Cross-language encryption mismatch (Flutter vs. extension) | Standardize on libsodium in both; write shared test vectors both sides must pass |
| Browser extension store review delays | Start the extension review process early; Manifest V3 requirements are strict, and broad host permissions draw the slowest review |
| **Android autofill fills credentials into a malicious look-alike app** | Digital Asset Links verification before any dataset is returned; never fall back to package name or label; explicit per-app confirmation where no verified association exists. Test with a purpose-built look-alike app (Phase 2.6) |
| **Autofill keeps the vault key resident to feel seamless, silently defeating auto-lock** | Resolve the conflict in Phase 0 and write the test with the feature. Requirements 4.3.3 is a security boundary, and it is exactly the kind of rule that erodes under UX pressure during implementation |
| **Compromised web delivery serves a malicious build** | Strict CSP, Subresource Integrity, tight deploy-path access control — and state the residual risk honestly rather than implying the web app equals a signed binary. Unavoidable in kind for any browser-delivered manager; the mitigation is honesty plus pipeline hygiene |
| **Play submission rejected because reviewers cannot sign in** | Purchase-first provisioning gives a reviewer no way to create an account. Provision the permanently entitled review account (Phase 4) and re-verify it before every submission |
| **Organization registration started late becomes the launch blocker** | Entity → D-U-N-S → Play verification runs on external clocks and blocks nothing else in the build, which is why it gets deferred. Start it in Phase 0 (6.3) |
| Store review rejection (password managers face extra scrutiny) | Review Google's policies for credential-storage apps before submission; Apple's when iOS ships |
| Lost conversion because mobile apps can't sell or link to a subscription | Make the website the top of the funnel, not the app stores (Requirements 6.1: SEO, paid acquisition, migration positioning, store-listing redirect). US-storefront external links are permanently out of scope, so this is mitigated by marketing, not by app changes |
| Solo developer bandwidth for a security-critical product | Scope ruthlessly (see Requirements doc's "Out of Scope"); don't add features before the audited core is solid |
| Admin console becomes a backdoor | Enforce no-vault-access at the database role level, not just in code; audit-log every action; require 2FA |
| Admin's dual role (vault user + operator) leaks into vault access | Keep `is_admin` out of every vault RLS policy; use a separate console database role with no ciphertext `SELECT`; automated test asserting the admin can read only their own vault and the console role can read none |
| Admin credentials compromised now also exposes a personal vault | The admin's vault is protected by their own master password and 2FA, and is no more exposed than any user's — but the operator's vault likely holds production secrets, so treat its contents and recovery key as infrastructure, and prefer a dedicated secrets manager over the personal vault for production credentials |
| Bootstrap admin credentials leak via env files, shell history, or committed `.env` | Document removing the vars after first launch; force password change on first sign-in; never commit example values that look usable |
| Unencrypted CSV exports leaking user vaults | Require re-authentication, warn prominently in-product, prompt to delete the file after migration |
| App Review rejection if pricing or a checkout link appears in a mobile build | Keep all billing UI out of the app codebase entirely, not behind a feature flag that could ship enabled |
| Paid-only model suppressing signups for a high-trust product | The 30-day money-back guarantee is the only risk-reversal on offer; make it prominent at checkout and in the activation email |
| Accounts paid for but never provisioned (closed tab, dropped network) | Provision on the Paddle webhook, not the redirect; monitor for paid subscriptions with no matching account row |
| Users locked out by a mistyped checkout email | Self-service resend page keyed on the payment email, plus a support path |

---

## 6. Store & Operational Costs

Earlier versions of this plan and the Requirements never put a number on what it costs to *be* in a store. The direct fees turn out to be trivial; the constraints attached to them are not, and those are the part worth planning around.

### 6.1 Direct fees (v1)

| Item | Cost | Notes |
|---|---|---|
| Google Play Console | **$25**, one-time | Never charged again |
| Chrome Web Store | **$5**, one-time | Developer registration |
| Domain | ~$12/year | TLS free via the host |
| Web hosting (Flutter web + marketing site) | $0–20/month | Cloudflare Pages, Netlify or Vercel free tiers cover launch traffic comfortably |
| Legal entity formation | Varies by jurisdiction | Required for organization registration — see 6.3 |
| D-U-N-S number | Free | One to two weeks to issue |
| ~~Apple Developer Program~~ | **$99/year — not a v1 cost** | Starts when iOS work starts. Recurring, and lapsing it pulls shipped apps from sale |
| ~~Mac hardware or cloud macOS~~ | **Not a v1 cost** | Xcode is the only toolchain that signs an iOS build, and it is macOS-only. Deferring iOS defers this entirely |

**Total to launch v1: roughly $42 in fees**, plus entity formation and hosting. Deferring iOS is what removes both the recurring $99 and the Mac from the launch budget — the two largest platform costs in the original scope.

### 6.2 The cost that isn't being paid

Apple and Google take **15–30% of in-app purchases**. The Paddle-only design in Requirements 4.5 avoids that entirely: Paddle charges roughly 5% + 50¢ as merchant of record and absorbs global VAT/GST registration and filing in exchange. On a $50 annual subscription that difference is substantial, and it is the justification for accepting the conversion cost described in Requirements 6.1.

Worth stating explicitly in both documents, because the conversion cost is highly visible — every user who finds the app before the website is a felt loss — while the commission saving is invisible and never shows up as a line item. A plan that only records the visible half of that trade will eventually be re-litigated by someone reading the funnel numbers.

### 6.3 Organization registration, and why it is on the critical path

Store accounts are registered to an **organization**, not an individual. Three reasons:

1. **It exempts the Play account from the closed-testing gate.** Personal developer accounts created after November 2023 must run a closed test with **12 testers who genuinely use the app for 14 continuous days** before production access is granted (the requirement was 20 testers until December 2024). Google checks for real engagement, not installs. For a pre-audit password manager this means asking twelve people to put credentials into an unaudited build — an unappealing proposition on both sides. Organization accounts skip the gate.
2. **It keeps personal identity off the store listing.** Individual enrollment publishes the developer's legal name as seller, and Play requires a verified contact address on the listing. For a paid security product a company name is both a trust signal and a privacy measure.
3. **Paddle requires business details anyway** as merchant of record, so a single entity serves billing and both stores.

The chain is **entity formation → D-U-N-S issuance → Play organization verification**, and only the first step is under your control. It blocks nothing else in the build — which is exactly why it gets forgotten until it is the last thing standing between a finished app and a listing. Start it in Phase 0.

One coupling worth noting: the entity's jurisdiction is what a governing-law clause in the Terms would name. Requirements 8 defers legal review until after launch; choosing the entity with that eventual review in mind costs nothing now and avoids re-papering later.

### 6.4 What each deployment target actually involves

**Web** — `flutter build web`, push static files, done. No gatekeeper, no review, deploy as often as you like. This is why Phase 1 ships here first. Two caveats: the marketing site is a separate static build (Section 1), and the delivery path is itself a threat-model entry (Requirements 5.1.1).

**Google Play** — Build an AAB; APKs are not accepted for new apps. You hold an upload key while Google holds the distribution key via Play App Signing, so a lost upload key is recoverable — one of the few genuinely forgiving parts of store deployment. Console setup is real work: listing copy, phone and tablet screenshots, a 1024×500 feature graphic, a 512×512 icon, content rating questionnaire, target audience declaration, **Data safety form** (where the zero-knowledge model is declared), privacy policy URL, and **App access credentials for reviewers** (Requirements 4.5). Ship via staged rollout, which can be halted. Ongoing: the target API level must be raised annually or the app stops reaching new users.

**Chrome Web Store** — Manifest V3 package submitted as a zip. Broad host permissions put a password manager in the most heavily scrutinised review category, and every permission needs a written justification. Each update is re-reviewed.

**iOS, when it comes** — Mac, Apple organization enrollment, App Store Connect, TestFlight, per-device-class screenshots, privacy nutrition labels, and a native `ASCredentialProviderExtension` in Swift for autofill. Plan it as a platform project, not an additional build target.

---

## 7. Suggested Immediate Next Steps

1. **Start the entity formation and D-U-N-S application.** External clock, blocks nothing, will otherwise become the launch blocker (6.3).
2. Stand up the Supabase project and validate the Argon2id + libsodium round-trip in a throwaway Flutter script — including the **split derivation** of wrapping key and auth value under distinct salts (2.1).
3. Decide the autofill/auto-lock resolution (Requirements 4.3.3) on paper before Phase 2.4 or 2.6 begins.
4. Build the unlock + vault CRUD flow end-to-end on Web only, before touching Android.
5. Get one outside pair of eyes (even informally) on the encryption design — specifically the single-password split derivation — before writing a lot more code around it.
