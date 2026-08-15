# SecretPass — Product Requirements Document

**Version:** 1.4 (Draft)
**Status:** Pre-development
**Owner:** Ghenadie Berco
**Last updated:** 14 August 2026

---

## 1. Overview

SecretPass is a cross-platform password manager built to be sold as a subscription product. It stores a user's credentials, generates strong passwords, and produces TOTP (2FA) codes, all protected by a zero-knowledge encryption model — SecretPass's servers never see plaintext vault data.

**Goals for v1:**
- Ship a trustworthy, secure vault core on the platforms that matter most to early users.
- Prove out the subscription business model.
- Keep the v1 feature set narrow enough to ship, review, and (critically) get security-reviewed before wide release.

**Non-goals for v1:** family/team sharing, breach monitoring, desktop-native apps (Windows/macOS/Linux), secure notes/card storage. These are explicitly deferred — see Section 7.

---

## 2. Target Platforms

| Platform | Priority | Delivery mechanism |
|---|---|---|
| Web | v1 | Flutter web build; built first, and serves as the "desktop" experience via browser |
| Android | v1 | Flutter app **plus a native autofill service** (see 4.3.2), Google Play |
| Browser extension | v1 | **Chrome only** at launch (Manifest V3). Edge and other Chromium browsers accept the same package with minimal change; Firefox and Safari are post-v1. Separate lightweight JS codebase |
| iOS | **Post-v1** | Flutter app plus a native credential-provider extension, App Store |
| Windows / macOS / Linux (native) | Post-v1 | Flutter desktop build, once mobile+web are stable |

The web app is the intended desktop experience for v1 — it covers the "must work on desktop" requirement without the extra QA burden of three native desktop builds before the product is proven.

**iOS is deferred out of v1 — decided, and worth recording why.** Producing a signed iOS build requires macOS; Xcode is the only toolchain that can do it, and there is no path to one from a Linux or Windows development machine. Deferring iOS therefore removes a hardware dependency from v1 outright, removes the strictest of the three store reviews from the launch path, and removes the Keychain/Secure Enclave half of the biometric work along with a second native autofill implementation.

The cost is real and should not be minimised: iOS users convert better on paid security products. But because no platform may sell a subscription in-app (4.5), the acquisition funnel runs through the website on every platform regardless. Deferring iOS defers *installs*, not a sales channel — which makes it a materially cheaper trade for SecretPass than it would be for a typical consumer app. iOS ships once the vault core is audited and the model is proven.

---

## 3. User Persona (v1)

A single individual (not a family or team) who currently uses browser-saved passwords or a spreadsheet, wants a real password manager, and is willing to pay a small subscription for cross-device sync, autofill, and 2FA codes in one place.

---

## 4. Functional Requirements

### 4.1 Core vault (baseline — required for the product to function at all)
- Create, read, update, delete vault items.
- Search and filter vault items by title, username, email, or website.
- Organize items with folders or tags.
- Vault syncs across all of a user's signed-in devices.
- Local copy of the vault is available offline; changes sync when connectivity returns.

**Vault item fields:**

| Field | Required | Notes |
|---|---|---|
| Title | **Yes** | Display name in the list; needed so items are identifiable and searchable |
| Email | No | |
| Username | No | Separate from email — many services use both |
| Password | No | |
| Website | No | Also used to match the item during browser autofill |
| Note | No | Free-text |
| TOTP secret | No | Stored per item; see 4.4 |

All fields except Title are optional — an item may be saved with only a title filled in. Empty fields are hidden in the item detail view rather than shown blank.

### 4.2 Password generator
- Generate random passwords with configurable length, character sets (upper/lower/numbers/symbols), and ambiguous-character exclusion.
- One-tap copy to clipboard (with auto-clear after a short delay).
- Generator accessible standalone (not just when creating a vault item) — e.g., for signing up on a site before a vault item exists.

### 4.3 Autofill

Autofill is **two separate implementations against two entirely different operating-system mechanisms**, sharing no code between them. Both are in v1. Treating this as one feature is the mistake to avoid — the browser extension cannot serve mobile, and the mobile service cannot serve desktop.

#### 4.3.1 Browser extension (desktop)
- Detect login forms on web pages and offer to fill saved credentials.
- Offer to save new credentials when a user submits a new login form.
- Trigger the password generator inline on signup/password-change forms.
- Extension authenticates to the same account and reads the same encrypted vault as the Android and web apps.
- **Match on the registrable domain, never on a substring of the URL.** The extension must not fill credentials into a look-alike domain.

#### 4.3.2 Android autofill (mobile)

On a phone, most logins happen inside native applications — the banking app, the social app — where there is no web page for an extension to inject into. The extension model simply does not exist on mobile. Android instead exposes a system-level provider interface: an app registers as the device's autofill provider, and thereafter, when the user focuses a credential field *in any application*, the OS asks that provider what it holds and surfaces the suggestion above the keyboard.

Registering as that provider is what makes SecretPass a password manager on mobile. Without it, the Android app is a searchable list that users copy out of one field at a time, switching apps between each — which is the experience the product exists to replace.

- Implement an `AutofillService` (Android Autofill Framework, API 26+ / Android 8.0) so SecretPass is offered as a fill source in any app or mobile browser.
- **This is native Kotlin, not Flutter.** Flutter renders into a single canvas view; the autofill service is a separate Android component that the operating system instantiates and calls directly, outside the Flutter engine. It is platform work, and planning it as a Flutter screen will underestimate it.
- Offer to save newly entered credentials captured from other apps.
- Onboarding must actively prompt the user to set SecretPass as the device autofill provider. It is disabled until explicitly enabled in system settings, and users do not find that setting on their own — an autofill implementation nobody turns on is indistinguishable from one that was never built.
- **App-to-domain matching is a security requirement, not a convenience.** Before offering any credential, verify the requesting application's signing certificate against the target domain's Digital Asset Links (`assetlinks.json`). Handing a saved banking password to a look-alike application is the mobile form of filling into a look-alike domain, and it is the highest-severity defect this feature is capable of shipping.
- Where no verified association exists, **do not fall back to package-name or app-label matching** — both are chosen by the attacker. Require explicit, per-app user confirmation before the first fill, then remember that decision.
- Credential Manager (Android 14+) is where Google is taking this, and passkeys arrive with it. Keep the vault lookup logic cleanly separated from the `AutofillService` plumbing so that adopting Credential Manager later is an addition rather than a rewrite.

#### 4.3.3 Autofill against a locked vault — the constraint both implementations must satisfy

The autofill service is a **second entry point into the vault, invoked by the operating system while the SecretPass app is backgrounded** — which by 4.7 is exactly when the vault is locked and the vault key has been zeroed. These two requirements are in direct tension, and the resolution must be settled before either is built, because discovering it afterwards means rewriting one of them.

The resolution, stated once:

- **Auto-lock (4.7) wins as written.** A backgrounded vault is locked, and a locked vault holds no key in memory. Autofill must **not** keep the vault key alive in a background-reachable form in order to make itself seamless.
- An autofill request against a locked vault therefore **authenticates before it fills**: a biometric prompt (4.6), falling back to the master password in the cases 4.6 enumerates. The resulting unlock is scoped to servicing that single request and is not retained afterwards.
- A fill performed while the vault is already unlocked and inside the inactivity window requires no additional prompt.
- **Servicing an autofill request does not count as interaction with the app** for the purposes of the 4.7 inactivity timer, and must never extend it.

This makes SecretPass marginally less frictionless than managers that keep a key resident in the background. That is the intended trade, and it should be described honestly in the cryptographic design document rather than quietly softened during implementation.

### 4.4 TOTP / 2FA code storage
- Store TOTP secrets (via manual entry or QR code scan) alongside the related vault item.
- Display a live, auto-refreshing 6-digit code with a countdown indicator.
- One-tap copy of the current code.

### 4.5 Account & subscription
**A single master password per account**

There is **one password**, not two. An earlier draft specified a separate account login password alongside the master password; that has been reversed (see 8).

- The master password never leaves the device. An **authentication value is derived from it client-side, under different KDF parameters and a different salt**, and that derived value — never the password itself — is what authenticates to Supabase. The server therefore still never receives anything that unlocks the vault, and the zero-knowledge property is fully intact. See Implementation Plan 2.1 for the derivation.
- The two-level key hierarchy is unchanged by this decision: the master password still derives a *wrapping* key, which unwraps the vault key, which decrypts items. Only the authentication path changed.
- **Consequence, stated plainly, because it is the entire trade:** there is no server-side password reset that restores vault access. A forgotten master password is recovered with the recovery key (4.10), or not at all. This is not a newly introduced failure mode — it was already true of the master password under the two-password design — but it is now the *only* password, so the recovery key carries the whole burden. The save-confirmation step in 4.10 becomes the single most important screen in onboarding, and should be treated as such rather than as a dismissible interstitial.
- **Why one and not two.** Two passwords would have made "forgot my login" a routine email reset, but at the cost of asking a user who has already paid to create and distinguish three secrets in one sitting — account password, master password, recovery key — at the highest-drop-off moment in the funnel. Worse, nothing prevents a user from setting both passwords to the same string, and many would: at that point the credential sent to the server on every sign-in *is* the vault-unlocking secret, and zero knowledge is broken through the UI behaving exactly as designed. One password removes that failure by construction. Activation now asks for two secrets instead of three.

**Account existence**

- **No customer accounts exist by default.** Every customer must purchase and activate; there is no pre-seeded or shared customer account.
- **Exception one: the bootstrap admin account** (4.9). It is created from an environment variable on first launch rather than by purchase, it never pays, and it is permanently entitled. It is a full vault user like any customer — the exception is how it comes into existence and how it is entitled, not what it can do. Exactly one such account exists, it is created once, and the bootstrap path disables itself permanently afterwards.
- **Exception two: the store review account.** Google Play reviewers must be able to sign in and exercise the app. Because purchase-first provisioning means a reviewer arriving at the sign-in screen has no way to create an account, a submission without working credentials is rejected for reviewer inaccessibility — a common and entirely avoidable rejection, and one that costs a full review cycle each time. Provision one permanently entitled review account, supply its credentials in the Play Console's App access section, and treat it as production: it holds no real credentials, it is excluded from the dormancy deletion job, and its master password and recovery key are operational secrets. The same account serves Apple when iOS ships. Verify it signs in from a clean device before every submission — a stale review account wastes a review cycle as surely as a missing one.
- SecretPass is paid only: **$5/month or $50/year** (annual saves ~17%). All features — vault, generator, sync, browser extension, TOTP — are included at both price points. There is one tier, and it is paid.
- **30-day money-back guarantee:** users can request a full refund within 30 days of purchase, no questions asked. This is how someone evaluates SecretPass without risk.

**Account creation: purchase-first provisioning**

No customer account exists until payment succeeds. This is deliberate — because the mobile apps can't show pricing or link to checkout (see below), an unpaid account on mobile would be a dead end with nowhere for the user to go. (The bootstrap admin and store review accounts are the two exceptions, per above.)

1. Visitor chooses monthly or yearly on the SecretPass website.
2. Paddle checkout collects email and payment. No account or password is created at this stage — fewer fields before payment.
3. On payment success, the **Paddle webhook** provisions the account. The webhook is the source of truth, not the browser redirect: users close tabs and networks drop, and provisioning on redirect loses accounts that were paid for.
4. An activation email is sent containing a **single-use link, valid for 7 days**.
5. The user follows the link and completes setup: **master password, then the recovery key** with its save-confirmation step. Two secrets, not three.
6. Vault created. The user can now sign in on web or Android (and on iOS once it ships).

**Activation link rules**
- Valid 7 days; single-use, consumed once setup completes.
- **The link expires; the entitlement does not.** An expired link never requires re-purchase.
- Self-service, unlimited resend from a public page that accepts the email used at checkout. This page is also the recovery path for a mistyped checkout email.

**Failed renewal payments**
- Paddle's dunning retries run first (typically ~2 weeks of retries at widening intervals).
- Once Paddle marks the subscription past due, a **7-day grace period** begins. The account stays fully functional throughout.
- Reminder emails at day 1 and day 5 of grace, each linking to update payment details.
- The account moves to the lapsed state at day 7. By then the user has had roughly three weeks of notice — sync should never stop as a surprise.

**Self-service cancellation and automatic refunds**
- Users can cancel their subscription from within the app, at any time, without contacting anyone.
- **If the cancellation falls within 30 days of the initial purchase, the full refund is issued automatically** via the Paddle API — no admin approval, no support ticket, no developer involvement in the normal path.
- The cancel flow states plainly which outcome applies before the user confirms: inside the window, "you'll be refunded in full"; outside it, "you'll keep access until [date], and no further payments will be taken."
- The refund and the resulting state change are driven by Paddle webhooks, so the account transitions to lapsed automatically once the refund settles.
- Requests that fail at the payment provider must be retried and, if still failing, surfaced in the admin console's "needs attention" queue — the only case where a human sees a refund at all.
- **Abuse control:** automatic refunds are limited to one per customer. A second refund request from the same account or payment method goes to manual review. Without this, subscribe → import vault → export CSV → refund is a repeatable way to use the product without paying. This is fraud control, not an approval step on the normal path.
- **Scope of the guarantee:** it applies to the initial purchase. Renewal charges (month two onward, or year two) are not automatically refundable — cancellation stops future billing instead. Note that EU/UK consumer law grants separate statutory withdrawal rights; Paddle as merchant of record handles those obligations, and the Terms must not claim less than the law provides.

**Refunds and chargebacks — handled differently**
- **Refund** (customer decision, incl. the 30-day guarantee): account moves to the **lapsed state**, not deleted. The vault stays readable on devices it's already on and CSV export remains available, so the user can retrieve credentials they may have migrated in and not backed up elsewhere. Deleting a vault on refund would destroy what may be the only copy of a user's credentials.
- **Chargeback** (fraud signal): suspend immediately; the account may not resubscribe without review. Same data handling, different account status.
- **Dormancy horizon:** lapsed vaults are not kept indefinitely. After 90 days lapsed, send warning emails at 90, 97, and 104 days, then delete. This bounds storage and liability without ambushing anyone.

**Payment processing**
- **One processor for everything: Paddle**, acting as merchant of record. (Lemon Squeezy is a comparable alternative.)
- **All subscriptions are purchased on the SecretPass website.** There is no in-app purchase on iOS or Android.
- The Android and web apps are sign-in only: a user subscribes on the web, then signs in on any platform with an already-paid account.
- Consequence: no StoreKit, no Google Play Billing, no RevenueCat, no cross-rail entitlement reconciliation, and no duplicate-subscription bugs. One integration, one webhook, one payout.
- **Tax:** Paddle is the seller of record and handles VAT/GST/sales-tax registration, collection, and remittance globally. SecretPass does not register for tax in the countries it sells into. This is the reason for choosing a merchant of record over Stripe.

**App store constraint — important for the mobile builds**
- Apple and Google restrict how apps may reference outside purchasing. The US storefront now permits external purchase links, but most other regions do not, and SecretPass is selling globally.
- The safe design for a global launch: **the mobile apps contain no purchase flow — no "Subscribe" button, no link to checkout, and no pricing shown to anyone who is not already a subscriber.** A signed-out or unentitled user sees a sign-in screen and a neutral message that the account isn't active, without being directed anywhere.
- **This rule is about purchasing, not about arithmetic.** An existing subscriber may be shown their own plan, their own price, their renewal date, and the exact refund amount in the cancel flow. That is account management, which the stores permit, and the cancel flow cannot be honest without it — "you'll be refunded in full" is a worse disclosure than "you'll be refunded $50". The line to hold: nothing in the app may *initiate or advertise* a purchase; showing a subscriber the terms of the subscription they already bought is fine.
- "Update payment method" may link out to Paddle's customer portal for an existing subscriber. This is billing management for an active subscription, not an external purchase link, and it is the standard pattern for a merchant-of-record setup.
- This is a real conversion cost. A user who downloads the app first has to find the website on their own. Plan marketing so that the website, not the app store, is the top of the funnel — see Section 6.1.
- **Region-gated external purchase links are out of scope permanently, not deferred.** Region-aware links are permitted on the US storefront and would recover some of this conversion, but the decision is to keep every mobile build identical and free of billing UI in all markets. One App Review story, no region-specific behaviour to maintain, and no chance of a region flag shipping wrong. The conversion cost is accepted as the price of that simplicity.

**Lapsed and cancelled subscriptions**
- When a subscription lapses or is cancelled, the user **must not lose access to their own passwords.** Locking someone out of their credentials over an expired card is both a serious user-harm event and a support and reputation problem.
- Required behaviour on lapse: the local vault remains readable on the device it is already on, and CSV export remains available. Sync, the browser extension, and new-item creation may be suspended.
- Deleting or making a paying-customer-turned-lapsed user's vault unreadable is explicitly out of scope and should never be implemented.

### 4.6 Biometric unlock
- Fingerprint or face unlock on Android, in v1. (Face ID / Touch ID arrives with the iOS app post-v1; the requirement below is written to cover both, so that iOS is an implementation of an existing rule rather than a new one.)
- Biometrics unlock a copy of the vault key held in the platform's secure hardware — **Android Keystore** for v1, iOS Keychain with Secure Enclave when iOS ships — released only on a successful biometric match. Keep this behind a platform-agnostic interface in the client so the second platform is an implementation, not a refactor.

**Vault key lifecycle — stated once, precisely, because both 4.6 and 4.7 depend on it:**
- **At rest on device:** only inside the Keychain/Keystore, biometric-gated. Never in app preferences, never in the local SQLite file, never in logs.
- **At rest on the server:** only as ciphertext — two independently wrapped copies (master-password-derived key, recovery key). SecretPass's database never holds the vault key in a usable form.
- **In memory:** the key *is* held in application memory while the vault is unlocked. This is unavoidable — the secure element gates access to the key but does not perform XChaCha20 on the app's behalf, so decryption requires the key in process memory. Any claim otherwise is false and would not survive cryptographic review.
- **On lock:** the key and all decrypted items are zeroed from memory (4.7). Minimise the window it exists in, since that window is what a device-level attacker is trying to hit.
- **Biometrics are a convenience layer over the master password, not a replacement for it.** The master password must still be required:
  - on first setup and on each new device,
  - after device restart,
  - after the auto-lock inactivity period elapses (see 4.7),
  - after a number of failed biometric attempts,
  - before any sensitive action: CSV export, viewing the recovery key, changing the master password, deleting the account.
- Opt-in, and switchable off at any time in settings.
- If the user's enrolled biometrics change (a new fingerprint or face is added to the device), invalidate the stored key and require the master password again — otherwise anyone who can add their own biometric to an unlocked phone inherits vault access.

### 4.7 Auto-lock
- The vault re-locks automatically after a period of inactivity. **Default: 5 minutes.**
- **User-configurable from 1 to 60 minutes**, in settings. The range is capped at both ends: there is no "never" option, because a vault that never re-locks turns a borrowed or stolen unlocked device into full credential access.
- Independent of the timer, the vault locks immediately when: the app is backgrounded, the device is locked, or the device is restarted. These are not user-configurable and are not affected by the inactivity setting.
- "Inactivity" means no interaction with the SecretPass app, not device-wide idle time.
- On re-lock, the decrypted vault and the vault key are cleared from application memory — locking is not merely a UI state change.
- Unlocking after auto-lock accepts biometrics where enrolled (4.6); the master password is still required in the cases 4.6 enumerates.

### 4.8 Contact page
- A public contact page with a form. **The destination email address is never displayed** anywhere in the app, the website, the Terms, or the page source.
- **Must be reachable without signing in.** The people most likely to need support are locked out — expired activation link, mistyped checkout email, failed payment — so a login-gated form would fail exactly the users it exists for.
- Fields: name (optional), reply-to email (required), subject or category, message. Category options should cover billing, activation problems, and general questions.
- **The form must state, visibly, that SecretPass support will never ask for a master password, recovery key, or vault contents.** Support channels for password managers are a standard phishing target; saying this up front on every contact is cheap and sets the expectation that any message asking for those things is fraudulent.
- Confirmation on submit, plus an auto-acknowledgement email, so users know the message arrived and don't resend.
- Spam protection: rate limiting per IP, a honeypot field, and a CAPTCHA only if abuse warrants it.
- Outbound replies come from a support sender address, not from the destination inbox — the destination must not leak via reply headers.

### 4.9 Admin console
A separate web console for operating the service. Ships in v1.

**Access model**
- **Single admin account.** No multi-admin or role hierarchy in v1.
- **The admin is a full vault user as well as an operator.** The admin account has its own vault and access to every user-facing feature — vault CRUD, generator, TOTP, sync, browser extension, biometric unlock, CSV import/export — exactly as a paying customer does. The admin console is an *additional* capability layered on that account, not a replacement for it. The admin's capabilities are a strict superset of a regular user's.
- The admin's vault is entitled permanently and is not tied to a Paddle subscription. Entitlement checks must treat it as always-active without a `subscriptions` row, and it must never enter the past_due/grace/lapsed state machine or the 90-day dormancy deletion job.
- Admin must enroll 2FA before the console becomes usable. The vault side of the account is usable before console 2FA enrollment; the console is not.
- Entering the console requires a fresh 2FA challenge, separate from the app session. An unlocked vault does not imply an open console, and vice versa.
- All admin actions are written to an append-only audit log.

**The admin's own vault does not weaken the zero-knowledge boundary — provided one rule holds**

The admin having a personal vault is safe, and it is safe for exactly the reason every other user's vault is: it is encrypted client-side under the admin's own master password, which the server never receives. The admin can read their own vault for the same reason you can read yours. What must not happen is the console's privileges bleeding into vault access.

Concretely:
- The admin reads their own vault **only through the normal client apps**, decrypting client-side with their own master password — the identical code path every user takes. Never through the console.
- **The console's database role retains zero read access to vault ciphertext columns — including the admin's own rows.** This is unchanged from the original design and remains enforced in Postgres permissions, not application code. The console cannot display a vault, any vault, including its operator's.
- The admin's master password is separate from the console credential and its 2FA. Console access confers no vault access, and the audit log records console actions only — it must never log vault contents or anything derived from them.
- Row-level security applies to the admin's user account like any other: as a vault user, the admin can read only rows where `user_id` matches their own session.

The hard constraint below is therefore untouched. "Admin has a vault" means *the admin has a vault of their own*, not *the admin can open vaults*.

**Bootstrapping the initial admin**

The single-password decision in 4.5 simplifies this, and improves it. Because the authentication credential is now derived from the master password on the device, **there is no server-settable login password for an environment variable to carry** — so the bootstrap no longer needs to handle a password at all.

- On first launch, the app reads **`SECRETPASS_ADMIN_EMAIL`** from the environment and creates an unactivated admin account **only if no admin already exists**. There is no `SECRETPASS_ADMIN_PASSWORD`, and there must not be one.
- Bootstrap marks the account `is_admin` and permanently entitled, then issues a single-use activation token exactly as the Paddle webhook does for a paying customer. The activation link is delivered to that email address.
- The admin activates through the **ordinary customer activation flow**: master password, then the recovery key with its save-confirmation step. One code path, exercised by every account including the operator's — which also means the operator tests it in production before any customer does.
- 2FA enrollment is forced before the console becomes usable. The vault side of the account is usable beforehand.
- Once an admin exists, the bootstrap path permanently disables itself and the environment variable is ignored.
- No default credentials are ever shipped in the repository or the built application.
- **This removes a credential from the environment entirely**, which is a direct security improvement: a bootstrap password would have passed through shell history, deploy logs, and probably a committed `.env` file. An email address in an environment variable is not a secret, and nothing derived from it can open a vault.
- **The admin's own vault is as unrecoverable as anyone else's.** If the operator loses their master password and recovery key, their vault is gone, and the console provides no override — there is nothing in the console that could provide one. Treat the operator's recovery key as production infrastructure and store it accordingly.
- Deployment docs must instruct operators to remove the environment variable after first launch.

**What the admin CAN do**

*As a vault user (through the normal client apps, never through the console):*
- Everything a paying customer can do: vault CRUD, search, folders/tags, password generator, TOTP, sync across devices, browser extension, biometric unlock, CSV import and export — all against their **own** vault only.

*As an operator (through the console):*
- List, search, suspend, and delete user accounts.
- View subscription and billing status; handle failed payments and refunds.
- View service metrics: signups, active devices, sync volume, error rates.
- Trigger support actions: resend verification email, initiate account deletion.
- Review the admin audit log.

**What the admin CANNOT do — hard constraint**
- Read, decrypt, or export **any other user's** vault contents.
- Read any vault — including their own — *through the console*. Vault access happens only in the client apps, under the admin's own master password.
- Reset a user's master password.
- Recover a user's vault under any circumstance, their own included.

This is not a policy choice that can be relaxed later for support convenience. If an admin can decrypt vaults, the zero-knowledge claim in the Terms of Service (4.11) is false and a single compromised admin account becomes a total breach of every customer. Support's only answer to "I lost my master password" is the recovery key (4.10).

Granting the admin a personal vault does not soften this. The admin gains a vault *of their own*, encrypted under a master password the server never sees, on the same terms as every other user — they gain no ability to open anyone else's. The distinction to hold onto: the operator wears two hats on one account, and the console hat can never see vault data.

### 4.10 Account recovery
- At account creation, SecretPass generates a **recovery key** — a high-entropy secret shown to the user once, to be saved offline.
- The recovery key can independently unlock the vault if the master password is forgotten. It is generated and held client-side; SecretPass's servers never receive it.
- If a user loses both the master password and the recovery key, the vault is unrecoverable. There is no support-side exception path — any such path would be a backdoor into every user's vault.
- Setup flow must make the recovery key impossible to skip past accidentally: show it once, require an explicit confirmation that it's been saved.
- **Since 4.5 collapsed the design to a single password, the recovery key is the only reset path that exists at all.** There is no email-based reset that restores vault access, because there is no server-held credential that could authorise one. This raises the stakes on this screen considerably: it is not an optional backup step users may skip and return to, it is the entire recovery story. Design it accordingly — the confirmation must require the user to demonstrate they have the key (re-entering a portion of it, for instance), not merely tick a box asserting they saved it.
- **Out of scope:** trusted-contact / emergency access. Considered and deliberately dropped from v1.

### 4.11 Terms of Service
- A Terms of Service page written in plain, human language — it should explain how the vault works and what the recovery tradeoff means, not read as boilerplate.
- Accessible from within the app at any time (Settings), and at both consent points below.

**Consent is captured twice, because purchase-first splits signup across two moments**

Purchase happens on the website; the vault is created later, at activation. Neither moment alone is sufficient, so both carry a mandatory checkbox:

1. **At web checkout, before payment.** Agreement is required to proceed to Paddle. This is the commercially meaningful consent — it happens before money changes hands, which is where a consumer-law challenge would look.
2. **At activation, before the vault is created.** Agreement is required to complete setup. This is where the unrecoverable-vault warning actually means something, because it is the moment the user sets a master password and is shown a recovery key. At checkout they have no mental model of either yet.

- At both points, an unchecked box blocks progress, with an inline error if the user tries to continue.
- The Terms open in a popup/modal in both flows — the user must not have to leave the flow or open a browser to read them.
- The agreement checkbox text must state the unrecoverable-vault consequence explicitly, not bury it. The activation-time wording should be the more concrete of the two, since the user is holding a recovery key when they read it.
- Record both agreements server-side with a timestamp and the Terms version agreed to. "Which version did they accept, and when" is the only question that matters if this is ever disputed, and it is unanswerable retroactively.
- **Content constraint:** the Terms explain the security model in concept (device-side encryption, zero knowledge, no recoverable master password) but must not publish operational detail useful to an attacker — no algorithm parameters, recovery-key format or length, session/token handling, rate limits, or lockout thresholds. If you want to publish that for transparency, it belongs in a separately reviewed security whitepaper.

### 4.12 CSV import and export
**Import**
- Import vault items from a CSV file (migration from another password manager or a spreadsheet).
- Column-mapping step: the user maps the source file's columns onto SecretPass's fields before anything is written, with an option to skip columns.
- Show a preview and item count before committing; import is all-or-nothing so a bad mapping doesn't half-populate the vault.
- Malformed rows are reported rather than silently dropped.

**Export**
- Export the full vault to CSV.
- Re-authentication (master password) required before an export runs.
- **The UI must clearly warn that a CSV export is unencrypted plaintext** — every password in the file is readable by anyone who opens it. This warning is a requirement, not optional polish: it is the single largest self-inflicted risk the product exposes to users.
- Recommend (and default to) prompting the user to delete the file once migration is complete.
- **Export is also the data-loss backstop for the pre-audit period (5.1.1).** Until the external review lands, export is the only mechanism by which a user whose vault is damaged by a bug can recover their credentials. Treat it as a reliability feature during v1 rather than only a migration convenience, and do not gate it behind anything beyond the master-password re-authentication above.

### 4.13 Account deletion

Required independently by both app stores: an app that supports account creation must offer in-app account deletion, not merely a support request. It is also promised in the Terms.

**This requirement does not depend on shipping to a store.** The GDPR right to erasure applies to SecretPass as data controller regardless of distribution channel or legal form, and it applies from the first EU customer. A web-only v1 with no legal entity carries the identical obligation — absent a company, the controller is the founder personally. Account deletion is therefore not deferrable alongside the store work.

- **Self-service, from within the app.** No support ticket, no email, no contacting anyone — the same standard as cancellation (4.5).
- Requires master-password re-authentication before it runs, per 4.6.
- The confirmation screen must state plainly and before the user commits: the vault is destroyed, SecretPass holds no readable copy and cannot restore it, and this is irreversible in a way that a forgotten password is not.
- **Offer a CSV export from the deletion confirmation screen itself.** Deleting a vault is the single most destructive action in the product, and the user may be deleting the only copy of credentials they migrated in. Do not make them find the export screen first.
- On deletion: remove vault items, folders, sync log, wrapped key copies, and the user record. Cancel any active subscription through Paddle as part of the same flow, so deletion doesn't leave a subscription billing against a vault that no longer exists.
- Deletion is distinct from cancellation (4.5) and from lapsing. Cancelling stops billing and keeps the vault; deleting destroys it. The UI must never let one be mistaken for the other.
- Retain the minimum billing and tax records Paddle requires as merchant of record. These contain no vault data.

### 4.14 Privacy Policy

- A published Privacy Policy, separate from the Terms of Service, covering what is collected, why, how long it's retained, who processes it (Supabase, Paddle, the transactional email provider), and how to request deletion.
- **Required by both app stores before submission** — a missing or unreachable policy is a rejection, regardless of what the Terms say.
- **Also required independently of any store.** GDPR obliges a published policy from the first EU customer, and SecretPass sells globally from launch (6). A web-only v1 with no legal entity still needs one, because the obligation attaches to whoever is the data controller — which, absent a company, is the founder personally. Cost is zero; omission is the cheapest possible compliance failure.
- Reachable without signing in, from the marketing site, the app's settings screen, and both store listings.
- Must state the same thing the Terms do about vault contents: SecretPass cannot read them, so they are not collected in any usable sense. The policy should be consistent with the Terms' Section 7, not a separately drafted account of the same facts.
- Same content constraint as the Terms (4.11): explain the model, don't publish operational security detail.

---

## 5. Non-Functional Requirements

### 5.1 Security (highest priority — see Implementation Plan for technical detail)
- Zero-knowledge architecture: the server stores only encrypted blobs. The master password never leaves the device, and neither does the wrapping key derived from it. The one value derived from the master password that *does* reach the server is the authentication value (4.5), derived under different parameters and a different salt specifically so that it cannot be used to unwrap anything.
- Industry-standard, well-vetted cryptography only — no custom/home-grown crypto primitives.
- Master password is never recoverable by SecretPass; account recovery flows must not create a backdoor into the vault.
- Clipboard contents from copy actions (passwords, TOTP codes) auto-clear after a short timeout.
- All network traffic over TLS; certificate pinning on mobile where feasible.

### 5.1.1 Security testing requirements

**Security testing runs on two tracks, and v1 ships only the first.** The automated and self-directed half costs nothing, runs from Phase 0 onward, and is a hard condition of launch. The paid half — third-party penetration testing and independent cryptographic review — is **deferred to v2** (see 7).

The reason is scale, not principle: the two engagements together cost hundreds of times the monthly running cost of the entire service (Implementation Plan 6.1), and cannot be funded before the product has demonstrated that anyone will pay for it. The trade is recorded in 8.

**The exposure is named rather than left implicit.** v1 ships with no external validation of the cryptographic design — including the split derivation introduced in 4.5, which is new in v1.3 and is the one place the password model touches the zero-knowledge boundary. Three things compensate, and all three are conditions of launch rather than aspirations: every automated control below is free and therefore not deferred at all; the launch surface is narrowed; and the disclosure posture is honest (5.1.2).

**Documentation before testing**
- A written threat model exists before the security review begins, covering: a stolen device, a compromised sync server, a malicious or compromised admin account (including the admin's dual role as a vault user, per 4.9), a hostile network, a malicious CSV import, a compromised browser extension, a compromised support channel, **a compromised web delivery path**, and **a malicious application requesting autofill**.
- **Compromised web delivery needs explicit treatment, because the web app is a primary v1 platform.** A browser-delivered client re-downloads its cryptographic code on every load, so whoever controls the hosting or the deploy pipeline can serve a modified build that exfiltrates the master password — and no amount of client-side correctness prevents it. This is the standard and legitimate criticism of browser-based password managers, and reviewers will raise it. Mitigate with a strict Content Security Policy, Subresource Integrity on every asset, and tightly controlled deploy access — then state the residual risk plainly in the cryptographic design document rather than implying the web app is equivalent to a signed, store-reviewed binary. It is not, and claiming otherwise will not survive review.
- A cryptographic design document describing key derivation, **the separate derivation of the server-facing authentication value (4.5)**, key wrapping, the recovery-key path, and the sync protocol — the artifact a reviewer reads first. The auth-value derivation deserves its own section: it is the one output of the master password that reaches the server, and a reviewer's first question will be whether it can be used to unwrap anything.

**Automated testing, running continuously in CI**
- **Static analysis (SAST)** on every commit for the Flutter app, the extension, and backend functions.
- **Dependency and supply-chain scanning (SCA)** with builds failing on known-vulnerable packages. A password manager inherits the security of everything it imports.
- **Secret scanning** on every commit and across git history, to catch committed keys, tokens, or the admin bootstrap credentials.
- **Cryptographic test vectors** shared between the Flutter app and the extension: both must produce and consume identical ciphertext for the same inputs. A cross-implementation mismatch is a data-loss bug, not just a security one.
- **Authorization tests** proving Supabase row-level security actually isolates users — that user A cannot read user B's rows through any API path, including direct PostgREST calls.
- **Admin-boundary tests** proving the admin console's database role cannot read vault ciphertext. This must be an automated test that fails the build, not a code review convention. Because the admin is also a vault user (4.9), these tests must additionally prove the dual role does not widen access: an admin-authenticated session reads its own vault rows and no one else's, and the console role reads no ciphertext at all.

**Self-directed manual testing before launch — the v1 gate. All of it is free, and none of it is deferred.**
- **Mobile application security self-testing** against the OWASP MASVS/MASTG checklist, if and when Android ships — local storage, Keystore usage, biometric binding and key invalidation, screenshot and background-snapshot leakage, clipboard behaviour, logging, and anti-tampering basics. The checklist is public and self-application is free; only the formal third-party assessment is deferred.
- **Autofill target validation, on both implementations** (4.3): that the browser extension will not fill into a look-alike domain, and that the Android service will not fill into an application whose signing certificate fails Digital Asset Links verification against the target domain. **Test this with a deliberately built look-alike app and a deliberately registered look-alike domain, not by code inspection.** Across both platforms this is the highest-severity defect class the product can ship.
- **Autofill lock-boundary testing** (4.3.3): confirm that an autofill request against a locked vault prompts for authentication, that the key is not retained after the request is serviced, and that servicing a fill does not extend the auto-lock inactivity timer.
- **Web and API testing** against OWASP ASVS, at the level appropriate for an application handling credentials.
- **Browser extension review**: content-script isolation, message-passing between the page and the extension, autofill target validation (the extension must not fill credentials into a look-alike domain), and permission scope.
- **Business-logic testing** of the paths where money and access meet: refund abuse, entitlement bypass, activation-token reuse, subscription-state race conditions.

**Release gate (v1)**
- No open critical or high findings **from the automated and self-directed testing above** at launch. Medium findings are documented with an owner and a target date.
- Test reports and remediation evidence are retained, self-directed testing included — they become the starting point for the v2 external review rather than something rewritten for it.
- Re-testing by the original tester belongs to the third-party engagements and moves to v2 with them. Self-directed findings are re-tested by re-running the automated suite, which is precisely why the automated controls are a hard gate and not a convenience.

**After launch (v1)**
- Dependency and secret scanning continue on every commit.
- A published **responsible disclosure policy** with a security contact, before launch. Free, and it matters more without a penetration test rather than less: in v1, researchers are the external review. Give them a route that isn't the support form.
- A written incident response plan covering who is notified, how users are told, and the disclosure timeline. Writing it during an incident is too late, and an unaudited product should assume it is likelier to need one.

**Deferred to v2 (see 7) — the paid engagements**
- **Third-party penetration test** covering the sync API, authentication and session handling (including the client-side derivation of the server-facing auth value, 4.5), the activation and recovery-key flows, the payment webhook path, the admin console, the browser extension, and the Android autofill service.
- **Independent cryptographic review** of the encryption design and its implementation, by someone who did not write it. Point the reviewer at the split derivation in Implementation Plan 2.1 first.
- **Formal OWASP MASVS/MASTG assessment** of the Android build.
- **Re-test by the original tester** of every fix arising from the above.
- Thereafter, penetration test repeated at least annually and after any change to the cryptographic design, the sync protocol, or the admin console.

**The trigger for v2 is not a date.** Whichever comes first: revenue that covers the engagements (Implementation Plan 6.1 puts this at roughly 245 subscribers), or distribution to users who did not knowingly opt into an unaudited product — which in practice means a public app-store listing.

### 5.1.2 Disclosure posture — what v1 may and may not claim

v1 ships without third-party validation (5.1.1). That is a legitimate position for a trial launch, and it remains legitimate only for as long as nothing SecretPass publishes implies otherwise. This is the cheapest requirement in this document and the one whose breach carries the most direct legal exposure.

- **Never claim, in any channel, that SecretPass has been "independently audited", "penetration tested", "security reviewed", or "third-party verified"** until it has been. This covers the marketing site, the store listings, the Terms, the Privacy Policy, in-app copy, and all advertising and social creative.
- Describing the *design* accurately is always permitted and encouraged — zero-knowledge architecture, client-side encryption, Argon2id, XChaCha20-Poly1305, the two-level key hierarchy. What may not be implied is external validation of that design.
- **State the pre-audit status plainly** to anyone signing up during the trial phase, at the same point the Terms consent is captured (4.11). A user who knows they are an early adopter of an unaudited security product and chose it anyway is in a completely different position, ethically and legally, from one who assumed otherwise.
- The residual-risk disclosure already required for the web delivery path (5.1.1) is the model for tone: state the limitation, do not soften it.
- When the v2 engagements complete these claims become available — and the cryptographic design document written for v1 is what makes them cheap to substantiate.

### 5.2 Performance

Unlock has two distinct paths with very different costs, and holding both to one number would force the key derivation to be weakened to hit a UX target. They are budgeted separately:

- **Biometric unlock: under 2 seconds** from cold start on a mid-range device. This path unwraps the vault key from the Keychain/Keystore and runs no key derivation, so it is the fast path and the one users hit most often.
- **Master-password unlock: under 4 seconds** from cold start on a mid-range device, of which the Argon2id derivation is the dominant cost. This is deliberately looser. Argon2id is slow *by design* — that slowness is the defence against offline brute-forcing of a stolen vault, so trading it away for a faster unlock trades away the protection it exists to provide.
- **Argon2id parameters are chosen on security grounds first**, then measured. If the measured unlock time exceeds this budget on target hardware, revisit the budget before revisiting the parameters, and never tune the KDF down purely to hit a performance number. Record the chosen parameters and measured times in the cryptographic design document (5.1.1).
- Search across a vault of 500+ items returns results in under 200ms. Search runs in memory against the already-decrypted vault (items are stored as single encrypted blobs and are not SQL-queryable by field), so this budget applies to filtering, not decryption.

### 5.3 Availability & sync
- Sync backend targets high availability; local-first design means the app remains usable during backend outages, with sync resuming automatically after.

### 5.4 Usability
- Biometric unlock is in scope for v1 — see 4.6.
- Auto-lock defaults to 5 minutes and is user-configurable from 1 to 60 minutes — see 4.7. Biometrics are what keep a short default from being punishing: the security setting and the convenience feature have to be designed as a pair, not separately.

---

## 6. Monetization

- **Model:** Paid subscription only. A single tier — there is no unpaid version of SecretPass.
- **Pricing:** $5/month, or $50/year.
- **What's included:** everything — vault, password generator, multi-device sync, browser extension, TOTP storage. There is no feature gating between tiers because there is only one tier.
- **Markets:** global from launch.
- **Rail:** Paddle web checkout only, on all platforms. No in-app purchase.
- **Guarantee:** 30-day money-back, in place of a trial.
- **Provisioning:** purchase-first — the account is created by the payment webhook, not by a signup form.
- **Entitlement:** one subscription covers every platform. Because all purchasing happens on the website through Paddle, entitlement is a single boolean on the user record that all clients read — there are no store rails to reconcile and no cross-rail entitlement problem to solve.

### 6.1 Acquisition

The mobile apps cannot sell or link to a subscription (see 4.5), and that constraint is now permanent. The app store is therefore never the top of the funnel — it is where an already-paying user goes to install the client. Everything below exists to make the website the first thing a prospective customer encounters.

**Implementation constraint the SEO channel depends on:** the marketing site must be a **separate, statically rendered site — not a page inside the Flutter web build.** Flutter web renders into a canvas that search engines cannot meaningfully index, and it carries a multi-megabyte first load. Building the marketing site as ordinary HTML is what makes the SEO channel possible at all, and it has a second benefit: the checkout page, the Terms, the Privacy Policy, the contact form (4.8), and the activation and resend pages all stay reachable without loading the application. That matters more than it sounds, because the users who most need those pages are precisely the ones who cannot sign in.

All four of the following are in the launch plan:

- **Search / SEO on the marketing site.** Own the search intent this product is bought on: password manager comparisons, and migration guides ("moving off LastPass/1Password/browser-saved passwords"). Costs nothing per visitor and compounds, but takes months to rank — so it starts before launch, not after.
- **Paid acquisition to a checkout landing page.** Search and social ads pointing directly at the Paddle checkout, bypassing the stores entirely. This is the only channel that gives a fast read on whether $5/month actually converts, which is worth paying for early even at negative margin per install.
- **Migration-targeted positioning.** Users already leaving another password manager are the highest-intent segment and they search the web rather than browse the app store, so they land on the site by default. CSV import (4.12) is the feature that closes them and should be prominent in the marketing copy, not buried as a settings-screen utility.
- **App store listing as a redirect, not a dead end.** No purchase link may appear in the app *binary*, but the store listing description is ours: it must state plainly that accounts are created on the SecretPass website. This does not recover users who never look at the listing, but it stops the silent drop-off from people who found the app first, installed it, and hit a sign-in screen for an account they have no way to create.

With iOS deferred (see 2), three of these four channels are unaffected — they are all web-side. Only the store-listing redirect narrows, to Google Play alone, and it is the weakest of the four regardless.

---

## 7. Out of Scope for v1 (explicitly deferred)

- **iOS app — deferred, not cancelled** (see 2). Requires macOS to build at all, adds the strictest of the three store reviews to the launch path, and duplicates the biometric and autofill work in a second native language. Ships after the audited core is live.
- Firefox and Safari browser extensions — Chrome only at launch (see 2).
- Secure notes and payment card storage.
- Family/team vault sharing.
- Breach monitoring / dark-web alerts.
- Native desktop apps (Windows/macOS/Linux) — web app covers this need for v1.
- Emergency access via trusted contacts (considered during planning, dropped — recovery key is the sole backup path).
- Passkey storage and Android Credential Manager — 4.3.2 requires only that the autofill implementation not foreclose it.

**Deferred from the launch gate rather than from the product.** The four items below were conditions of launch in v1.3 and are now v2 (see 8 for the reasoning). They are recorded here so that "deferred" stays visible rather than becoming "forgotten":

- **Third-party penetration test** (5.1.1). Trigger: revenue coverage or public store distribution, whichever comes first.
- **Independent cryptographic review** (5.1.1). The highest-leverage item on this list — deferred because it is unaffordable pre-revenue, never because it is optional. It is the first thing bought when v2 triggers.
- **Formal OWASP MASVS/MASTG assessment** (5.1.1), with the other paid engagements.
- **Legal entity formation, D-U-N-S, and organization store registration** (Implementation Plan 6.3). This has direct consequences for Google Play distribution and for personal liability; both are recorded there.

---

## 8. Resolved Decisions

The questions previously open in this section have been decided. Recorded here with their outcomes so the reasoning isn't lost:

| Question | Decision | Where it lives now |
|---|---|---|
| Auto-lock timeout and configurability | 5-minute default, user-configurable 1–60 minutes, no "never" option | 4.7 |
| Master password separate from account login? | **Reversed — one password only.** See the note below | 4.5, Implementation Plan 2.1 |
| Exact pricing for the "Premium tier" | Void — there is no Premium tier. Single tier at $5/month or $50/year | 4.5, 6 |
| Region-gated external purchase links on the US storefront | **Dropped permanently**, not deferred. Mobile builds stay identical and billing-free in every market | 4.5 |
| Driving users to the website rather than the app stores | Four committed channels: SEO, paid acquisition, migration positioning, and store-listing redirect | 6.1 |
| Legal review of the Terms of Service | Launching on the current plain-language copy; lawyer review scheduled post-launch | Below |
| **v1 platform scope** | **iOS deferred post-v1.** v1 is Web, Android, and the Chrome extension | 2, 7 |
| **Mobile autofill** | **In v1, Android only.** Native `AutofillService` with Digital Asset Links verification | 4.3.2 |
| **Autofill vs. auto-lock conflict** | Auto-lock wins; autofill authenticates per request against a locked vault and never extends the timer | 4.3.3 |
| **Store account registration** | **Organization**, not individual — requires a legal entity and a D-U-N-S number | Implementation Plan 6.3 |
| **Store review account** | One permanently entitled account provisioned for reviewers; a second documented exception to purchase-first | 4.5 |
| **Third-party pentest and cryptographic review** | **Deferred to v2.** v1 launches on the free automated and self-directed testing only | 5.1.1, 7 |
| **Legal entity formation** | **Deferred to v2.** v1 operates without one; consequences recorded in Implementation Plan 6.3 | 7 |
| **Claims about security validation** | Prohibited until the v2 engagements complete | 5.1.2 |

**Password model — a reversal, recorded because the earlier decision was marked settled.** Version 1.2 specified a separate account login password alongside the master password, on the reasoning that separation made a forgotten login a routine reset rather than vault loss. That reasoning was sound as far as it went, but it overstated the case: one password is fully compatible with zero knowledge, provided the server-facing authentication value is derived client-side under different parameters and never equals the wrapping key. This is what the established products in this category do, and the two-level key hierarchy in Implementation Plan 2.1 already supports it without modification.

Two things decided it. First, activation would otherwise ask a user who has **already paid** to create and distinguish three secrets in one sitting, at the highest-drop-off point in the funnel — and with a 30-day no-questions automatic refund (4.5), onboarding friction converts straight into refunds with no human in the loop to recover the sale. Second, and more seriously, nothing prevents a user from setting both passwords to the same string, and a meaningful share would: at that moment the credential transmitted to the server on every sign-in *is* the vault-unlocking secret, and the zero-knowledge property is broken by the interface working exactly as specified. A client-side equality check catches the identical case and nothing adjacent to it. Collapsing to one password removes the failure mode by construction rather than by validation.

The cost is accepted and named: there is no email-based password reset that restores vault access, so the recovery key (4.10) is the sole recovery path. This was already true of the master password before the change — what is gone is a safety net that only ever protected the less consequential of the two secrets.

**Legal review — decision and its consequences.** SecretPass will launch on the existing plain-language Terms, with a lawyer engaged after launch rather than before. This is a deliberate, informed trade of legal exposure for launch speed, and the exposure should be named rather than left implicit: the current copy has no jurisdiction or governing-law clause, has not been checked against consumer-protection law in the markets it sells into, does not carry the subscription and auto-renewal disclosures that several regions require, and lacks the app-store-specific terms Apple and Google expect. The Terms document's closing note lists the sections most affected (4, 5, 8, 9).

Two mitigations make this materially cheaper than it sounds, and both are already in the design: Paddle is merchant of record, which puts statutory withdrawal rights, tax, and much consumer-law compliance on Paddle rather than on SecretPass; and the 30-day automatic refund is more generous than most jurisdictions require, so the commercial terms are unlikely to be the thing that draws a complaint. Book the review early enough that any required changes land as a routine Terms update under Section 10, rather than as a response to a complaint.

**Launching without external security validation — the largest trade in this document.** v1.3 made the third-party penetration test and independent cryptographic review conditions of launch (5.1.1). They are now deferred to v2, and the reasoning deserves recording as carefully as the reasoning that originally put them there.

The arithmetic decided it. The two engagements cost roughly $13,000–35,000 against a running cost of about $26 per month (Implementation Plan 6.1), and the annual pentest repeat alone needs roughly 245 subscribers to fund from revenue. Sequencing an unfunded $13,000–35,000 gate ahead of any evidence that the product sells means the most likely outcome is that the review is never paid for at all — because the product never launches. Testing demand first and buying validation out of revenue is the only sequence in which the review actually happens.

What makes this defensible rather than merely cheap is that **the deferral is bounded on three sides**: the free controls in 5.1.1 are not deferred and remain a hard gate; 5.1.2 forbids claiming validation that has not happened; and 7 fixes a trigger — revenue coverage or public store distribution, whichever comes first — so v2 is a commitment rather than an intention.

What is genuinely given up, stated plainly: the split derivation in 4.5 is new in this version, it is the one place the password model touches the zero-knowledge boundary, and a mistake in it would be invisible to every test v1 runs. No automated control catches a design flaw of that shape. That is the residual risk, nothing in the v1 gate mitigates it, and it is why the cryptographic review is named first in the v2 list rather than sorted among the others.

## 9. Still Open

- **Will Paddle onboard the founder as an individual or sole trader?** With entity formation deferred (7), this is the one unanswered question that can invalidate a design rather than adjust it — Phase 4 assumes Paddle for provisioning, entitlement, refunds and tax, and this plan carries no fallback processor. Confirm in Phase 0, before Phase 4 work begins.
- **Does v1 ship to Google Play, or stay web-only?** Without an entity, a Play listing triggers the 12-tester/14-day closed-testing gate and publishes the founder's legal name and address (Implementation Plan 6.3). Web-only removes both, and removes Phase 2.6 with them. Not blocking until Phase 5e.
- New questions get added here as they arise.
