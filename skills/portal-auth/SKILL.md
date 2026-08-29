---
name: portal-auth
description: "CONTENT-WEB PROJECTS ONLY (content-web). How to build, harden, and ship an authenticated web portal — loopback Node API behind Nginx, OAuth with PKCE, universal email login with 6-digit OTP and disposable domain filtering, transactional email delivery (Resend), sessions and CSRF, systemd hardening, least-privilege filesystem layout, secret handling, timed-flow client defects, deployment/rollback, and a pre-launch checklist. Load before designing or deploying any content-web project where a user signs in. This also carries the launch discipline that release-launch provides for mobile projects."
---

# Portal Auth — Building an Authenticated Portal

**What this solves:** an authenticated portal is not "a site with a login button." It is a small distributed system — browser, reverse proxy, application process, service accounts, an identity provider you do not control, and a backup path — and almost every defect worth writing down lives in the seams between those parts, not inside any one of them. This skill is the accumulated seam knowledge from one portal that was designed, hardened, deployed to a real host, and then broke in ways nobody predicted.

Every pitfall below is a **defect that actually shipped**, in production or in the release artifact. Each is written as **symptom → root cause → fix**, because the symptom is the part that made them hard: three of them presented as "everything looks fine" while the system was dead, silently discarding data, or letting the wrong person in.

Reference implementation: `wisdom_capsules-folder` (`api-server.js`, `quiz/web/js/challenge.js`, `ops/`, `SECURITY_LAUNCH_GATE.md`, and the `SRTL fix:` commits). Read those for exact code; read this for the reusable rules.

**Read this before writing the first auth route, and again before the first deployment.** The systemd and filesystem sections are worthless after the fact — they describe failures that only appear on a fresh host.

---

## 1. Architecture

Three separate things, three separate trust levels:

```text
browser ──HTTPS──> Nginx (TLS, rate limits, security headers, static files)
                     │
                     ├─ /             → static files from <release>/dist/   (no application involved)
                     └─ /api/, /auth/ → proxy_pass http://127.0.0.1:<port>  (the API, loopback only)

API process (unprivileged service account)
   ├─ database + private data:  /var/lib/<app>/        (outside checkout, outside static root)
   └─ secrets:                  /etc/<app>/<app>.env   (root:<appuser> 0640)
```

**Hard rules:**

1. **The API binds loopback only, and the code enforces it.** Configuration is not a control — an env var can be edited by anyone who can edit env vars. Refuse to start otherwise:

   ```js
   const host = process.env.API_HOST || '127.0.0.1';
   if (process.env.NODE_ENV === 'production' && !['127.0.0.1', '::1', 'localhost'].includes(host)) {
     throw new Error('Production API_HOST must remain on a local/private interface behind the reverse proxy.');
   }
   app.server.listen(port, host, ...);
   ```

2. **Why the API must never bind publicly** — this is not defence in depth, it is a real vulnerability. Every perimeter control lives in Nginx: TLS, per-route rate limits, connection limits, header/body size caps, the method allowlist, query-free logging. A public bind bypasses all of them at once. Worse, the API trusts `X-Forwarded-For` when `TRUST_PROXY=true` in order to attribute rate limits to real client IPs — if the port is reachable directly, **any client can set that header and choose its own rate-limit bucket**, which silently defeats every per-IP limit in the application. Loopback binding is what makes the proxy-trust assumption true.

3. **Static output is served by Nginx directly from the release's `dist/`, never through the application.** The application never becomes a file server in production.

4. **The database and any private data must live outside both the static root and the application checkout**, and the application should refuse production startup if they do not:

   ```js
   if (!outsideStaticRoot(databasePath) || !outsideApplicationRoot(databasePath)) throw new Error(...);
   ```

   This is the guard that stops a future release script from "helpfully" placing data next to the code and publishing it.

5. **Two configured origins, both validated at startup.** `APP_ORIGIN` is where the browser is; `API_ORIGIN` is where callbacks land. Parse both with a strict validator that rejects anything with a path, query, fragment, or embedded credentials, and require `https:` in production. Everything else — CORS allowance, CSRF origin checks, OAuth callback URLs, absolute links in emails — is derived from those two values, so there is exactly one place to change a hostname.

6. **Node specifics:** ESM (`"type": "module"`), no framework required — `node:http` plus a route table is enough and keeps the dependency surface near zero. If you use `node:sqlite`, require Node 22.13+ in `engines` and verify `node --version` on the host *before* installing the unit; the runtime is the single most common host-side surprise.

---

## 2. OAuth provider integration

### The provider config table

Put every provider difference in data, never in branching code:

```js
const providerSettings = {
  google: {
    id: process.env.GOOGLE_CLIENT_ID, secret: process.env.GOOGLE_CLIENT_SECRET,
    auth: 'https://accounts.google.com/o/oauth2/v2/auth',
    exchange: 'https://oauth2.googleapis.com/token',
    profile: 'https://openidconnect.googleapis.com/v1/userinfo',
    scope: 'openid email profile', pkce: true,
    authParams: { prompt: 'select_account' }
  },
  // ...one row per provider
};
const enabledProviders = Object.entries(providerSettings)
  .filter(([, value]) => value.id && value.secret).map(([name]) => name);
```

**Rules that follow from the table:**

- **A provider is enabled only when both halves of its credential pair are present.** Refuse production startup when exactly one half is set (a half-configured provider is a truncated paste, and it fails at the worst possible moment — mid-sign-in). Refuse startup when *no* sign-in method is configured at all.
- **Publish the enabled set to the client** (`GET /api/auth/providers`) and let the sign-in page hide the buttons it cannot serve. Never ship a hard-coded list of buttons in HTML — an unconfigured provider then presents a live button that dead-ends.
- **`authParams` must be provider-scoped.** Providers reject or ignore parameters they do not implement, and a parameter that means one thing to one provider can mean another elsewhere. Apply them from the table, never globally:

  ```js
  for (const [key, value] of Object.entries(config.authParams || {})) target.searchParams.set(key, value);
  ```

### PKCE

Use PKCE wherever the provider supports it, even for a confidential (server-side) client:

```js
const verifier = config.pkce ? token(48) : null;               // 48 random bytes, base64url
target.searchParams.set('code_challenge_method', 'S256');
target.searchParams.set('code_challenge',
  crypto.createHash('sha256').update(verifier).digest('base64url'));
```

Store the verifier **server-side** with the state record; send it only in the token exchange. Never use `plain`.

### `state` belongs in a short-lived server-side store, bound to a cookie

Two independent things must agree before a callback is accepted:

```sql
CREATE TABLE oauth_states (
  state_hash TEXT PRIMARY KEY, provider TEXT NOT NULL, verifier TEXT,
  return_to TEXT NOT NULL, expires_at TEXT NOT NULL, used_at TEXT
);
```

- Store `hash(state)`, never the raw state. The table is a correlation index, not a credential store.
- **Single-use, consumed inside a transaction**: select, check `used_at` and `expires_at`, stamp `used_at`, all atomically. This is what makes replay impossible under concurrency, not the expiry.
- Ten-minute expiry. Delete expired rows on every start so the table cannot grow unbounded.
- Mirror the raw state into a **separate HttpOnly cookie scoped to `/auth/`** with a matching 10-minute lifetime, and compare it to the callback's `state` with a timing-safe comparison before touching the database. The cookie binds the callback to *this browser*; the table binds it to *this flow*. Require both.
- Clear the state cookie in the same response that sets the session cookie.

### Derive the callback URL from one configured origin

```js
const callbackUrl = name => new URL(`/auth/${name}/callback`, apiOrigin).toString();
```

The same expression builds the authorization request and the `redirect_uri` sent in the token exchange, so the two can never drift — providers reject the exchange if they differ. Register the exact HTTPS URL with the provider, character for character (a trailing slash is a different URL). **Never register a wildcard redirect URI.**

If the site answers on more than one hostname, redirect every alternate host to the canonical `APP_ORIGIN` at the proxy *before* any sign-in flow starts. OAuth state cookies are host-only; a user who begins on `www.` and returns on the apex has no state cookie and gets an unexplainable "this sign-in request is invalid."

### `prompt=select_account` — not optional

**Symptom:** a user opens sign-in, is never asked which account to use, and lands directly inside an existing portal account. It looks exactly like a leaked session cookie.

**Root cause:** an authorization request carrying only `client_id`, `redirect_uri`, `response_type`, `scope`, `state`, and PKCE lets the provider silently reuse whatever provider session the browser already holds. Verified in pilot testing: a fresh private window that had signed into the provider earlier in the same browser session was dropped straight into an existing portal account. It was confirmed to be a genuine fresh OAuth callback — a new session row and a new `authentication_success` event — so the session boundary itself was intact. The provider simply chose the account, and offered no way to switch.

**Fix:** send `prompt=select_account` via that provider's `authParams`. This matters most on a shared or family machine, where the previous person's provider session otherwise decides whose account is opened.

### Identity linking must be collision-safe

- `UNIQUE(provider, provider_subject)` on the identity table. Look up by that pair first; it is the only stable identifier a provider gives you.
- **Link by email only when the provider asserts the email is verified.** Google: `profile.email_verified === true`. GitHub: the profile response is not enough — call `/user/emails` and take the primary+verified entry. Facebook: its profile response carries no verified-email assertion, so it must never auto-link by email.
- If a verified email already belongs to a *different* sign-in method, **refuse with a 409 and tell the user to use the original method**. Do not silently merge. Silent merge on an unverified or provider-supplied email is account takeover.
- Rate-limit both `/auth/<provider>` and `/auth/<provider>/callback` per IP, and log an `authentication_failure` security event (with hashed subject) for every `oauth_state_invalid` / `oauth_exchange_failed` / `oauth_profile_failed`.

### `returnTo` is an open-redirect vector

Allowlist it by prefix, and fall back to a fixed default. Never accept an absolute URL:

```js
const safeReturnTo = value =>
  String(value || '').startsWith('/portal/') ? String(value) : '/portal/dashboard/';
```

Apply the allowlist when the flow *starts* (before storing it) **and** when the callback resolves it.

---

## 3. A critical provider caveat — a test-user list is not access control

**This is the single most dangerous assumption in this document.**

Google's OAuth consent screen in **Testing / unpublished** mode presents a "test users" list, and every plausible reading of the console UI suggests that only those accounts can sign in. **It does not reliably restrict sign-in when the application requests only non-sensitive scopes** (`openid email profile`).

**Verified in production on this project:** two Google accounts that were *not* on the test-user list completed sign-in, were provisioned full portal accounts, and held live sessions.

**Consequences you must design for:**

- If your pilot needs invite-only access, **you must implement an allowlist in the application.** The provider's publishing status is a review/verification state, not an authorization boundary.
- A portal page that is unlinked, `noindex`, and absent from the sitemap is **not access control either**. Invited users share URLs. Treat "unlisted" as a discovery preference and nothing more.
- Any planning document that says "restrict the pilot by keeping the app in Testing mode and adding test users" is wrong and must be corrected wherever it appears.

**Application allowlist — required behaviour if you need one:**

```text
PORTAL_BETA_EMAIL_ALLOWLIST=person@example.com,other@example.com
```

- Normalize addresses (trim, lowercase) on both sides of the comparison before matching.
- **OAuth sign-in must require a *verified* email that is on the allowlist.** A provider that supplies no verified-email assertion must be denied outright during an allowlisted pilot.
- Email/magic-link sign-in for a non-allowed address must return **exactly the same generic accepted response** as an allowed one, and must not send the webhook. Never reveal whether an address is on the list.
- Never log raw email addresses; log a hash/fingerprint.
- Add authentication regression tests for both the allowed and denied paths before shipping it.

---

## 4. Universal Email Authentication: 6-Digit OTP, Magic Links, Disposable Filtering & Delivery

Email authentication without passwords is clean and low-friction, but implementing it robustly across devices and real email ecosystems requires solving four critical problems:

```text
User enters any valid email ───> [Disposable Domain Filter] ───> [Generate Token + 6-Digit Code]
                                                                        │
                                                                 [Resend API]
                                                                        │
                       ┌────────────────────────────────────────────────┴───────────────────────────────┐
                       ▼                                                                                ▼
             [In-Tab 6-Digit Code]                                                           [1-Click Fallback Link]
   (User types OTP into current screen)                                              (User clicks link in browser)
                       │                                                                                │
            POST /api/auth/email/verify                                                       GET /auth/email/callback
                       │                                                                                │
                       └───────────────────────► [Validate, Hash, Consume] ◄────────────────────────────┘
                                                              │
                                                   [Issue Session Cookie]
```

### 4.1 Universal Email Acceptance Principle

- **Accept any legitimate domain:** Never restrict sign-in to a hardcoded whitelist of popular webmail providers (e.g. only Gmail/Outlook) unless explicitly building a closed corporate intranet. Corporate addresses, academic domains (`.edu`), privacy-focused providers (`proton.me`, `tuta.com`), regional domains, and personal custom domains must work seamlessly.
- **Normalization:** Normalize addresses on ingress (`trim()`, `toLowerCase()`, punycode conversion for internationalized domain names via `domainToASCII`).
- **Do not leak user existence:** The response to `/api/auth/email/start` must be an identical generic `202 Accepted` whether the user is new or existing.

### 4.2 Disposable & Burner Domain Filtering

- **Why filter disposable domains:** Temporary burner email services (Mailinator, 10MinuteMail, TempMail, GuerrillaMail, etc.) allow automated abuse, throwaway quiz spam, and certificate flooding while breaking long-term account recovery.
- **Load a canonical plain-text denylist:** Store disposable domains in a sorted, newline-delimited text file (e.g. `disposable-email-domains.txt`). Load it into a `Set` at startup:
  ```js
  function loadDisposableDomains(filePath) {
    if (!fs.existsSync(filePath)) return new Set();
    const content = fs.readFileSync(filePath, 'utf8');
    return new Set(content.split(/\r?\n/).map(line => domainToASCII(line.trim().toLowerCase())).filter(Boolean));
  }
  ```
- **Parent-domain traversal matching:** Attackers frequently create subdomains on disposable providers (e.g. `xyz.mailinator.com`). Your lookup must traverse the domain hierarchy from full host to root:
  ```js
  function disposableDomain(domain) {
    let current = domain;
    while (current) {
      if (disposableSet.has(current)) return true;
      const dot = current.indexOf('.');
      if (dot === -1) break;
      current = current.slice(dot + 1);
    }
    return false;
  }
  ```
- **Avoid false positives on local parts:** Reject ONLY based on the domain part after the `@`. An address like `mailinator.user@gmail.com` has a disposable keyword in the local part but is hosted on Gmail — it must be accepted.

### 4.3 The "Device Shift / Tab Shift" Problem & The 6-Digit OTP Solution

- **The Problem with Pure Magic Links:** When a user initiates sign-in on their laptop (e.g. during an exam or purchase), they almost always check their email on their **mobile phone**. Clicking a magic link on their phone authenticates the mobile browser, leaving the laptop tab stuck waiting on the sign-in screen.
- **The Solution: 6-Digit Verification Code (OTP) + Fallback Link:**
  - When the user submits their email, the open browser tab **stays on the sign-in screen** and transitions immediately to a **6-digit code input field**.
  - The email sent contains both:
    1. A prominent, large 6-digit numeric verification code (`123456`).
    2. A 1-click fallback link for users opening the email on the same device.
  - The user reads the 6 digits from their phone and types them directly into the open laptop screen.
  - Submitting the code calls `POST /api/auth/email/verify`, sets the session cookie in that browser, and redirects immediately to the target dashboard.

### 4.4 Data Model & Token Security

```sql
CREATE TABLE email_tokens (
  token_hash TEXT PRIMARY KEY,
  code_hash TEXT,
  email TEXT NOT NULL,
  return_to TEXT NOT NULL,
  expires_at TEXT NOT NULL,
  used_at TEXT
);
CREATE INDEX IF NOT EXISTS email_tokens_by_expiry ON email_tokens(expires_at, used_at);
```

- **Hash both credentials:** Store SHA-256 hashes of both the raw magic-link token (`token_hash = hash(token)`) and the 6-digit OTP code (`code_hash = hash(code)`). Never store raw tokens or plain text codes in SQLite.
- **Cryptographic randomness:** Generate the 6-digit code using a CSPRNG (`crypto.randomInt(100000, 1000000).toString()`).
- **Atomic consumption:** Check validity, expiry, and stamp `used_at = datetime('now')` inside a single database transaction. Once verified (via code or link), the token is consumed and cannot be replayed.
- **Auto-cleanup:** Delete previous tokens for the same email or expired tokens whenever a new sign-in is issued (`DELETE FROM email_tokens WHERE email = ? OR expires_at <= datetime('now')`).

### 4.5 Brute-Force Defense for 6-Digit Codes

A 6-digit code has only $1,000,000$ combinations. Without strict rate limiting, it can be brute-forced within its 10-minute validity window.
- **Limit verification attempts per account:** Maximum 10 verification attempts per hour per normalized email address on `POST /api/auth/email/verify`.
- **Limit verification attempts per IP:** Maximum 30 attempts per hour per IP.
- **Limit code issuance:** Maximum 5 code generation requests per hour per email address, 15 per hour per IP on `POST /api/auth/email/start`.
- **Short lifespan:** 10 minutes maximum.

### 4.6 Transactional Email Integration (Resend API)

Use a lightweight, zero-dependency REST integration with standard `fetch`:
- **Native Fetch Call:**
  ```js
  const response = await fetch('https://api.resend.com/emails', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${process.env.RESEND_API_KEY}`
    },
    body: JSON.stringify({
      from: process.env.EMAIL_FROM || 'App <support@yourdomain.com>',
      to: [email],
      subject: `${code} is your App verification code`,
      html: `...<h1>${code}</h1>...<a href="${signInUrl}">Or sign in with 1-click</a>...`,
      text: `Your verification code is: ${code}\n\nOr click: ${signInUrl}`
    }),
    signal: AbortSignal.timeout(8000)
  });
  ```
- **Domain Verification Caveat:** In Resend (and similar providers), testing with the default sandbox sender (`onboarding@resend.dev`) **only permits sending to your own account email**. Attempting to send to any external address (Outlook, Proton, etc.) will fail with an upstream 403 error until custom domain DNS records (DKIM, SPF, MX) are verified and `EMAIL_FROM` uses that verified domain.
- **Rollback on delivery failure:** If the upstream email API returns an error or times out, immediately delete the inserted `email_tokens` row and return `503 email_delivery_unavailable` so the user is not left waiting for a code that never dispatched.

### 4.7 Client-Side UX for OTP Sign-In

- **Single-Card Step Transitions:** Keep the user on the same card without full page reloads:
  1. *Step 1 (Email)*: Input field + "Continue".
  2. *Step 2 (Code)*: Hide OAuth provider buttons (Google, GitHub) and dividers to eliminate visual distraction; change title to "Enter verification code"; show target email in subtitle; autofocus the 6-digit input.
- **Auto-Submission on 6th Digit:** Listen to the `input` event. Strip non-digits, and automatically trigger form submission as soon as `value.length === 6`.
- **Digit Readability:** Style the input with monospaced font and wide letter-spacing (`letter-spacing: 0.35em; font-size: 1.5rem; text-align: center;`).
- **Escape Hatches:** Provide clear, styled text actions:
  - `← Use different email` (smoothly resets state back to Step 1).
  - `Resend code` (re-triggers generation and shows feedback).

---

## 5. Sessions and CSRF

### Cookies

Set together, cleared together:

| Cookie | httpOnly | secure | sameSite | path | Purpose |
|---|---|---|---|---|---|
| `<app>_session` | **true** | true (prod) | `lax` | `/` | The session credential. Opaque random value. |
| `<app>_csrf` | **false** | true (prod) | `lax` | `/` | Readable by JS so it can be echoed in a header. |
| `<app>_oauth_state` | true | true (prod) | `lax` | `/auth/` | Short-lived flow correlation only. |

`sameSite=lax` is what allows the OAuth callback (a top-level cross-site GET redirect) to arrive with cookies attached. `strict` breaks the callback; `none` gives away CSRF protection. `lax` is the correct answer here, and the origin + CSRF checks below cover what it does not.

### Store hashes, never values

```sql
CREATE TABLE sessions (
  token_hash TEXT PRIMARY KEY, user_id TEXT NOT NULL, csrf_hash TEXT NOT NULL,
  created_at TEXT NOT NULL, expires_at TEXT NOT NULL, last_seen_at TEXT NOT NULL, revoked_at TEXT
);
```

Session and CSRF values are CSPRNG `base64url` tokens; only their SHA-256 hashes are persisted. A database disclosure then does not hand over live sessions.

### CSRF: double-submit *plus* a server-side check

```js
function csrf(req, session) {
  const cookie = parseCookie(req.headers.cookie || '')[CSRF_COOKIE];
  const header = req.headers['x-csrf-token'];
  if (!cookie || !header || !timeSafeEqual(cookie, header) || !timeSafeEqual(hash(header), session.csrf_hash))
    throw new HttpError(403, 'Your session safety token is missing or invalid. Refresh and try again.', 'csrf_failed');
}
```

The third condition is the one people omit. Plain double-submit only proves the caller could read a cookie; binding the header to the **hash stored with the session row** proves the caller holds *this session's* token. Call `csrf()` on every state-changing authenticated route — POST, PUT, PATCH, DELETE. No exceptions, including logout.

Pair it with an origin check on state-changing methods, and **require the `Origin` header to be present in production** (a missing Origin is a non-browser or a stripped proxy, neither of which should be writing):

```js
if (['POST','PUT','PATCH'].includes(req.method)) requireTrustedOrigin(req);
```

Echo CORS headers only for an exact `APP_ORIGIN` match, with `Vary: Origin`.

### Idle vs absolute expiry — you need both

- **Absolute** (`expires_at`, e.g. 7 days): the session dies at a fixed wall-clock time no matter how active the user is. This caps the damage window of a stolen cookie.
- **Idle** (`last_seen_at` + idle window, e.g. 12 hours): the session dies from disuse. This is what protects an abandoned shared machine.

Check both on every authenticated request, refresh `last_seen_at` on success, and **revoke the row when either check fails** rather than merely returning 401 — otherwise an expired-but-unrevoked session lingers as a valid-looking database record. Validate at startup that idle ≤ absolute; the reversed configuration silently makes one of them dead code.

### Rotation and revocation

- **Rotate on sign-in.** Before inserting the new session, revoke the session named by any session cookie the browser already presented. This closes session fixation.
- **Logout must be server-side.** Set `revoked_at` on the row *and* clear the cookies. Clearing cookies alone leaves a fully valid session that anyone holding the old value can keep using — the cookie was never the authority.
- Query sessions with `WHERE token_hash = ? AND revoked_at IS NULL`, so a revoked row is unusable by construction.
- **A logout route nobody can reach is not a logout feature.** The reference portal shipped a correct, CSRF-protected, server-side-revoking `POST /auth/logout` — and *nothing in the client ever called it*. Every backend test passed. Users simply could not sign out. Grep the client for the route before calling the story done: if the endpoint appears zero times outside your tests, you have a spec, not a feature.
- Sign-out matters most on **shared machines**, which is also where the provider silently reuses its own session (§2, `prompt=select_account`). Treat "can this person hand the laptop to someone else safely?" as the acceptance criterion, not "does the endpoint return 200?"
- Put the control where users expect it: an account affordance in the header, showing **which identity is signed in** — name *and* email. On a household machine the email is the only thing that answers "whose account am I actually in?", which is precisely the question the provider's silent reuse creates.

---

## 6. systemd hardening pitfalls

**All three of these shipped broken.** Each was being compensated for by a hand-written drop-in override on the production host, so the unit files in the repository would have produced a service that could never start on any fresh deployment — and nobody would have known until the next host build.

> **Meta-lesson, and the most valuable rule in this section:** when a host drill forces you to change a unit on the box, **the repository unit is now wrong**. Fix it in the repo, delete the drop-in, and add a regression assertion on the unit file's text. A drop-in that "makes it work" is a silent time bomb aimed at your next deploy.

### 6.1 Release symlink + ESM entry guard → the service exits 0 and looks healthy

**Symptom:** `systemctl start` succeeds. `systemctl status` shows no error. `Restart=on-failure` never fires. Nothing is listening on the port. The journal is empty. The service is dead and reports success.

**Root cause:** the standard ESM "am I the entry point?" guard compares two different spellings of the same file:

```js
if (path.resolve(process.argv[1] || '') === fileURLToPath(import.meta.url)) { /* listen() */ }
```

When `ExecStart` points at `/srv/<app>/current/api-server.js` and `current` is a symlink into `releases/<sha>/`, Node resolves `import.meta.url` to the **realpath** (`/srv/<app>/releases/<sha>/api-server.js`) while `process.argv[1]` keeps the **symlink path**. They never match, the guard is false, `listen()` is skipped, the module finishes evaluating, and the process **exits 0** — a clean exit that `Restart=on-failure` deliberately will not retry.

**Fix:** run the entry point with `--preserve-symlinks-main`:

```ini
ExecStart=/usr/bin/node --preserve-symlinks-main /srv/<app>/current/api-server.js
```

Assert it in the regression suite against the unit file itself:

```js
assert(/^ExecStart=\/usr\/bin\/node --preserve-symlinks-main /m.test(apiUnit),
  'the API unit must pass --preserve-symlinks-main, or the entry guard never matches under the release symlink and the process exits 0 without listening');
```

*Also worth knowing:* `Type=simple` tells systemd the service is up as soon as it forks, so it cannot detect this either. If your runtime supports it, `Type=notify` plus a readiness notification after `listen()` turns this class of failure into a start-up error instead of a silent success.

### 6.2 `RestrictSUIDSGID=true` blocks a `2750` setgid directory

**Symptom:** the service refuses to start on a host where the data directory *already exists with the right mode*. The failure surfaces as a permission error from a `mkdir`/`chmod` call that should have been a no-op.

**Root cause:** `RestrictSUIDSGID=` is not a filesystem permission — it installs a **seccomp filter on the arguments** of `mkdir`, `mkdirat`, `chmod`, `fchmod`, `fchmodat`, and friends. It rejects any call whose *mode argument* carries the setuid or setgid bit, regardless of whether the resulting state would change anything. An application that (correctly) ensures its data directory is `2750` on startup issues exactly such a call and is killed by the filter.

**Fix:** set it false, and say why in the unit so nobody "hardens" it back:

```ini
# Must stay false. This is a seccomp filter on the ARGUMENTS of mkdir/chmod, so it
# rejects the 2750 setgid mode the application is required to apply to
# /var/lib/<app> - the very mode this unit's UMask depends on.
RestrictSUIDSGID=false
```

`NoNewPrivileges=true`, `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`, `RestrictAddressFamilies=`, and the `ReadOnlyPaths=`/`ReadWritePaths=` pair all remain in force — this single directive is the only one that must be relaxed, and the setgid group model it conflicts with is itself a security control.

### 6.3 `User=` strips CAP_SETUID from the effective set — **even `User=root`**

**Symptom:** a root-owned orchestrator script fails at `runuser -u <backupuser>` with `EPERM`. Nothing in the unit removes the capability. `systemctl show -p CapabilityBoundingSet` **lists `cap_setuid`**, which appears to prove the capability is present.

**Root cause:** whenever `User=` is specified — including `User=root` — systemd assumes all identity changes are complete and drops `CAP_SETUID` from the process's **effective** set. The *bounding* set still contains it, which is exactly why `CapabilityBoundingSet` is a misleading place to look. With `NoNewPrivileges=true` the capability cannot be regained, so any privilege drop the service performs itself fails.

**Fix:** do not set `User=`/`Group=` at all on a unit that must drop privileges internally. Services already run as root by default, so omitting them changes nothing except keeping `CAP_SETUID`:

```ini
# Deliberately NOT set. systemd strips CAP_SETUID from the effective set whenever
# User= is specified - even as root - so the orchestrator's runuser into
# <backupuser> fails with EPERM. Services already default to root.
# User=root
# Group=root
```

**Diagnosis technique for this whole class of problem — `systemd-run` bisection.** When a command works in a shell and fails under a unit, do not read the unit and reason about it. Run the exact command under `systemd-run` with a *minimal* property set, confirm it works, then add the unit's directives back one at a time until it breaks:

```bash
systemd-run --wait --pty /path/to/script.sh                       # baseline: does it work at all?
systemd-run --wait --pty -p NoNewPrivileges=true /path/to/script.sh
systemd-run --wait --pty -p User=root /path/to/script.sh          # ← this is where it breaks
```

The directive that changes the outcome is the answer. This takes minutes and replaces hours of reasoning about capability sets, and it is the only reliable way to find a directive whose documented description does not describe the behaviour you are hitting.

---

## 7. Filesystem layout and least privilege

Target layout (adapt names; keep the relationships):

```text
/srv/<app>/releases/<sha>/       app checkout                <appuser>:<appuser>       0755
/srv/<app>/current -> releases/<sha>
/var/lib/<app>/                  database + private data     <appuser>:<dbgroup>       2750
/var/backups/<app>/              encrypted backup staging    <backupuser>:<backupuser> 0700
/etc/<app>/                      secrets directory           root:root                 0751
/etc/<app>/<app>.env             app secrets                 root:<appuser>            0640
/etc/<app>/backup.env            backup config               root:root                 0600
/etc/<app>/backup-remote.conf    backup remote credentials   root:<backupuser>         0640
```

Accounts: `<appuser>` runs the API, `<backupuser>` runs backups, and both are members of `<dbgroup>` — that shared group is the *only* thing they have in common.

**The setgid data directory (`2750`).** The setgid bit makes every file the application creates in `/var/lib/<app>/` inherit group `<dbgroup>`, which is what keeps SQLite's `-wal` and `-shm` sidecars readable by the backup account without a cron job re-chowning them. `0` for other means nothing outside those two accounts can read the database at all. The API unit sets `UMask=0027` so new files land `0640`. This is also the mode that `RestrictSUIDSGID=true` (§6.2) forbids — the two directives are in direct conflict and the setgid model wins.

**`/etc/<app>` at `0751` — the traversal problem.** `<backupuser>` must read `/etc/<app>/backup-remote.conf`, but is deliberately **not** a member of `<appuser>`'s group, because that group can read `<app>.env` and therefore the OAuth client secrets. Without the `1` (execute/traverse) bit for other on the directory, `<backupuser>` cannot even path into `/etc/<app>/` to reach its own file.

> **Never solve a traversal problem by adding an account to a group.** `0751` grants traverse and nothing else: `<backupuser>` can open a file whose own mode permits it, and cannot list the directory or read anything else. Adding `<backupuser>` to `<appuser>` would "fix" it by handing the backup account your OAuth secrets — the exact separation the two accounts exist to create.

**Script ownership matters as much as mode.**

**Symptom:** a scheduled job fails with permission denied on a script that is `0750` and demonstrably executable — running it by hand as root works.

**Root cause:** `root:root 0750` grants execute to owner (root) and group (root) only. A script invoked through `runuser -u <backupuser> -- /path/script.sh` executes **as `<backupuser>`**, which is neither, so it is denied.

**Fix:** own it by the account that will run it — `root:<backupuser> 0750`. Root still owns it (so the unprivileged account cannot modify it), and the group bit grants exactly the execute permission needed. Scripts that only ever run as root stay `root:root 0750`.

**Re-apply ownership and mode after every release.** A fresh checkout resets both, and the failure is **silent until the next scheduled run** — a backup that has not run for a week looks identical to one that ran fine. Put the `install`/`chown`/`chmod` commands in the deployment procedure, and verify them as an explicit post-deploy step, not from memory:

```bash
install -o root -g <backupuser> -m 0750 ops/backup.sh /srv/<app>/current/ops/backup.sh
stat -c '%U:%G %a %n' /srv/<app>/current/ops/*.sh /etc/<app> /var/lib/<app>
```

**Privilege separation in the backup path.** The account that reads the database must never be able to write it. The working shape: a **root-owned orchestrator** stops the API, checkpoints the WAL, verifies the WAL is empty, then `runuser`s into `<backupuser>` for the snapshot, integrity check, encryption, and upload — and restarts the API from an `EXIT` trap **even when the backup fails**. The unprivileged half opens the database read-only (`file:...?immutable=1`), encrypts to a public recipient key only, and writes a SHA-256 sidecar. The private decryption key never exists on the host.

---

## 8. Secret handling

**Never print a secret value. Never ask the user to paste one into chat.** Both rules are absolute, and they hold even when the user offers.

The procedure that satisfies both:

1. **Write a placeholder template locally** for the user to fill in, outside the repository, in a path they control:

   ```text
   C:\Users\<user>\Documents\symphony-secrets\<app>\<app>.env
   ```

   Fill in everything that is not secret (origins, paths, timeouts) and leave `GOOGLE_CLIENT_SECRET=USER_ADDS_VALUE` style markers for the rest.

2. **Verify without reading.** You may check that the file exists, that every required variable name is present, that each value is non-empty, and that no placeholder marker survives. You may not print, echo, log, or quote any value:

   ```bash
   awk -F= '/^[A-Z_]+=/ { printf "%s %s\n", $1, (length($2) ? "SET" : "EMPTY") }' "$f"
   grep -c 'USER_ADDS_VALUE\|replace-with-secret' "$f"    # must be 0
   ```

3. **Transfer with `scp`, then install atomically** with explicit owner and mode — never `cp` followed by a separate `chmod`, which leaves a window where the file is world-readable:

   ```bash
   scp <app>.env <host>:/tmp/<app>.env.staged
   ssh <host> 'install -o root -g <appuser> -m 0640 /tmp/<app>.env.staged /etc/<app>/<app>.env'
   ```

4. **`shred` every intermediate**, on both ends, immediately after install:

   ```bash
   ssh <host> 'shred -u /tmp/<app>.env.staged'
   ```

5. **Run an absence sweep** to prove a private key or secret exists in exactly one place. Do not assume — search:

   ```bash
   git -C <repo> grep -I -n 'AGE-SECRET-KEY\|BEGIN OPENSSH PRIVATE KEY' -- . || echo 'clean: working tree'
   git -C <repo> log --all -p -S 'AGE-SECRET-KEY' | head            # history, not just HEAD
   ssh <host> "sudo grep -rIl 'AGE-SECRET-KEY' /srv /etc /var/log 2>/dev/null; \
               sudo journalctl --no-pager | grep -c 'AGE-SECRET-KEY'"
   ```

   Sweep the **git history**, not only the working tree — a secret removed in a later commit is still published. A key that survives the sweep in exactly one approved location, and nowhere else, is the only acceptable end state.

**Structural rules:**

- Secrets live in files outside the repository, read via `EnvironmentFile=`. Never in the unit, never in `ExecStart`, never in a build artifact, never in browser storage.
- Keep development, test, and production credentials distinct. Rotate immediately on any suspected disclosure, and treat "it appeared in a log once" as disclosure.
- Use a **dedicated OAuth client for infrastructure** (e.g. the backup remote). Never reuse the portal's sign-in client for anything else — they have different lifetimes, different blast radii, and different people needing to rotate them.
- Only the **public** half of an encryption keypair belongs on the host. Custody of the private half is the user's, off-host and off-repo; it is supplied temporarily for a restore drill and shredded afterwards.

**Keep secrets out of logs, structurally.** Magic-link tokens and OAuth authorization codes arrive in query strings, so a default access-log format publishes them:

```nginx
# Query-free log format: never use $request or $request_uri here.
log_format <app>_safe '$remote_addr [$time_local] "$request_method $uri $server_protocol" '
                      '$status $body_bytes_sent $request_time "$http_user_agent"';

location ~ ^/auth/(google|github)/callback$ {
  access_log off;
  error_log /dev/null emerg;      # error logs quote the full request line too
  proxy_pass http://127.0.0.1:<port>;
}
```

Application logs should be **minimal structured security events** — `authentication_success`, `authentication_failure`, `authorization_denied`, `rate_limit`, `logout`, `service_failure` — with subjects and IPs stored as a truncated hash, never in the clear. Never log request bodies, cookies, tokens, or provider responses.

---

## 9. Client-side pitfalls in timed / authenticated flows

Every one of these was a real defect in a live portal. They share a shape: the server was correct and the browser quietly disagreed.

### 9.1 Never compare a server-issued deadline against `Date.now()`

**Symptom:** a user's timed attempt is submitted blank, milliseconds after it starts, before a single question renders. In production: an attempt created and auto-submitted 257 ms later, scoring zero. Unreproducible on any developer machine.

**Root cause:** the countdown compared the server's `deadlineAt` to the **browser's** clock. A machine running more than the session duration fast satisfies `remaining <= 0` on the very first tick. The user's clock is not your clock, is not monotonic, and can be wrong by hours.

**Fix:** publish the server's own time alongside every deadline-bearing payload, measure the skew **once** at load, and anchor the countdown to the corrected clock so only *locally elapsed* time advances it:

```js
// server: include serverTime on every response that carries a deadline
{ serverTime: new Date(now()).toISOString(), attempt: { deadlineAt, ... } }

// client
const skew = payload.serverTime ? Date.now() - new Date(payload.serverTime).getTime() : 0;
const serverNow = () => Date.now() - skew;
const remaining = Math.max(0, Math.ceil((deadline - serverNow()) / 1000));
```

Do this on **every** surface that shows the same countdown — the timed screen *and* the "you have something in progress" panel elsewhere in the portal. And keep the server authoritative regardless: it must finalize an expired item itself and reject late writes, so a client that never ticks changes nothing.

### 9.2 bfcache: the back button restores a page without re-running your startup code

**Symptom:** a user goes back to a previous page and finds a button that is enabled but that the server rejects — with an error the page offers no way to act on.

**Root cause:** the browser's back/forward cache restores the whole page — DOM, JS heap, closures — **without re-executing module or `DOMContentLoaded` code**. Every piece of authorization or eligibility state captured at first load is now arbitrarily stale.

**Fix:** re-check with the server on `pageshow` when the page came from the cache:

```js
window.addEventListener('pageshow', event => { if (event.persisted) refreshFromServer(); });
```

Any page whose rendering depends on server-side state that can change while the user is away needs this. It is two lines and it is almost never written.

### 9.3 Offer *Resume*, not a disabled or lying button

**Symptom:** the user has work in progress on the server, and the page shows a Start button gated only by a local checkbox. Re-ticking the checkbox re-arms the button; the server then refuses with "finish your active attempt first" — advice the page provides no way to follow.

**Root cause:** the UI modelled a single state ("can start / cannot start") where the server has three ("can start", "in progress", "cooling down"), and the client's copy of that state was local.

**Fix:** when the server reports in-progress state, replace the action entirely — a **Resume** button that navigates to the existing work, with a live (skew-corrected) countdown, and the pre-conditions locked rather than re-armable. On any server refusal, re-read state and re-render rather than just displaying the error. And **name the states the user experiences**: work the server finalized as *expired* should read "you abandoned it," not a bare cooldown timestamp.

### 9.4 Guard unloads during a timed flow — and lift the guard on submit

```js
const unloadGuard = event => { if (submitting) return undefined; event.preventDefault(); event.returnValue = ''; return ''; };
window.addEventListener('beforeunload', unloadGuard);
// ...on successful submit, BEFORE navigating away:
window.removeEventListener('beforeunload', unloadGuard);
```

The state lives on the server and survives a reload, but a stray refresh or back gesture mid-flow is never intentional. **Forgetting the removal is its own defect** — the user completes the flow correctly and is then interrogated about leaving. Assert both halves in tests.

### 9.5 Persist form fields on blur, not only via a save button

**Symptom:** the user types their name, sees it on screen, proceeds — and the system uses the old value everywhere. Confirmed in production: the database still held the provider-supplied name and the access log recorded **zero** `PATCH` requests to the update route.

**Root cause:** the field persisted only through its own low-affordance "Save" text button. A typed-but-unsaved value looks identical to a saved one. The UI showed a value the server had never received.

**Fix:** persist on `blur`, **and** again immediately before the value is consumed by the next step — and refuse to proceed if that save fails:

```js
nameInput?.addEventListener('blur', () => { saveName(); });
// ...at the point of use:
if (!(await saveName())) return;    // do not create the attempt with an unsaved name
```

The general rule: **a value the user can see must be the value the server holds.** If those can diverge, the UI is lying.

### 9.6 A bare descendant selector matches nested elements too

**Symptom:** one column of a header row is taller than its neighbours; a label wraps onto a third line. Only on some screens, only for some content.

**Root cause:** `.bar span` matches **every** descendant span — including one nested inside a child `<strong>`. The nested span inherited a block-level uppercase label style, and wrapped.

**Fix:** scope to the direct-child path, and explicitly reset anything nested deeper:

```css
.bar > div > span { display: block; text-transform: uppercase; font-size: .62rem; line-height: 1.1rem; }
.bar strong span  { display: inline; font: inherit; color: inherit; text-transform: none; }
```

Equalize label and value line-heights so sibling blocks are exactly the same number of rows tall, and give any user-supplied string in a fixed row `overflow: hidden; text-overflow: ellipsis` rather than letting it stretch the layout.

### 9.7 User strings in fixed-size artifacts need a server-side cap *and* dynamic sizing

**Symptom:** a generated PDF/certificate renders a long name past the printed border, and a short name visibly off-centre.

**Root cause:** two independent bugs. (a) The length limit lived only in the input's `maxlength`, which is a UI convenience, not a constraint — any direct API call bypasses it. (b) The text was drawn at a **fixed size and fixed x-origin**, which can only be correct for one exact string length.

**Fix, both halves:**

```js
const NAME_MAX = 60;   // one printed line; comfortably fits long real names
// server-side, in the normalizer used by EVERY write path:
if (name.length < 2 || name.length > NAME_MAX)
  throw new HttpError(400, `Name must be between 2 and ${NAME_MAX} characters.`, 'invalid_name');

// dynamic sizing + centring (Helvetica average advance ≈ 0.55em, 792pt landscape page)
const size  = Math.min(26, 620 / Math.max(1, text.length * 0.55));
const width = text.length * size * 0.55;
const x     = Math.max(86, (792 - width) / 2);
```

Keep the input's `maxlength` and its helper copy **numerically identical** to the server constant, and assert that in tests — a mismatch is a guaranteed support ticket. Then prove the geometry at the boundaries: compute the drawn box at the minimum, a middle, and the maximum length and assert it stays inside the border at all three. Also escape the string for the artifact format (in PDF: backslash, parentheses, and newlines) and let the HTML rendering **wrap** long values rather than clipping them against an `overflow: hidden` parent.

### 9.8 Scope authenticated chrome by construction, not by convention

If the portal is in a **soft launch** — reachable but unlisted, `noindex`, absent from navigation and sitemap — then any authenticated UI you add is a new way to leak its existence. An account menu placed in the *shared* base template renders on every public marketing page too, and a signed-in visitor browsing public content advertises the private area.

Do not solve this with a runtime `if (page.startsWith('/app'))`. Inject authenticated chrome in the **same build step that injects the authenticated CSS and JS**, so the public template physically cannot contain it:

```js
function withAppAssets(html) {
  return html
    .replace('</head>', '  <link rel="stylesheet" href="/styles/app.css">\n</head>')
    .replace(/^[ \t]*<button class="hamburger"/m, ACCOUNT_MENU + '        <button class="hamburger"')
    .replace('</body>', '  <script src="/js/app.js"></script>\n</body>');
}
```

Then assert **both directions** against built output — present on every authenticated route, absent from a list of public routes — and verify each assertion fails when deliberately broken. Absence tests are the ones that rot silently: nobody notices a leak, because a leak looks like a feature.

Deliberately exclude the control from any **timed flow** page. Sign-out during a live attempt strands server-side state behind a cooldown, and a destructive action one click from a running exam is a UX trap. If that page renders from its own template, exclusion is free — take it.

---

## 10. Testing discipline

> **Every regression assertion must be verified to FAIL when its defect is deliberately reintroduced. A test that cannot fail is worthless.**

This is the rule that was enforced on every fix in the reference project, and it is the reason the pitfalls above stay fixed. An assertion written after a fix, against already-correct code, passes on day one and proves nothing — it may be asserting on the wrong file, the wrong string, or nothing at all.

**The technique, per assertion:**

1. Write the assertion. Run the suite. It passes.
2. **Temporarily reintroduce the exact defect** — revert the one line, loosen the one constant, delete the one listener.
3. Run the suite. Confirm it fails **with the specific message you wrote**, not merely that something failed somewhere.
4. Restore the fix. Run again. Confirm green.
5. Record in the commit message that each new assertion was confirmed to fail when its defect is reintroduced.

**Write the assertion message as the explanation of the defect**, not as a restatement of the code. The message is what a future agent reads at 2am:

```js
assert(!/const remainingMs = deadline - Date\.now\(\);/.test(client),
  'countdown must not measure a server-issued deadline against the browser clock: a fast client auto-submits a blank attempt on the first tick');

assert(client.includes("window.removeEventListener('beforeunload', unloadGuard)"),
  'the unload warning must be lifted once the attempt is submitted');

assert(!/^User=/m.test(backupUnit) && !/^Group=/m.test(backupUnit),
  'the backup unit must not set User=/Group=: systemd then strips CAP_SETUID and the orchestrator cannot runuser into <backupuser>');
```

**What to assert at each level:**

| Level | Assert | Example |
|---|---|---|
| Source text | Client/CSS defects with no runtime hook | no bare `.bar span` selector; a `pageshow`/`persisted` handler exists; both halves of the unload guard |
| Config artifacts | **Shipped unit and proxy files, as files** | `--preserve-symlinks-main` present; `RestrictSUIDSGID=false`; no `User=` on the backup unit; provider `authParams` present |
| Request level | Real HTTP against a real server instance with a **fake clock** | over-limit name rejected / at-limit accepted; CSRF-less write is 403; cross-account read is 404 |
| Leak checks | Negative assertions on serialized responses | payloads never contain answer keys, private metadata, or another user's data |

**A fake clock is mandatory** for anything with expiry, cooldowns, or deadlines: inject `clock` into the server factory and advance it (`currentTime += 20 * 60 * 1000 + 1`) rather than sleeping. It makes idle/absolute expiry, single-use state, and timed finalization deterministically testable in milliseconds.

**Assert on the shipped ops artifacts.** The systemd defects in §6 were invisible to every application test in the suite because nothing read the unit files. Reading a config file and asserting on its text is cheap, and it is the only thing standing between a host drop-in and a broken fresh deploy.

---

## 11. Deployment and rollback

**Release layout:**

```text
/srv/<app>/releases/<sha>/    one directory per release, named by commit SHA (or timestamp)
/srv/<app>/current -> releases/<sha>
/srv/<app>/current/RELEASE    the SHA, build time, and operator — readable without git
```

**Deployment sequence:**

1. Deploy from a **clean, pushed, explicitly authorized commit**. The authorization names the host *and* the SHA. A new SHA needs a new authorization.
2. Build and run the full verification suite locally first — build, regression, server, security, dependency audit. A deployment script should do local verification only and never mutate a remote host as a side effect.
3. Create `releases/<sha>/`, write `RELEASE`, then **re-apply ownership and modes** (§7) — this is the step that gets forgotten.
4. **Back up the currently installed units and proxy config before replacing them**, along with an archive of the current release. Rollback is only fast if the previous state was captured *before* the change:
   ```bash
   ssh <host> 'tar czf /root/<app>-predeploy-$(date -u +%Y%m%dT%H%M%SZ).tar.gz \
     /etc/systemd/system/<app>*.service /etc/nginx/sites-available/<app>* /srv/<app>/current/'
   ```
5. **Test config before reloading anything.** `nginx -t` first, always; `systemctl daemon-reload` after unit changes. A syntax error found by `-t` is a non-event; the same error found by `reload` is an outage.
6. **Swap the symlink atomically, then verify, then roll back automatically if verification fails:**
   ```bash
   ln -sfn "/srv/<app>/releases/$NEW" /srv/<app>/current.new && mv -Tf /srv/<app>/current.new /srv/<app>/current
   systemctl restart <app>.service
   sleep 3
   if ! systemctl is-active --quiet <app>.service; then
     ln -sfn "/srv/<app>/releases/$PREV" /srv/<app>/current.new && mv -Tf /srv/<app>/current.new /srv/<app>/current
     systemctl restart <app>.service
     echo "rolled back to $PREV" >&2; exit 1
   fi
   ```
   `ln -sfn` + `mv -Tf` replaces the symlink in one rename rather than deleting and recreating it — there is no instant where `current` does not exist.
7. **`is-active` is not liveness** — see §6.1, where a dead process reported success. Add a real check: `ss -ltnp` shows the API on loopback only, and an actual request returns the expected status.
8. **Verify the perimeter from a network that is not the host** — a `curl` from the box itself proves nothing about the firewall or the proxy:
   ```bash
   curl -I  http://example.com/portal/                  # expect 301 to https
   curl -sSI https://example.com/portal/                # expect CSP, nosniff, Referrer-Policy, X-Frame-Options
   curl -sSI https://example.com/api/me                 # expect no-store + API security headers
   curl -sS --max-time 5 http://<host>:<port>/api/me    # MUST fail: no public API port
   ```
9. **Enable HSTS only after** the certificate is valid *and* `certbot renew --dry-run` passes. HSTS on a site whose renewal is broken is a self-inflicted outage with a max-age-long tail. Let the ACME client's shared options file own TLS protocol/session directives — duplicating them in the site file makes current generated configs invalid.
10. Record operator, timestamp, commit, commands, and result for every deploy, restart, and rollback.

---

## 12. Pre-launch checklist

Executable, in order. `[x]` requires recorded evidence — a command and its output — not a recollection.

**Application**
- [ ] `node --version` on the host meets the `engines` requirement.
- [ ] Production startup guards verified: refuses non-loopback `API_HOST`, non-HTTPS origins, insecure cookies, untrusted proxy, data inside the checkout or static root, half-configured provider credentials, zero configured sign-in methods.
- [ ] `ss -ltnp` shows the API on `127.0.0.1:<port>` (or `::1`) **only**; no public listener; no firewall rule exposing that port.
- [ ] `GET /api/auth/providers` lists exactly the configured methods; unconfigured providers are absent from the sign-in page.
- [ ] Secrets file present, mode `0640`, owner `root:<appuser>`, **no placeholder markers**, no value ever printed.
- [ ] `journalctl -u <app>` contains no secret values, tokens, magic links, or request bodies.

**Auth**
- [ ] At least one sign-in method completes **end-to-end on the production hostname** — not localhost, not staging.
- [ ] Registered redirect URI matches `new URL('/auth/<p>/callback', API_ORIGIN)` exactly; no wildcards; alternate hostnames redirect to the canonical origin before any sign-in begins.
- [ ] `prompt=select_account` (or the provider equivalent) is sent: sign out, sign in, confirm the account chooser appears; then repeat in a browser already holding a provider session.
- [ ] OAuth `state` is single-use and expires: replay a used callback URL → rejected. Wait past expiry → rejected. Strip the state cookie → rejected.
- [ ] Session cookie is `HttpOnly; Secure; SameSite=Lax`; CSRF cookie is readable and non-HttpOnly.
- [ ] Logout revokes server-side: capture the session cookie, log out, replay the cookie → 401.
- [ ] Logout is **reachable from the UI** — not merely implemented. Sign in as a normal user and sign out using only the interface, no devtools.
- [ ] The signed-in account control shows which identity is active (name and email), and is absent from public routes and from any timed-flow page. Assert both directions against built output.
- [ ] Idle and absolute expiry both enforced (fake-clock test) and idle ≤ absolute.
- [ ] A state-changing request without `X-CSRF-Token` → 403. With a foreign `Origin` → 403. With no `Origin` in production → 403.
- [ ] Cross-account access → 404/403 on every owned resource.
- [ ] **If access is meant to be restricted, an application-level allowlist is implemented and tested — the provider's test-user list is not it (§3).**

**Host**
- [ ] Unit files in the repository are correct on their own; **every host drop-in override deleted** and the behaviour re-verified from the repo units alone.
- [ ] `--preserve-symlinks-main` present; `RestrictSUIDSGID=false` on the unit that applies setgid modes; no `User=`/`Group=` on a unit that drops privileges internally.
- [ ] `stat` output recorded for `/var/lib/<app>` (`2750`), `/etc/<app>` (`0751`), each env file, and each ops script (ownership **and** mode).
- [ ] `nginx -t` passes; security headers, per-route rate limits, and the query-free log format are live; `/auth/` callback logging is suppressed.
- [ ] TLS valid; `certbot renew --dry-run` passes; **then** HSTS enabled and re-checked.
- [ ] Firewall permits public 80/443 only; SSH is key-only; unrelated services on the host were preserved and are still running.
- [ ] Log rotation installed for proxy logs; journal retention capped.

**Data**
- [ ] A **real** off-host encrypted backup has been produced by the timer, not by hand.
- [ ] A non-destructive restore drill passed against a **retrieved** backup: `integrity_check` ok, `foreign_key_check` empty, live database untouched, temporary key and restored file shredded. Record date, backup ID, and both results.
- [ ] Backup account cannot read the application's secrets file (test it: `runuser -u <backupuser> -- cat /etc/<app>/<app>.env` → permission denied).
- [ ] Rollback archive of the pre-deploy state exists and its path is recorded.

**Evidence**
- [ ] Full regression, server, security, and dependency-audit runs recorded for the exact release SHA.
- [ ] Every new regression assertion was confirmed to fail when its defect was reintroduced (§9).
- [ ] Absence sweep for private keys and secrets across working tree, **git history**, `/srv`, `/etc`, `/var/log`, and the journal (§7).
- [ ] Operator, timestamp, commit, and result recorded for the deployment.

---

## Provider status — what was and was not exercised

| Provider | Configured in code | Exercised in production | Notes |
|---|---|---|---|
| Google | Yes — PKCE, `prompt=select_account`, `email_verified` gate | **Yes** | The `select_account` and Testing-mode findings (§2, §3) come from this provider. |
| Email OTP & Magic Link (Resend) | Yes — 6-digit numeric OTP in-tab verification + fallback link, 10 min, Resend REST API | **Yes** | Universal email acceptance, disposable domain filtering, 6-digit in-tab verification to eliminate device shift, custom DNS domain verification (§4). |
| GitHub | Yes — PKCE, second call to `/user/emails` for a primary+verified address | **No** | Configured but never run against the live provider. Treat its notes as **untested**. |
| Facebook | Yes — state-protected, `pkce: false`, no verified-email assertion so it never auto-links by email | **No** | Configured but never run against the live provider. Treat its notes as **untested**. |

**Before relying on GitHub or Facebook, re-verify from scratch:** the authorization/exchange/profile endpoints and their current API versions, whether PKCE is supported, the exact shape of the profile response, whether a verified-email assertion exists, and which `authParams` (if any) force account selection. Do not assume the table above is current — provider endpoints and consent behaviour change, and these two rows have never been proven.

### Future providers — fill in when added

When a new provider is exercised in production, add a row above and a subsection here recording: the endpoint triple and API version; PKCE support and method; scope string; the verified-email assertion (or its absence); the account-chooser parameter and whether it was confirmed to work; any provider-specific access restriction and whether it was **empirically verified** (§3); rate-limit or quota behaviour observed; and the date and release SHA of the verification.

<!-- Template — copy, fill, and move the summary row into the table above.
### <provider>  (verified <YYYY-MM-DD>, release <sha>)
- Endpoints / API version:
- PKCE:
- Scope:
- Verified-email assertion:
- Account-chooser parameter (confirmed?):
- Access restriction, empirically verified:
- Surprises:
-->
