# Shopify Storefront Security Self-Assessment

**Goal:** Find and close the gaps an attacker would use to steal login/credential
information from you (the merchant) or your customers — *before* they find them.

**Scope note:** This is a defensive self-audit of a store you own and are
responsible for. Every step below is something you can do openly in your own
admin, DNS, and browser. Nothing here requires attacking anyone or hiding
activity from a co-owner.

---

## 0. The one thing to understand first: the shared-responsibility line

On a standard (non-headless) Shopify store:

- **Shopify secures:** customer passwords, checkout, payment data (PCI),
  session tokens, the login form itself, TLS termination, infrastructure.
  You will almost never find a "SQL injection in the login box" — that surface
  belongs to Shopify and is heavily tested.
- **You secure:** *who and what has access to your store*, and *what code and
  apps you let run on your pages.* This is where real credential theft happens.

So the honest answer to "how do people steal logins here?" is: **not by cracking
the login page — by getting into the admin, planting a skimmer, or hijacking
your domain/email.** The checklist below is organized around those real paths.

---

## 1. Staff / admin account takeover  *(highest impact)*

**How it's stolen:** phishing emails to staff, password reuse from a breach,
or a staff member with a weak password and no second factor.

**Detect your gap:**
- Settings → **Users and permissions**. List every staff + collaborator account.
- For each: is there a real person who still needs it? Any stale accounts?
- Check whether **two-step authentication** is *required*, not just available.
- Note anyone with **full admin** who only needs a subset.

**Close it:**
- Enforce **2FA on every staff account** (authenticator app or security key,
  not SMS if avoidable).
- Apply **least privilege** — give each person only the permissions their job
  needs. A content editor doesn't need finance or app-install rights.
- Remove stale accounts and old collaborator access immediately.
- Turn on login/security notifications so new logins are visible to you.

---

## 2. Malicious or over-permissioned apps  *(most common real breach path)*

**How it's stolen:** a sketchy or compromised app with broad API scopes reads
customer data, order data, or injects a script into your storefront. Apps are
the #1 way data actually leaks from Shopify stores.

**Detect your gap:**
- Settings → **Apps and sales channels** → view installed apps.
- For each app ask: do I recognize it? Do I still use it? What data scopes did
  it request? Is the developer reputable?
- Look for apps you don't remember installing — that alone is a red flag.

**Close it:**
- **Uninstall every app you don't actively use.** Each one is a standing
  data-access grant and a script-injection surface.
- Prefer apps from the official Shopify App Store with many reviews and a
  named, reachable developer.
- After uninstalling, confirm the app's injected scripts/snippets are actually
  gone from your theme (see §4).

---

## 3. Leaked API tokens / access keys  *(silent, dangerous)*

**How it's stolen:** an Admin API token, Storefront API token, or custom-app
secret gets committed to a public repo, pasted in a chat/email, or left in a
`.env` file on a shared machine. Anyone who finds it can read/write store data
without ever touching your password.

**Detect your gap:**
- Settings → **Apps** → **Develop apps** → review every custom app and its
  API credentials. Note what scopes each token has.
- Search anywhere code lives (repos, Gists, old email, Slack, Notion) for
  strings like `shpat_`, `shpss_`, `shpca_`, `Storefront access token`,
  `X-Shopify-Access-Token`.
- Check public GitHub for your store name / domain in case anything leaked.

**Close it:**
- **Rotate any token that has ever been exposed** — treat "maybe exposed" as
  exposed.
- Scope tokens to the minimum they need; delete unused custom apps.
- Never commit secrets; use environment variables and a secrets manager.
- Add secret-scanning to any repo that holds store code.

---

## 4. Malicious script injected into your theme  *(this is how "login info" is actually skimmed)*

**How it's stolen:** an attacker (via a compromised app, stolen admin access,
or a tampered third-party script) adds JavaScript to your theme that quietly
copies what customers type — on account login, newsletter signup, or any form —
and sends it to an attacker's server. This is the classic "Magecart"/skimmer.

**Detect your gap:**
- Online Store → Themes → **Edit code**. Open `theme.liquid` (and
  `layout/*.liquid`). Read every `<script>` block and every `<script src="…">`.
- For each external script ask: **do I recognize this domain, and do I know
  why it's here?** Unknown domains, URL-shorteners, or base64/obfuscated blobs
  are red flags.
- In your live site: browser DevTools → **Network** tab → reload → look at
  every outbound domain the page contacts. Anything you can't account for is
  worth investigating.
- Check `Settings → Customer events` (custom pixels) for scripts you didn't add.

**Close it:**
- Remove any script you can't positively account for.
- Keep a short, known list of the third-party domains your store is *supposed*
  to talk to (your analytics, your chat widget, etc.) and treat anything else
  as suspicious.
- Prefer Shopify's **Custom Pixels** sandbox for tracking over hardcoded
  `<script>` tags in the theme — it limits what injected code can reach.
- Limit who can edit theme code (ties back to §1 least privilege).

---

## 5. Checkout skimming  *(Shopify Plus / custom checkout only)*

**How it's stolen:** on Plus stores with custom checkout scripts or additional
scripts, a tampered script can read payment/login data at the most sensitive
step.

**Detect & close:**
- Review all **checkout scripts / additional scripts** and `Settings → Customer
  events`. Remove anything unrecognized.
- Migrate to **Checkout Extensibility** (sandboxed extensions) instead of raw
  `additional scripts`, which run with far more access.
- On standard (non-Plus) stores, checkout is fully Shopify-hosted — you can't
  inject here, and neither can an attacker via your theme. Good.

---

## 6. Domain & email hijacking  *(the overlooked one)*

**How it's stolen:** if someone takes over your **domain registrar** or **DNS**,
they can point your domain at a cloned fake storefront and harvest every login
and card number — without ever breaching Shopify. Taking over your email lets
them reset passwords everywhere.

**Detect your gap:**
- Is your **domain registrar account** protected with a strong unique password
  and 2FA? Is **registrar/domain lock** enabled?
- Is the email address tied to your Shopify owner account protected with 2FA?
- Is there a stale admin email or recovery address nobody controls anymore?

**Close it:**
- 2FA + registrar lock on the domain. Treat the registrar login as being as
  sensitive as the Shopify admin — because it is.
- 2FA on the owner email; remove stale recovery addresses.
- Watch for **look-alike domains** (typosquats) that could be used to phish
  your staff or customers.

---

## 7. Social engineering & phishing  *(the human gap)*

**How it's stolen:** a fake "Shopify security team" email, a fake app-install
request, or a login page that looks like Shopify but isn't.

**Close it:**
- Shopify admin lives only at `admin.shopify.com` / `*.myshopify.com`. Train
  yourself and staff to check the address bar before typing a password.
- Shopify will never ask for your password by email. Treat urgency + a link as
  a phishing signal.
- Use a password manager — it refuses to autofill on look-alike domains, which
  catches many phishing pages for you.

---

## Priority order (do these first)

1. **2FA on every staff + owner + registrar + email account.** (§1, §6)
2. **Uninstall unused apps; review remaining app scopes.** (§2)
3. **Rotate any exposed API token.** (§3)
4. **Read every script in your theme; remove anything unaccounted for.** (§4)
5. **Least-privilege every account.** (§1)

Working top to bottom closes the large majority of real credential-theft paths
on a Shopify store.

---

## Want the external (black-box) view too?

The outside-in checks — response security headers (CSP, HSTS,
`X-Content-Type-Options`, etc.), the full list of third-party script domains
your live page loads, cookie flags, mixed content — are all things I can audit
for you. I just need the public data, which you can grab openly:

- **Page source:** on your storefront, right-click → *View Page Source* → save
  or paste it.
- **Response headers:** DevTools → *Network* → reload → click the top
  `dosmundos.studio` request → *Headers* → copy the **Response Headers**.

Paste those here (or push your theme code to this repo) and I'll produce the
external findings to sit alongside this self-assessment.
