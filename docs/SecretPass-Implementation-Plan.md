# SecretPass — Implementation Plan

**Version:** 1.1 (Draft)
**Companion to:** SecretPass Requirements Document v1.2
**Last updated:** 14 August 2026

---

## 1. Tech Stack Summary

| Layer | Choice | Notes |
|---|---|---|
| Client apps (iOS, Android, Web) | **Flutter** | Single codebase, matches your stated preference |
| Browser extension | **JS/TS, Manifest V3** | Separate small codebase — Flutter can't produce a browser extension |
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
┌─────────────┐   ┌─────────────┐   ┌──────────────┐   ┌─────────────────┐
│  iOS App    │   │ Android App │   │   Web App    │   │ Browser         │
│  (Flutter)  │   │  (Flutter)  │   │  (Flutter)   │   │ Extension (JS)  │
└──────┬──────┘   └──────┬──────┘   └──────┬───────┘   └────────┬────────┘
       │                 │                 │                    │
· · · ·│· · · · · · · · ·│· · · · · · · · ·│· · · · · · · · · · │· · · · ·
       │      CLIENT SIDE — plaintext, vault key, all crypto    │
       │                 │                 │                    │
       │      SERVER SIDE — ciphertext only, never the key      │
       └────────┬────────┴────────┬────────┴───────────┬────────┘
                │                 │                    │
                     ┌────────────▼─────────────┐
                     │        Supabase          │
                     │  (Auth + Postgres +      │
                     │   encrypted blob storage)│
                     └──────────────────────────┘
```

Everything above the dotted line encrypts and decrypts locally — all four clients, the extension included. Supabase's job is limited to authenticating the user and storing/syncing opaque encrypted blobs; it never has the vault key and cannot derive it.

### 2.1 Zero-knowledge flow

**Two passwords, deliberately separate.** The *account password* authenticates to Supabase Auth and travels to the server on every sign-in. The *master password* never leaves the device and is the sole input to the vault key derivation. Keep the two code paths from ever touching: the master password must not be sent to Supabase, logged, or reused as an auth credential, and a server-side account-password reset must have no effect whatsoever on vault decryptability. This separation is what makes "forgot my account password" a routine email flow rather than a vault-loss event.

**Key hierarchy — two levels, not one.** The master password does *not* encrypt vault items. It derives a wrapping key, which encrypts the vault key, which encrypts the items. That indirection is what makes both the recovery key (2.2) and master-password changes possible. Getting this wrong — collapsing it to one level — is the single most expensive mistake available here, because it is only discovered when the recovery path is built and found not to work.

1. At activation, the client generates a **random vault key** (`crypto_secretbox_keygen`). This key, and only this key, ever encrypts vault data.
2. The user sets a master password. Argon2id derives a **wrapping key** from it, using a per-user random salt stored server-side — the salt is not secret, only the derived key is.
3. The vault key is encrypted ("wrapped") under that wrapping key. The wrapped copy is stored server-side; the vault key itself is never stored anywhere in the clear.
4. Vault items are encrypted locally with the **vault key** using libsodium's `crypto_secretbox` (XChaCha20-Poly1305), one ciphertext blob per item. Only ciphertext is ever sent to Supabase.
5. On unlock, the client re-derives the wrapping key from the master password + salt, unwraps the vault key, and holds it in memory only while the vault is unlocked (see Requirements 4.7).
6. On a new device: same master password, same salt, same derivation — unwraps the same vault key, which decrypts the synced blobs.

Two consequences worth stating, because they are the reason for the indirection:
- **Changing the master password re-wraps one key.** Derive a new wrapping key, re-encrypt the vault key under it, upload. Vault items are untouched — no bulk re-encryption, no risk of a partial rewrite corrupting a vault.
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
| `users` | `id`, `email`, `kdf_salt`, `created_at`, `is_admin`, `entitlement_override`, `refunds_issued` | Managed largely by Supabase Auth. `is_admin` grants console access only — it must never appear in any RLS policy governing vault reads. `entitlement_override` marks the admin's account permanently entitled without a `subscriptions` row. `refunds_issued` enforces the one-automatic-refund-per-customer rule at the database level |
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

### Phase 0 — Foundations (1–2 weeks)
- Repo setup, CI/CD skeleton, Flutter project scaffolding for iOS/Android/Web targets.
- **Turn on the automated security tooling now** (see Phase 5a). Retrofitting secret scanning after a key has already been committed to history is a much worse day.
- Supabase project setup: Auth, Postgres schema, RLS policies.
- Integrate libsodium in Flutter; validate Argon2id + XChaCha20-Poly1305 round-trip.
- **Validate the full two-level key hierarchy (2.1) before building anything on top of it:** generate a vault key, wrap it under a master-password-derived key *and* under a recovery key, unwrap via both paths, and confirm a master-password change re-wraps without touching item ciphertext. This is a day's work now and a rewrite later.
- Measure Argon2id on real mid-range hardware and record the parameters and timings — Requirements 5.2 budgets unlock against measured numbers, not guesses.

### Phase 1 — Core vault MVP (3–5 weeks)
- Master password setup + unlock flow.
- Local encrypted SQLite store.
- Vault item CRUD (create/read/update/delete), search, folders/tags.
- Sync engine: push/pull encrypted blobs to/from Supabase, basic last-write-wins conflict resolution.
- Ship on Web first (fastest iteration loop), then Android, then iOS.

### Phase 2 — Generator & TOTP (1–2 weeks)
- Password generator screen + inline generator (when creating/editing an item).
- TOTP secret storage (manual entry + QR scan) and live code display.

### Phase 2.4 — Biometric unlock (3–5 days)
- Use `local_auth` for the platform prompt, and store the wrapped vault key in the iOS Keychain / Android Keystore with biometric-gated access — do not hand-roll key storage.
- Set the key invalidation flag so enrolling a new fingerprint or face on the device invalidates the stored key. This is one line of configuration and it's the difference between biometrics being a convenience and being a hole.
- Enforce master-password re-entry on reboot, after inactivity, after failed biometric attempts, and before export, recovery-key display, master-password change, or account deletion.
- **Auto-lock (Requirements 4.7):** 5-minute default, user-configurable 1–60 minutes, no "never". Lock immediately on app background, device lock, and restart regardless of the setting. Locking must zero the vault key and decrypted items in memory — a lock that only swaps the visible route leaves the plaintext vault sitting in a Dart object for anyone who can attach a debugger or trigger a heap dump. Test that explicitly; it is the kind of thing that silently regresses.

### Phase 2.5 — CSV import/export, Terms, account deletion (1–2 weeks)
- CSV import with a column-mapping step; parse with a real CSV library, validate before committing, all-or-nothing write.
- CSV export gated behind master-password re-authentication, with an unmissable plaintext warning in the UI.
- Terms of Service screen in the app (Settings), and the **activation-time** consent modal and checkbox. The *checkout-time* consent lives on the marketing site and is built in Phase 4 — the two consent points ship in different phases, so neither should assume the other exists. Write both agreements to `tos_agreements` with the version and timestamp.
- Self-service account deletion (Requirements 4.13): master-password re-auth, an explicit irreversibility warning, a CSV export offered from the confirmation screen itself, and a Paddle subscription cancellation as part of the same flow. Required by both app stores, so it is not optional polish.

### Phase 3 — Browser extension, Chrome only (2–3 weeks)
- Manifest V3 extension: auth against the same Supabase project.
- Chrome only at launch. Keep the code free of Chrome-specific APIs where a standard equivalent exists, so Edge and Firefox are a port rather than a rewrite later.
- Form detection, autofill, save-new-credential prompt.
- Reuse the same libsodium.js-based crypto so the extension reads/writes the identical ciphertext format as the apps.

### Phase 4 — Subscription & entitlements (3–4 weeks)

The largest phase after the vault core, and previously the most under-estimated part of this plan at one week. What follows is a payment integration, a five-state lifecycle, a provisioning path, a refund system with abuse controls, and six or so transactional emails — none of which can be half-built, because every one of them is a path where money and vault access meet. Budget accordingly, and expect a meaningful share of it to be spent testing webhooks against Paddle's sandbox rather than writing code.

- Integrate Paddle checkout on the marketing website, including the **checkout-time** Terms consent checkbox that blocks progress to payment (Requirements 4.11).
- Paddle webhook → Supabase `subscriptions` table; entitlement is a single boolean per user.
- Apps check that boolean at sign-in and on sync. No billing SDK ships in the mobile or web app at all.
- Ensure the mobile builds contain no purchase flow — no subscribe button, no checkout link, and no pricing shown to anyone who isn't already subscribed (see Requirements 4.5). This is an App Review rejection risk if it slips in. Showing an existing subscriber their own plan, price, renewal date, and refund amount is account management and is permitted; the rule is about initiating or advertising a purchase, not about displaying numbers.
- **Purchase-first provisioning:** Paddle webhook creates the account row and issues a single-use activation token (7-day expiry). Provision on the webhook, never on the browser redirect.
- Activation page: consumes the token, then walks the user through account password → master password → recovery key confirmation.
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

### Phase 4.4 — Contact page (2–3 days)
- Public form on the marketing site; no authentication required.
- Server-side handler holds the destination address in an environment variable — never in client code, page source, or a form `action` attribute pointing at a mail address.
- Rate limit per IP; honeypot field; validate and sanitise before relaying.
- Relay through a transactional email provider with a support sender address, so replies don't expose the destination inbox.
- Auto-acknowledgement email on submit.
- Include the standing notice that support never asks for a master password, recovery key, or vault contents (Requirements 4.8).

### Phase 4.5 — Admin console (2–3 weeks)
- Separate web app, separate deployment, separate auth from the user-facing product.
- First-launch bootstrap: read `SECRETPASS_ADMIN_EMAIL` / `SECRETPASS_ADMIN_PASSWORD`, create the single admin only if no admin exists, then permanently disable the bootstrap path. Force password change and 2FA enrollment on first sign-in.
- Build admin queries against non-vault tables only. The console's database role should have **no read access to the `vault_items` ciphertext columns at all** — enforce the "admin cannot read vaults" rule in Postgres permissions, not just in application code, so a bug in the console can't become a data breach.
- Append-only audit log of every admin action. Log console actions only — never vault contents or anything derived from them.

**The admin is also a vault user (Requirements 4.9)**

The operator holds one account that carries both roles: a normal vault, and console access. The implementation trick is to keep the two roles on separate connections rather than separate accounts.

- **`is_admin` is a console flag, not a data-access flag.** It gates entry to the console app and nothing else. It must never appear in an RLS policy governing `vault_items`, `folders`, or `sync_log` — the moment an admin flag can widen a vault query, the zero-knowledge property is gone. Vault RLS stays exactly `user_id = auth.uid()` for everyone, admin included.
- **Two database roles, one human.** The client apps connect as the ordinary authenticated user role and read the admin's own vault rows through standard RLS. The console connects as a separate role with no `SELECT` on any ciphertext column. Same person, same login, different connection — and the console role physically cannot return vault data even if someone writes a query asking for it.
- The bootstrap env vars set the account login password only. Vault setup (master password → recovery key confirmation) happens in the client app on first sign-in, through the normal user flow. Never derive a vault key from a bootstrap credential — anything in an env var has been in shell history, a deploy log, and probably a `.env` file.
- Entitlement: set `entitlement_override` on the admin's user row. Verify the admin's account is excluded from the past_due/grace/lapsed state machine and, critically, from the 90-day dormancy deletion job — an operator who stops using their own vault for three months should not have it deleted by a cron job.
- Console entry requires a fresh 2FA challenge independent of the app session. Unlocking the vault must not open the console; an open console must not unlock the vault.
- **Add an automated test for this specifically:** authenticate as the admin, then attempt to read another user's `vault_items` through PostgREST, and attempt to read *any* `vault_items` ciphertext through the console's role. Both must fail. This extends the existing admin-boundary test in 5a, and it is the test that catches the exact regression this feature makes possible — someone "fixing" admin vault access by widening a policy instead of switching roles.

### Phase 5 — Security testing & launch prep (4–6 weeks, do not skip)

Security testing is not one task at the end — the automated half runs from Phase 0 onward, and this phase is where the manual half happens.

**5a. Set up in Phase 0, running continuously (half a day to configure)**
- SAST in CI: Semgrep or CodeQL on the Flutter app, extension, and backend functions.
- Dependency scanning: Dependabot or Snyk, with the build failing on known-vulnerable packages.
- Secret scanning: gitleaks or trufflehog on every commit and across git history — the admin bootstrap variables make this more than theoretical.
- Cross-implementation crypto test vectors: a fixture file of plaintext/key/ciphertext triples that both the Dart and JS implementations must pass. Add it the day the extension work starts.
- RLS authorization tests: automated tests that authenticate as user A and attempt to read user B's rows through PostgREST directly, not just through your app code.
- Admin-boundary test: assert the console database role gets a permission error on the vault ciphertext columns. Make it fail the build. Since the admin is also a vault user, extend this to assert that an admin-authenticated session can read its **own** vault rows and gets nothing for any other user's — the dual role must not widen RLS.

**5b. Prepare the review package (3–5 days)**
- Write the threat model and the cryptographic design document. Reviewers cost less and find more when they aren't reverse-engineering your intent.
- Freeze scope. Don't ship features during the review window.

**5c. Manual testing (3–4 weeks, mostly waiting on others)**
- **Third-party penetration test.** Book this 4–8 weeks ahead; good firms are not available on short notice. Scope: sync API, auth and session handling, activation and reset flows, the Paddle webhook path, the admin console, the extension.
- **Independent cryptographic review** by someone who didn't write the code. This is the single highest-leverage spend on the whole project.
- **Mobile testing** against OWASP MASVS/MASTG — MobSF for the automated pass, manual work for keychain/keystore usage, biometric key invalidation, background-snapshot leakage, clipboard, and logging.
- **Extension review**: content-script isolation, message passing, and autofill domain matching. Verify the extension will not fill credentials into a look-alike domain — this is the highest-severity bug class an extension can ship.
- **Business-logic testing**: refund abuse, entitlement bypass, activation-token reuse, subscription-state races.
- **Self-testing you can do meanwhile**: OWASP ZAP or Burp against your own API; verify rate limiting on auth, activation, resend, and contact endpoints; confirm clipboard auto-clear; confirm the vault re-locks as specified.

**5d. Remediate and re-test (1–2 weeks)**
- Fix critical and high findings. Medium findings get an owner and a date.
- **Re-test by the original tester** — self-certifying a fix is how fixes turn out to be incomplete.
- Retain reports and remediation evidence.

**5e. Before you announce**
- Publish a responsible disclosure policy with a security contact. Researchers will find things; give them a route that isn't the contact form.
- Write the incident response plan: who's notified, how users are told, what the disclosure timeline is. Writing it during an incident is too late.
- App Store / Play Store review prep — password managers get extra scrutiny, so plan for review cycles.

**After launch:** scanning stays on in CI; repeat the penetration test annually and after any change to the cryptographic design, the sync protocol, or the admin console.

**Rough total: ~18–28 weeks** for one experienced developer working solo, before factoring in the security review timeline (audits typically add 2–4 weeks of calendar time, much of it waiting).

That total went up from ~16–25 weeks when Phase 4 was re-estimated from 1 week to 3–4. The original number wasn't wrong about the work — it was wrong about the billing phase, and a plan that under-estimates the phase where money and vault access meet is the wrong plan to be optimistic in.

---

## 5. Key Risks

| Risk | Mitigation |
|---|---|
| Crypto implementation bugs | Use libsodium exclusively; never hand-roll primitives; get an external review before launch |
| Collapsing the two-level key hierarchy into one | The master password must derive a *wrapping* key, never the item-encryption key (2.1). Flattening it silently breaks the recovery key and turns a master-password change into a full vault re-encryption — and it is typically discovered only when the recovery path is built. Write the wrap/unwrap round-trip test in Phase 0, before any vault code depends on it |
| Cross-language encryption mismatch (Flutter vs. extension) | Standardize on libsodium in both; write shared test vectors both sides must pass |
| Browser extension store review delays | Start the extension review process early; Manifest V3 requirements are strict |
| App Store rejection (password managers face extra scrutiny) | Review Apple's guidelines for credential-storage apps before submission |
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

## 6. Suggested Immediate Next Steps

1. Stand up the Supabase project and validate the Argon2id + libsodium round-trip in a throwaway Flutter script.
2. Build the unlock + vault CRUD flow end-to-end on Web only, before touching mobile.
3. Get one outside pair of eyes (even informally) on the encryption design before writing a lot more code around it.
