# 🔐 INT308 – Lab 1: Session Management Vulnerabilities

![Security](https://img.shields.io/badge/Category-Web%20Security-red?style=flat-square)
![Tools](https://img.shields.io/badge/Tools-Burp%20Suite%20%7C%20DVWA%20%7C%20Firefox%20DevTools-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Kali%20Linux%20%7C%20VirtualBox-purple?style=flat-square)
![OWASP](https://img.shields.io/badge/OWASP-A07%3A2021%20Auth%20Failures-orange?style=flat-square)

---

## 📌 Overview

This lab investigated **session management vulnerabilities** in web applications using **DVWA (Damn Vulnerable Web Application)** running locally on Kali Linux inside VirtualBox at `http://127.0.0.1:42001`.

Three critical session vulnerabilities were identified, demonstrated, and documented with full proof-of-concept screenshots across all three DVWA security levels (Low, Medium, High).

---

## 🎯 Objectives

- Inspect and analyze session cookie attributes using Firefox DevTools
- Simulate a **Session Fixation** attack by planting a known session ID before login
- Perform a **Session Hijacking** attack using Burp Suite Community Edition as an intercepting proxy
- Confirm full session takeover without valid credentials

---

## 🛠️ Tools & Environment

| Tool | Purpose |
|------|---------|
| Kali Linux (VirtualBox) | Attack and testing environment |
| DVWA (`127.0.0.1:42001`) | Vulnerable target web application |
| Firefox DevTools (F12) | Cookie inspection and manipulation |
| Burp Suite Community Edition | HTTP interception and session token replay |
| Notepad (Text Editor) | Capturing session IDs during fixation attack |

---

## 🧪 Exercises

---

### Exercise 1 — Analyzing Session Cookie Attributes

**Steps taken:**
1. Logged into DVWA using default credentials (`admin` / `password`)
2. Set DVWA security level to **Low**
3. Opened Firefox DevTools (F12) → Storage tab → Cookies
4. Inspected the `PHPSESSID` cookie attributes in detail
5. Repeated inspection at **Medium** security level to compare behavior

**Findings — Cookie: `PHPSESSID`**

| Attribute | Observed Value | Risk |
|-----------|---------------|------|
| `HttpOnly` | `false` | JavaScript can read the cookie — XSS can steal the session |
| `Secure` | `false` | Cookie transmitted in plain text — vulnerable to sniffing |
| `SameSite` | `None` | CSRF attacks possible from cross-site requests |
| `Expiry` | Session-based | Expires on browser close |

**Key finding:** Even at **Medium** security level, all three insecure cookie attributes remained unchanged — confirming DVWA applies no meaningful session hardening across security levels.

**XSS Proof-of-Concept:**  
Injected `<script>alert("You have been hacked")</script>` into the XSS (Reflected) input field. The alert executed successfully, demonstrating that the missing `HttpOnly` flag allows JavaScript to interact with session data.  
At Medium level, the filter was bypassed using: `<scr<script>ipt>alert("You have been hacked")</script>`

**Mitigation:**
- Set `HttpOnly=true` — blocks JavaScript from reading cookies
- Set `Secure=true` — forces cookie transmission over HTTPS only
- Set `SameSite=Strict` — prevents cross-site request inclusion
- Enforce HTTPS across the entire application

---

### Exercise 2 — Session Fixation Attack

**What is Session Fixation?**  
An attacker plants a known session ID in the victim's browser *before* they log in. If the server does not regenerate the session ID after authentication, the attacker's pre-planted token becomes a valid authenticated session — giving them access without credentials.

**Steps taken:**
1. Opened the DVWA login page — noted the pre-login `PHPSESSID` via DevTools Storage tab
2. Copied the session ID value to a text editor
3. Manually changed the `PHPSESSID` value to `HACKERSESS001` in the DevTools Storage panel — simulating what an attacker delivers via a malicious link or XSS payload
4. Logged in normally with credentials `admin` / `password`
5. After successful login, returned to DevTools — the `PHPSESSID` value was **still `HACKERSESS001`**
6. Opened a private browser window (Ctrl+Shift+P) → navigated to `http://127.0.0.1:42001/index.php`
7. In the Console tab, injected the fixed session token:
```javascript
document.cookie = "PHPSESSID=HACKERSESS001; path=/";
location.reload();
```
8. ✅ **DVWA dashboard loaded as admin — full session takeover confirmed without entering any credentials**

**Root Cause:**  
DVWA's login handler never calls `session_regenerate_id()` at any point during authentication. The pre-authentication and post-authentication states shared the same session identifier, eliminating any security boundary.

**Mitigation:**

| Control | Implementation | Defeats |
|---------|---------------|---------|
| Regenerate session on login | `session_regenerate_id(true)` called immediately after auth | Session fixation (primary fix) |
| HttpOnly cookie flag | `session_set_cookie_params(['httponly' => true])` | XSS-based session theft |
| Secure cookie flag | `session_set_cookie_params(['secure' => true])` | Network sniffing |
| SameSite=Strict | `session_set_cookie_params(['samesite' => 'Strict'])` | CSRF and cross-site attacks |
| Disable URL session IDs | `ini_set('session.use_only_cookies', 1)` | URL-delivered fixation |
| Session timeout | Absolute and idle timeouts enforced server-side | Persistent session attacks |
| Session binding | Bind session to IP + User-Agent on login | Hijack from different client |
| Destroy on logout | `session_destroy()` + `unset($_SESSION)` | Session reuse after logout |

> **Note:** The `true` parameter in `session_regenerate_id(true)` is critical — it deletes the old session file server-side, not just on the client.

---

### Exercise 3 — Session Hijacking Using Burp Suite

**Steps taken:**
1. Configured Firefox to route all traffic through Burp Suite proxy (`127.0.0.1:8080`)
2. Enabled **Intercept is ON** in Burp Suite → Proxy tab
3. Entered credentials on the DVWA login page and clicked Login — browser froze, confirming Burp intercepted the POST request to `login.php`
4. In the Burp Response tab, located the `Cookie` header and changed the `PHPSESSID` value to the attacker-controlled token: `HACKERSESS001`
5. Forwarded the modified request
6. In the **Repeater tab**, confirmed the Cookie header contained `PHPSESSID=HACKERSESS001` and clicked **Send**
7. ✅ **Server returned HTTP 200 OK with the full DVWA authenticated dashboard HTML — session hijack confirmed**

**Key Observation:**  
The server performed zero additional validation beyond checking whether the session ID existed in its session store. No IP binding, no User-Agent verification, no token rotation — making token replay trivially effective.

**Mitigation:**
- Enforce HTTPS site-wide to prevent token interception in transit
- Implement server-side session binding (IP + User-Agent)
- Use short session expiry windows with inactivity timeouts
- Implement anomaly detection for sessions used from multiple IPs simultaneously

---

## 📋 Summary

| Exercise | Vulnerability | Severity | Key Evidence |
|----------|--------------|----------|-------------|
| 1 – Cookie Analysis | Missing HttpOnly, Secure, SameSite flags | 🔴 High | XSS alert executed; cookies visible in DevTools |
| 2 – Session Fixation | No `session_regenerate_id()` on login | 🔴 High | `HACKERSESS001` persisted post-login; dashboard accessed without credentials |
| 3 – Session Hijacking | No server-side session validation | 🔴 Critical | HTTP 200 OK returned with full dashboard on replayed token |

---

## 📄 Full Report

📎 [`INT308_Lab1_Report_XypherSec.pdf`](./INT308_Lab1_Report_XypherSec.pdf)

The full report includes step-by-step screenshots of every exercise, cookie attribute analysis across DVWA security levels, and detailed mitigation tables.

---

## 📚 References

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP Top 10 – A07:2021 Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
- [PortSwigger Web Security Academy – Session Management](https://portswigger.net/web-security/authentication)
- [DVWA Documentation](https://dvwa.co.uk/)

---

## 👤 Author

**Christopher** — Cybersecurity Analyst  
**XypherSec**  
Enugu State, Nigeria  
Phase 1 Cybersecurity & Ethical Hacking Internship — INT308
