# External Attack Surface — Your Store Through an Outsider's Eyes

**Goal:** See exactly what an attacker sees during *passive reconnaissance* of
`dosmundos.studio`, and know which of those findings are the weak spots that
make an attacker decide you're worth attacking — so you can close them first.

Everything here is **public information** an outsider can read without touching
anything private. The point is defensive: look at yourself the way they do, then
remove the soft targets.

---

## How an attacker triages a store (the recon funnel)

1. Fingerprint the platform → decide the strategy.
2. Read security headers → find out if injected code could pay off.
3. Enumerate third-party scripts → find a weaker supplier to compromise.
4. Check DNS/email records → find out if they can phish your people.
5. Hunt for leaked secrets → look for a free key to the admin.
6. Note the human/brand surface → prep social engineering.

They spend ~minutes on this. If it's all hard, most move on. Your job is to
make every rung of that funnel a dead end.

---

## 1. Platform fingerprint — "what am I even attacking?"

**What they see:** response headers and page source reveal Shopify instantly
(`cdn.shopify.com` assets, `Shopify.theme` / `Shopify.shop` JS globals,
`x-shopify-*` headers, the `/products.json` endpoint, a `*.myshopify.com`
origin).

**What it tells them:** "Customer login and checkout are Shopify-hosted and
hard. Don't bother with the login form — pivot to the theme, the apps, the
domain, or the humans." So a Shopify fingerprint isn't itself a weakness — it
just *routes the attacker to items 2–6.* You can't hide it, and that's fine.

**Self-check:** View Page Source; search for `cdn.shopify.com` and `Shopify.`.

---

## 2. Security response headers — THE big external tell  ⚠️ weak-spot zone

These ship with every page and an attacker reads them in seconds. They reveal
how much damage injected or malicious code could do.

| Header | What its absence/weakness means to an attacker |
|---|---|
| **Content-Security-Policy (CSP)** | The big one. A missing or loose CSP means *if* they get a script onto your page (via a compromised app/supplier), it can exfiltrate data to any server they like. A tight CSP makes a skimmer far harder to cash in. |
| **Strict-Transport-Security (HSTS)** | Missing → room for downgrade / man-in-the-middle on untrusted networks (cafe wifi), where creds can be captured in transit. |
| **X-Frame-Options / frame-ancestors** | Missing → your pages can be framed for clickjacking (tricking a logged-in user into actions). |
| **X-Content-Type-Options: nosniff** | Missing → MIME-sniffing tricks that can turn an uploaded/served file into executable script. |
| **Referrer-Policy / Permissions-Policy** | Loose values leak URLs/referrers or allow unneeeded browser features. Lower severity, but tells them you haven't hardened. |

**Why this matters most externally:** a missing/weak CSP is the single finding
that turns "they planted a script" (item 3/§4 of the self-assessment) into
"they successfully stole logins." It's the difference between a gap and a breach.

**Self-check:** DevTools → **Network** → reload → click the top
`dosmundos.studio` request → **Headers** → read **Response Headers**. Note which
of the above are present and what CSP (if any) allows. *Paste these to me and
I'll grade them precisely.*

---

## 3. Third-party scripts — the supply-chain map  ⚠️ weak-spot zone

**What they see:** every `<script src="…">` domain in your page source. Each is
a company whose compromise = your compromise. Attackers specifically look for:

- A script host that's small, abandoned, or on an expired/typosquattable domain
  (they take it over, and your store serves their code).
- An analytics/widget vendor with a sloppy origin they can poison.
- More third-party domains than a store needs — each one widens the surface.

**Weak spot:** every external domain you can't justify. The fewer, the better.

**Self-check:** In page source, list every distinct `src` domain. In DevTools →
Network, watch every outbound domain on load. Anything you don't recognize is a
lead. *Paste the page source and I'll enumerate the domains and flag the odd
ones.*

---

## 4. DNS & email records — how they'd phish you  ⚠️⚠️ biggest overlooked weak spot

This is the recon step store owners never think about, and attackers always run.
It's all public DNS, readable by anyone.

**What they check and why:**
- **SPF / DKIM / DMARC records.** If your domain has **no DMARC** (or a weak
  `p=none` policy), an attacker can send email that appears to come from
  `@dosmundos.studio` — to your own staff, your suppliers, or your customers.
  That's the launchpad for the admin-takeover and phishing paths. **A missing
  DMARC record is one of the most attractive external weak spots there is,
  because it costs the attacker nothing.**
- **Subdomains.** `dev.`, `staging.`, `old.`, `test.` — stale subdomains
  pointing at non-Shopify servers are often forgotten and unpatched, and can be
  a soft way in or a subdomain-takeover target.
- **Registrar/DNS exposure.** They note your registrar and nameservers; a
  registrar account without 2FA is the ultimate prize (point your domain at a
  cloned store, harvest everything).

**Self-check (do this openly, it's just public lookups):**
- Look up your domain's **DMARC**: query the TXT record at
  `_dmarc.dosmundos.studio`. If nothing's there, that's a gap. Aim for a policy
  of `quarantine` or `reject` once you've confirmed legit mail passes.
- Look up **SPF** (TXT at the root) and confirm your real senders are listed.
- List your subdomains; retire any that aren't in use.
- Confirm 2FA + domain lock on the registrar.

*Paste your DMARC/SPF TXT records (or just tell me the domain's DNS and I'll
tell you what to query) and I'll tell you whether you're spoofable.*

---

## 5. Leaked secrets — the free key  ⚠️ weak-spot zone

**What they do:** search public GitHub, Gists, paste sites, and breach dumps for
your domain and for Shopify token patterns (`shpat_`, `shpss_`, `shpca_`,
`X-Shopify-Access-Token`) and for staff emails. A leaked Admin API token lets
them in without any password.

**Weak spot:** any token, key, or `.env` that ever touched a public or shared
place.

**Self-check:** search public GitHub for `dosmundos` and your domain. Review
custom-app tokens in Settings → Apps → Develop apps. **Rotate anything that
might have been exposed.** Check staff emails against breach-notification
services (e.g. have-I-been-pwned-style checks) and force password resets where
hits appear.

---

## 6. Human & brand surface — social-engineering prep

**What they see:** staff names (blog authors, "about" page, LinkedIn), support
email addresses, your tone — everything needed to write a convincing fake
"Shopify security" or "supplier invoice" email. They also register
**look-alike domains** (`dosmundos-studio.com`, `dosrnundos.studio`) for
phishing.

**Weak spot:** staff who'd click a well-crafted email + no DMARC (item 4) to
stop spoofing.

**Self-check:** reduce unnecessary public contact detail; train staff to verify
sender domains; consider monitoring for look-alike domain registrations.

---

## The outsider's weak-spot shortlist (what they'd actually target)

Ranked by how attractive each is to a passive attacker:

1. **Missing / weak DMARC** → they can phish as you, for free. *(§4)*
2. **Missing / loose CSP** → any planted script can exfiltrate successfully. *(§2)*
3. **Leaked API token in public code** → direct admin access, no password. *(§5)*
4. **Unrecognized / fragile third-party scripts** → supply-chain way in. *(§3)*
5. **Stale non-Shopify subdomains** → forgotten, unpatched entry point. *(§4)*
6. **Missing HSTS / security headers** → downgrade & clickjacking room. *(§2)*

Close these six and the passive attacker's funnel runs dry — they see a hard
target and move on.

---

## To get your *actual* external findings (not just the method)

I can't reach your domain from this sandbox (its network policy blocks outbound
hosts, and that's fine — nothing to change). But every input I need is public
and safe for you to share openly:

1. **Response headers** — DevTools → Network → top request → Headers → copy
   Response Headers. → I grade §2.
2. **Page source** — right-click → View Page Source → paste/save. → I map §1
   and §3.
3. **DNS TXT records** for the root and `_dmarc.dosmundos.studio` (any public
   DNS lookup tool shows these). → I tell you if you're spoofable (§4).

Send any of those and I'll turn this methodology into concrete, store-specific
findings.
