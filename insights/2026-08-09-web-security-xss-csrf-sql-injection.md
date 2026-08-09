# 📌 Web security: XSS, CSRF, SQL injection
*August 09, 2026 · Daily Dev Insight*

## 🧠 Overview

Web security vulnerabilities are like leaving your house keys under the doormat—they're often invisible to the homeowner but glaringly obvious to attackers. XSS (Cross-Site Scripting), CSRF (Cross-Site Request Forgery), and SQL injection represent the "big three" of web application vulnerabilities, consistently appearing in OWASP's top 10 for nearly two decades. Despite countless frameworks and libraries designed to prevent them, these vulnerabilities persist because they exploit fundamental trust relationships: XSS abuses trust in user-supplied data, CSRF exploits trust in authenticated users, and SQL injection capitalizes on trust in database query construction.

What makes these attacks particularly dangerous isn't just their technical impact—it's their accessibility. You don't need advanced hacking skills to execute a basic SQL injection; you just need to understand where user input meets system processing without proper validation. The real challenge for developers isn't learning what these attacks are (most of us know), but building security-conscious habits that make these vulnerabilities architecturally impossible rather than just "checked for." Modern frameworks have made tremendous progress here, but they can only protect you if you use them correctly.

The harsh reality is that every text input, every URL parameter, and every cookie in your application is a potential attack vector. Security isn't a feature you bolt on at the end—it's a mindset you adopt from the first line of code. Let's look at how these attacks work and, more importantly, how to defend against them in production code.

## 💡 Key Concepts

- **XSS allows attackers to inject malicious scripts** into web pages viewed by other users, typically by exploiting insufficient input sanitization. There are three types: stored (persistent in database), reflected (in URL/form submission), and DOM-based (client-side JavaScript manipulation).

- **CSRF tricks authenticated users into performing unwanted actions** by exploiting the browser's automatic inclusion of session cookies. The attack works because your server can't distinguish between legitimate user requests and forged ones from malicious sites.

- **SQL injection occurs when untrusted data is concatenated into SQL queries** without proper parameterization, allowing attackers to manipulate query logic, exfiltrate data, or even execute system commands depending on database permissions.

- **Defense-in-depth is essential**—never rely on a single protection mechanism. Use parameterized queries AND input validation, implement CSRF tokens AND SameSite cookies, escape output AND use Content Security Policy headers.

- **Context matters for encoding**—data safe in one context (HTML body) may be dangerous in another (JavaScript string, URL parameter, CSS). Always encode based on where the data will be used, not where it came from.

## 🐍 Python Example

```python
from flask import Flask, request, render_template_string, session
from flask_wtf.csrf import CSRFProtect
import secrets
import sqlite3
from markupsafe import escape

app = Flask(__name__)
app.config['SECRET_KEY'] = secrets.token_hex(32)
csrf = CSRFProtect(app)

# WRONG: Vulnerable to SQL injection
def vulnerable_login(username, password):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    # Never do this! Concatenating user input into SQL
    query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
    cursor.execute(query)
    return cursor.fetchone()

# RIGHT: Using parameterized queries
def safe_login(username, password):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    # Parameterized queries prevent SQL injection
    query = "SELECT * FROM users WHERE username=? AND password=?"
    cursor.execute(query, (username, password))
    return cursor.fetchone()

# WRONG: Vulnerable to XSS
@app.route('/comment/unsafe', methods=['POST'])
def unsafe_comment():
    comment = request.form.get('comment')
    # Directly inserting user input into HTML
    return render_template_string(f"<div>Comment: {comment}</div>")

# RIGHT: Proper escaping
@app.route('/comment/safe', methods=['POST'])
@csrf.exempt  # Remove this - just for demo
def safe_comment():
    comment = request.form.get('comment')
    # escape() prevents XSS by encoding HTML special characters
    safe_comment = escape(comment)
    return render_template_string(f"<div>Comment: {safe_comment}</div>")

# CSRF protection is automatically applied via CSRFProtect
# Templates must include: {{ csrf_token() }}
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const csrf = require('csurf');
const cookieParser = require('cookie-parser');
const helmet = require('helmet');
const { body, validationResult } = require('express-validator');

const app = express();
app.use(helmet()); // Sets security headers including CSP
app.use(cookieParser());
app.use(express.json());

// CSRF protection middleware
const csrfProtection = csrf({ cookie: true });

// WRONG: Vulnerable to XSS
app.get('/search/unsafe', (req, res) => {
  const query = req.query.q;
  // Directly embedding user input in HTML
  res.send(`<h1>Results for: ${query}</h1>`);
});

// RIGHT: Proper sanitization and Content Security Policy
app.get('/search/safe', (req, res) => {
  const query = req.query.q;
  // HTML-encode user input
  const safeQuery = query
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
  
  res.send(`<h1>Results for: ${safeQuery}</h1>`);
});

// CSRF-protected endpoint
app.post('/transfer', 
  csrfProtection,
  body('amount').isInt({ min: 1 }),
  body('recipient').isEmail(),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    
    // Process the validated, CSRF-protected transfer
    const { amount, recipient } = req.body;
    // ... actual transfer logic
    res.json({ success: true });
  }
);

// Serve CSRF token to clients
app.get('/form', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});
```

## ⚖️ When To Use / When To Avoid

**Always Use These Protections:**
- ✅ Parameterized queries for ALL database operations—no exceptions
- ✅ CSRF tokens on state-changing operations (POST, PUT, DELETE)
- ✅ Context-appropriate output encoding for any user-supplied data
- ✅ Content Security Policy headers to restrict script sources
- ✅ SameSite cookie attributes (`Strict` or `Lax`) for session cookies

**Common Mistakes To Avoid:**
- ❌ Don't rely solely on client-side validation—it's easily bypassed
- ❌ Don't use blacklist filtering for XSS (there are infinite bypass techniques)
- ❌ Don't disable CSRF protection for "convenience" on REST APIs with cookie auth
- ❌ Don't trust data just because it came from your own database (stored XSS exists)
- ❌ Don't roll your own sanitization—use battle-tested libraries

## 📚 Further Reading

- [OWASP Top Ten Web Application Security Risks](https://owasp.org/www-project-top-ten/) - The definitive guide to current web vulnerabilities with real-world examples
- [MDN Web Security Guidelines](https://developer.mozilla.org/en-US/docs/Web/Security) - Comprehensive documentation on Content Security Policy, CORS, and browser security features
- [PortSwigger Web Security Academy