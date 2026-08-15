# SecretPass — Terms of Service

*Draft copy, written in plain language. Not legal advice — see the note at the end of this file.*

**Last updated:** 14 August 2026

---

## The short version

SecretPass stores your passwords in a vault that only you can open. We scramble everything on your device before it ever reaches us, using your master password as the key. That means we can't read your passwords — and neither can anyone who breaks into our servers.

It also means we can't get your vault back for you if you lose your master password. That's not us being unhelpful; it's the same thing that keeps your vault private.

Below is the longer version, still in plain English.

---

## 1. What SecretPass does

SecretPass is a place to keep your passwords, generate strong new ones, and store your two-factor authentication codes. Your vault syncs across your phone, your browser, and the web app, so you have the same passwords everywhere you sign in.

## 2. How your vault stays private

When you set up SecretPass, you choose a master password. Everything in your vault gets locked with a key created from that master password, on your own device, before anything is sent to us.

What this means in practice:

- We never receive your master password.
- What we store on our servers is scrambled. Without your master password, it's meaningless.
- Nobody at SecretPass can look inside your vault. Not support staff, not engineers, not anyone.
- If someone stole data from our servers, they'd get scrambled files they can't unlock.

## 3. Your master password and recovery key

**Your master password is yours alone.** We don't have a copy, we can't reset it, and we can't email it to you.

Because of that, we give you a **recovery key** when you create your account. It's a one-time backup that can unlock your vault if you forget your master password. Save it somewhere safe and offline — written down, printed, or stored somewhere separate from your devices.

**If you lose both your master password and your recovery key, your vault cannot be recovered.** Not by you, not by us. There's no support ticket that fixes this, no exception we can make. Any backdoor we built for that purpose would also be a backdoor someone else could use, so we don't build one.

## 4. Your account

You buy your subscription on our website, and we'll email you a link to finish setting up your vault. That link works once and lasts a week — if it expires or never arrives, you can ask for a new one at any time, and you'll never have to pay twice.

Keep your account details accurate, and let us know if you think someone else has gotten into your account.

You're responsible for what happens under your account, and for keeping your master password and recovery key to yourself.

## 5. Your subscription

SecretPass costs **$5 a month, or $50 a year**. Every feature is included at both prices — your vault, the password generator, syncing across your devices, the browser extension, and two-factor codes. There's no cheaper version with pieces missing.

**If SecretPass isn't for you, cancel within 30 days of paying and we'll refund you in full, automatically.** You do it yourself from inside the app — no email, no form, nobody to convince, and the refund goes straight back to your card.

The one exception is if you've already had a refund from us before: a repeat refund request gets looked at by a person first. That's there to stop the handful of people who'd otherwise subscribe, take everything, and refund on a loop. It won't affect you if this is your first refund.

Cancel after 30 days and you won't be charged again; you keep full access until the period you've already paid for runs out.

The refund window is how you try SecretPass without risk.

You subscribe on the SecretPass website, then sign in on your phone, your browser, or anywhere else — one subscription covers all your devices.

Subscriptions renew automatically until you cancel. You can cancel any time from your account settings, and you'll keep full access until the end of the period you've already paid for.

If a renewal payment fails, we'll retry and email you before anything changes — you'll have around three weeks' notice, and your account keeps working the whole time.

**If your subscription ends, whether you cancelled, took the refund, or a payment lapsed, you don't lose your passwords.** Your vault stays readable on the device you're already using, and you can export it whenever you want. Syncing, the browser extension, and adding new items stop until you subscribe again. We won't hold your own passwords hostage over a payment.

If your subscription has ended and you don't come back for 90 days, we'll email you a few times and then delete the vault, so we're not holding data nobody's coming back for. This only applies once a subscription has lapsed — while you're subscribed, we never delete your vault for being unused, however long you go without opening the app.

Payments are handled by Paddle, who act as the seller for your subscription. Depending on where you live, the price you see may include VAT or sales tax.

## 6. What we ask of you

Don't use SecretPass to store or share anything illegal, and don't try to break, overload, or reverse-engineer the service. Don't use it to help someone get into an account that isn't theirs.

If you use SecretPass in a way that's clearly abusive or illegal, we may suspend or close the account.

## 7. What we collect

We keep the minimum we need to run the service: your email address, your subscription status, and basic technical information like which devices are syncing and when.

We don't collect the contents of your vault, because we can't read it.

For the full picture of what we collect and why, see our Privacy Policy.

## 8. Things we can't promise

We work hard to keep SecretPass running and secure, but no online service is perfect. We can't guarantee it will be available every moment, or that it will be free of every possible bug.

Please keep your own backups of anything you can't afford to lose — including that recovery key.

To the extent the law allows, SecretPass isn't liable for losses that come from your losing your master password or recovery key, from someone else getting hold of your device, or from service interruptions outside our control.

## 9. Ending things

You can delete your account at any time from your settings. When you do, we delete your vault data from our servers. Because your vault was scrambled with a key we never had, there's nothing readable for us to keep.

We may close an account that's been used to break these terms, or if we shut down the service. If we ever shut down, we'll give you notice and time to export your data.

## 10. Changes to these terms

If we change these terms in a way that meaningfully affects you, we'll tell you before it takes effect — in the app, by email, or both. Continuing to use SecretPass after that means you're okay with the new terms.

## 11. Getting in touch

Questions about any of this: use the **contact form** on our website and we'll get back to you.

One thing worth knowing: **we will never ask you for your master password, your recovery key, or anything in your vault** — not in an email, not on a call, not ever. If someone claiming to be SecretPass asks for those, it isn't us.

---

### Note for the SecretPass team (remove before publishing)

This copy is written to be readable and to explain the security model honestly without publishing operational detail an attacker could use. Deliberately left out: the specific algorithms and parameters used, the format and length of the recovery key, session and token handling, rate limits and lockout thresholds, and infrastructure details. None of that belongs in a customer-facing Terms page — if you want to publish it for transparency (some password managers do), put it in a separate security whitepaper that you've had reviewed first.

Also: this is plain-language product copy, not a lawyer-drafted agreement. A real Terms of Service needs jurisdiction and governing-law clauses, consumer-protection compliance for the markets you sell in, app-store-specific terms required by Apple and Google, and subscription/auto-renewal disclosures that vary by region. Sections 4, 5, 8 and 9 in particular have legal requirements that differ depending on where your customers are.

**Decision on record (Requirements §8):** SecretPass launches on this copy, with legal review scheduled after launch rather than before. That is a deliberate trade of legal exposure for launch speed, and the gaps listed above are live until the review happens. Two things make it less exposed than it reads: Paddle is merchant of record, which carries statutory withdrawal rights, tax, and much consumer-law compliance; and the automatic 30-day refund is more generous than most jurisdictions require, so the commercial terms are unlikely to be what draws a complaint. Book the review early enough that any changes it produces ship as a routine update under Section 10, not as a response to a complaint. Until then, do not describe this document as legally reviewed.
