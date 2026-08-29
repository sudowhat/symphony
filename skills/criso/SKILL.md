---
name: criso
description: How to build, harden, and ship Criso — private, zero-third-party, cookie-free aggregate analytics reporting from query-free server logs. Load before implementing or operating privacy-preserving analytics in web portals.
---

# Criso — Private Aggregate Analytics & Operational Reporting

**What this solves:** traditional web analytics tools (Google Analytics, Mixpanel, Segment) require client-side tracking scripts, third-party cookies, intrusive privacy banners, and send user metadata to external servers. Criso provides **100% server-side, privacy-preserving, zero-tracker aggregate reporting** derived strictly from server logs with offline snapshot generation, cryptographic security, and role-based access control.

Reference implementation: `wisdom_capsules-folder` (`stats.js`, `analytics-report.js`, `quiz/web/js/criso.js`, `quiz/web/pages/criso.html`, `ops/nginx/wisdom-capsules.conf`, and `api-server.js`).

---

## 1. Architecture Overview

Criso operates in three distinct phases:

```text
1. TRAFFIC INGRESS (Real-time)
   Browser ──► Nginx (Query-free access log: /var/log/nginx/<app>.access.log)
               (No client JS tracking, no third-party scripts, no cookies required)

2. OFFLINE AGGREGATION (Hourly systemd Timer)
   systemd timer ──► stats.js ──► parses log window (e.g. 24h)
                                ──► categorizes routes & status codes
                                ──► writes /var/lib/<app>/criso-report.json (0640)

3. AUTHENTICATED DASHBOARD (On Demand)
   Admin Browser ──► /criso/ (Static HTML + criso.js)
                 ──► GET /api/criso/report
                 ──► API verifies Session + CRISO_ADMIN_EMAILS
                 ──► Returns precomputed aggregate JSON (No raw logs or IPs)
```

---

## 2. Privacy & Security Guarantees by Construction

1. **Zero Client-Side Trackers:** The public website runs zero analytics JavaScript. No beacons, no fingerprinters, no third-party network requests.
2. **Query-Free Ingress Logging:** Nginx must use a query-free log format (`$uri`, never `$request` or `$request_uri`) so tokens, OAuth codes, and parameters never enter logs:
   ```nginx
   log_format <app>_safe '$remote_addr [$time_local] "$request_method $uri $server_protocol" '
                         '$status $body_bytes_sent $request_time "$http_user_agent"';
   ```
3. **No Personal Data in Snapshot:** The generated JSON file contains only route counts, HTTP status totals, and aggregate request volumes. It **never** contains IP addresses, account IDs, cookies, query strings, user agents, or individual user timelines.
4. **Isolated Storage:** The snapshot file lives outside the application checkout and outside the public static root (e.g. `/var/lib/<app>/criso-report.json`), owned `root:<appuser>` or `<appuser>:<dbgroup>` with mode `0640`.
5. **No Self-Counting:** Nginx excludes `/criso/` and `/api/criso/report` from the access log (`access_log off;`) to prevent admin dashboard views from polluting the stats.

---

## 3. Server Log Ingestion & Route Normalization (`stats.js` & `analytics-report.js`)

### 3.1 Route Pattern Collapsing

Dynamic parameterized routes (such as user IDs, UUIDs, or attempt IDs) must be collapsed into canonical route classes before counting to prevent unbounded cardinality:

```js
function routePath(value) {
  const path = String(value || '');
  if (!path.startsWith('/')) return '/';
  if (/^\/api\/attempts\/[^/]+\/answers\/[^/]+$/.test(path)) return '/api/attempts/:attempt/answers/:question';
  if (/^\/api\/attempts\/[^/]+$/.test(path)) return '/api/attempts/:attempt';
  if (/^\/certificate\/[^/]+$/.test(path)) return '/certificate/:certificate';
  return path;
}
```

### 3.2 Page View Classification

Filter out static assets (`.css`, `.js`, `.png`, `.svg`, `.ico`, etc.) and internal API calls to calculate clean, meaningful human page views:

```js
function isStaticAsset(path) {
  return /\/(?:styles|js|images)\//.test(path) || /\.(?:css|js|map|png|jpe?g|gif|svg|ico|webp|woff2?|xml|txt|json)$/i.test(path);
}

function isSuccessfulPageRequest(method, status, path) {
  return method === 'GET' && status >= 200 && status < 300 && !path.startsWith('/api/') && !path.startsWith('/auth/') && !isStaticAsset(path);
}
```

### 3.3 Atomic Output Generation

Always write the snapshot to a temporary file (`.tmp`) and atomically rename it to prevent race conditions during API reads:

```js
const temporary = `${target}.${process.pid}.tmp`;
fs.writeFileSync(temporary, `${output}\n`, { encoding: 'utf8', mode: 0o640 });
fs.renameSync(temporary, target);
```

---

## 4. Backend Access Control (`api-server.js`)

Access to the Criso report endpoint must require both **authentication** and **explicit administrative authorization**:

```js
function crisoReportFor(user) {
  const email = String(user.email || '').trim().toLowerCase();
  if (!email || !crisoAdminEmails.has(email)) {
    throw new HttpError(403, 'This account is not permitted to view Criso.', 'criso_forbidden');
  }
  if (!crisoReportPath) {
    throw new HttpError(503, 'Criso reporting is not configured yet.', 'criso_unavailable');
  }
  if (production && (!path.isAbsolute(crisoReportPath) || !outsideStaticRoot(crisoReportPath) || !outsideApplicationRoot(crisoReportPath))) {
    throw new HttpError(503, 'Criso reporting is not configured safely.', 'criso_unavailable');
  }
  
  let report;
  try {
    report = JSON.parse(fs.readFileSync(path.resolve(crisoReportPath), 'utf8'));
  } catch {
    throw new HttpError(503, 'The latest Criso report is unavailable.', 'criso_unavailable');
  }

  // Strictly validate schema version before returning
  if (report?.schemaVersion !== 1 || typeof report.generatedAt !== 'string') {
    throw new HttpError(503, 'The latest Criso report is invalid.', 'criso_unavailable');
  }
  return report;
}
```

### Environment Configuration:
```ini
CRISO_REPORT_PATH=/var/lib/<app>/criso-report.json
CRISO_ADMIN_EMAILS=admin@example.com,lead@example.com
```

### 4.2 Assessment Performance & Question Insights

When the application includes exams, quizzes, or certifications, Criso enriches the aggregate report with database-backed completion and question analytics:

```js
function challengeExamStats() {
  const totalAttempts = stmt('SELECT COUNT(*) as c FROM attempts').get().c || 0;
  const completedAttempts = stmt("SELECT COUNT(*) as c FROM attempts WHERE status = 'completed'").get().c || 0;
  const activeAttempts = stmt("SELECT COUNT(*) as c FROM attempts WHERE status = 'active'").get().c || 0;
  const passedAttempts = stmt('SELECT COUNT(*) as c FROM attempts WHERE passed = 1').get().c || 0;
  const totalCertificates = stmt('SELECT COUNT(*) as c FROM certificates').get().c || 0;
  const totalQuestionsAnswered = stmt('SELECT COUNT(*) as c FROM attempt_answers WHERE selected_option_id IS NOT NULL').get().c || 0;
  const totalHintsUsed = stmt('SELECT COUNT(*) as c FROM attempt_answers WHERE hint_revealed_at IS NOT NULL').get().c || 0;

  // Top questions requiring hints
  const topHintedQuestions = stmt(`
    SELECT question_id, COUNT(*) as hints_used
    FROM attempt_answers WHERE hint_revealed_at IS NOT NULL
    GROUP BY question_id ORDER BY hints_used DESC, question_id ASC LIMIT 10
  `).all().map(r => ({ questionId: r.question_id, hintsUsed: Number(r.hints_used) }));

  // Questions left unanswered / skipped on completion
  const mostUnansweredQuestions = stmt(`
    SELECT aq.question_id, COUNT(*) as presented_count,
      SUM(CASE WHEN aa.selected_option_id IS NULL THEN 1 ELSE 0 END) as unanswered_count
    FROM attempt_questions aq JOIN attempts a ON a.id = aq.attempt_id
    LEFT JOIN attempt_answers aa ON aa.attempt_id = aq.attempt_id AND aa.question_id = aq.question_id
    WHERE a.status IN ('completed', 'expired')
    GROUP BY aq.question_id HAVING unanswered_count > 0
    ORDER BY unanswered_count DESC, presented_count DESC LIMIT 10
  `).all().map(r => ({ questionId: r.question_id, unansweredCount: Number(r.unanswered_count), presentedCount: Number(r.presented_count) }));

  return {
    totalAttempts, completedAttempts, activeAttempts, passedAttempts,
    passRate: completedAttempts > 0 ? Math.round((passedAttempts / completedAttempts) * 100) : 0,
    totalCertificates, totalQuestionsAnswered, totalHintsUsed,
    topHintedQuestions, mostUnansweredQuestions
  };
}
```

---

## 5. Systemd Automation Units

### Service (`/etc/systemd/system/<app>-criso.service`)
```ini
[Unit]
Description=Criso aggregate analytics snapshot generator
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/node /srv/<app>/current/stats.js --log /var/log/nginx/<app>.access.log --hours 24 --json --output /var/lib/<app>/criso-report.json
ExecStartPost=/bin/chown <appuser>:<dbgroup> /var/lib/<app>/criso-report.json
ExecStartPost=/bin/chmod 0640 /var/lib/<app>/criso-report.json
UMask=0027
NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectHome=true
ProtectSystem=strict
ReadOnlyPaths=/srv/<app>/current /var/log/nginx
ReadWritePaths=/var/lib/<app>
StandardOutput=journal
StandardError=journal
```

### Timer (`/etc/systemd/system/<app>-criso.timer`)
```ini
[Unit]
Description=Hourly Criso aggregate analytics refresh

[Timer]
OnCalendar=hourly
RandomizedDelaySec=5m
Persistent=true

[Install]
WantedBy=timers.target
```

---

## 6. Nginx Web Server Configuration

```nginx
# Private, unlisted aggregate reporting
location = /criso {
  access_log off;
  return 301 /criso/;
}

location /criso/ {
  access_log off;
  if ($request_method !~ ^(GET|HEAD)$) { return 405; }
  expires -1;
  add_header X-Robots-Tag "noindex, nofollow, noarchive" always;
  try_files $uri $uri/ $uri/index.html =404;
}

location = /api/criso/report {
  limit_req zone=wc_api_ip burst=20 nodelay;
  access_log off;
  proxy_pass http://127.0.0.1:<port>;
  proxy_http_version 1.1;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $remote_addr;
  proxy_set_header X-Forwarded-Proto https;
}
```

---

## 7. Verification Checklist

- [ ] `/criso` redirects to `/criso/`.
- [ ] `/criso/` is absent from sitemap.xml and main navigation.
- [ ] `/criso/` sends `X-Robots-Tag: noindex, nofollow, noarchive`.
- [ ] `GET /api/criso/report` returns 401 when unauthenticated.
- [ ] `GET /api/criso/report` returns 403 for an authenticated user not in `CRISO_ADMIN_EMAILS`.
- [ ] `GET /api/criso/report` returns 200 with schema-validated aggregate data for an admin user.
- [ ] Criso snapshot file is located at `/var/lib/<app>/criso-report.json` (outside checkout) with mode `0640`.
- [ ] Systemd timer is active (`systemctl list-timers <app>-criso.timer`).
- [ ] Zero raw IP addresses, user agents, or query parameters exist in the JSON snapshot.
