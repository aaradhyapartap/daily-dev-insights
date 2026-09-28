# 📌 Web security: XSS, CSRF, SQL injection
*September 28, 2026 · Daily Dev Insight*

## 🧠 Overview

These three vulnerabilities have been in the OWASP Top 10 for over two decades, yet they still plague modern applications. Why? Because security is often treated as an afterthought rather than a fundamental design principle. XSS (Cross-Site Scripting) allows attackers to inject malicious scripts into your pages, CSRF (Cross-Site Request Forgery) tricks users into performing unintended actions, and SQL injection lets attackers manipulate your database queries. Each exploits a different trust boundary in your application.

The common thread? Trusting user input. Every piece of data entering your system—from form fields to URL parameters to cookies—is potentially hostile. The developer who assumes "my users wouldn't do that" has already lost. Modern frameworks provide some protection out of the box, but understanding these attacks deeply is the only way to know when you're stepping outside those guardrails.

The good news: these are solved problems with well-established countermeasures. The bad news: you have to actually implement them, and implementation details matter enormously.

## 💡 Key Concepts

- **Context-aware output encoding**: Data that's safe in HTML is not safe in JavaScript, and vice versa. Always encode for the specific context where data will be rendered.

- **Parameterized queries are non-negotiable**: String concatenation with user input in SQL queries is malpractice, full stop. ORMs and prepared statements exist—use them religiously.

- **CSRF tokens validate origin**: These random tokens prove that form submissions actually originated from your application, not from an attacker's site making cross-origin requests.

- **Defense in depth wins**: No single countermeasure is perfect. Combine input validation, output encoding, CSP headers, SameSite cookies, and HTTPS to create overlapping layers of protection.

- **The principle of least privilege**: Database users should only have permissions they absolutely need. Your web app probably doesn't need DROP TABLE rights.

## 🐍 Python Example

```python
from flask import Flask, request, session, render_template_string
import psycopg2
from psycopg2 import sql
from secrets import token_urlsafe
from markupsafe import escape

app = Flask(__name__)
app.secret_key = 'your-secret-key-here'

# Bad: SQL Injection vulnerable
def get_user_bad(username):
    conn = psycopg2.connect("dbname=mydb")
    cur = conn.cursor()
    # Never do this! Attacker could send: ' OR '1'='1
    query = f"SELECT * FROM users WHERE username = '{username}'"
    cur.execute(query)
    return cur.fetchone()

# Good: Parameterized query
def get_user_safe(username):
    conn = psycopg2.connect("dbname=mydb")
    cur = conn.cursor()
    # The library handles escaping/sanitization
    cur.execute("SELECT * FROM users WHERE username = %s", (username,))
    return cur.fetchone()

# XSS Protection: Always escape user content
@app.route('/profile/<username>')
def profile(username):
    # escape() prevents script injection
    safe_username = escape(username)
    return render_template_string(
        '<h1>Profile: {{ username }}</h1>',
        username=safe_username
    )

# CSRF Protection: Validate tokens
@app.route('/transfer', methods=['POST'])
def transfer_money():
    # Check CSRF token before processing sensitive actions
    submitted_token = request.form.get('csrf_token')
    if not submitted_token or submitted_token != session.get('csrf_token'):
        return "CSRF validation failed", 403
    
    # Process the legitimate request
    amount = request.form.get('amount')
    return f"Transferred ${amount}"

# Generate CSRF token for forms
@app.before_request
def generate_csrf_token():
    if 'csrf_token' not in session:
        session['csrf_token'] = token_urlsafe(32)
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const csrf = require('csurf');
const cookieParser = require('cookie-parser');
const DOMPurify = require('isomorphic-dompurify');
const mysql = require('mysql2/promise');

const app = express();
app.use(express.json());
app.use(cookieParser());

// CSRF protection middleware
const csrfProtection = csrf({ cookie: true });

// Database connection pool
const pool = mysql.createPool({
  host: 'localhost',
  user: 'webapp',
  database: 'mydb',
  waitForConnections: true
});

// Bad: SQL Injection vulnerable
async function searchProductsBad(searchTerm) {
  const conn = await pool.getConnection();
  // Attacker could inject: '; DROP TABLE products; --
  const query = `SELECT * FROM products WHERE name = '${searchTerm}'`;
  const [rows] = await conn.execute(query);
  return rows;
}

// Good: Parameterized query
async function searchProductsSafe(searchTerm) {
  const conn = await pool.getConnection();
  // Placeholders prevent injection
  const [rows] = await conn.execute(
    'SELECT * FROM products WHERE name = ?',
    [searchTerm]
  );
  return rows;
}

// XSS Protection: Sanitize user-generated content
app.post('/api/comment', csrfProtection, (req, res) => {
  const rawComment = req.body.comment;
  
  // DOMPurify removes malicious scripts while preserving safe HTML
  const sanitizedComment = DOMPurify.sanitize(rawComment, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong'],
    ALLOWED_ATTR: []
  });
  
  // Store sanitized version
  res.json({ comment: sanitizedComment });
});

// CSRF-protected endpoint
app.post('/api/delete-account', csrfProtection, async (req, res) => {
  // Token automatically validated by middleware
  // Only requests with valid token from same origin succeed
  const userId = req.body.userId;
  await deleteUserAccount(userId);
  res.json({ success: true });
});

// Provide CSRF token to client
app.get('/api/csrf-token', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});
```

## ⚖️ When To Use / When To Avoid

**Always Use These Protections:**
- ✅ Any application handling user input (spoiler: that's all of them)
- ✅ Forms performing state-changing operations (POST/PUT/DELETE)
- ✅ Displaying user-generated content from databases
- ✅ Building SQL queries with dynamic parameters

**Extra Vigilance Required:**
- ⚠️ Rich text editors and comment systems (XSS paradise if mishandled)
- ⚠️ Single-page applications (CSRF token management is more complex)
- ⚠️ APIs with mixed authentication schemes (cookies + tokens = confusion)
- ⚠️ Legacy codebases without framework protections

## 📚 Further Reading

- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) - Comprehensive context-specific encoding guide
- [Content Security Policy Reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) - MDN's guide to CSP headers as XSS defense in depth
- [SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) - OWASP's definitive guide to parameterized queries
- [SameSite Cookie Attribute](https://web.dev/samesite-cookies-explained/) - Modern browser-level CSRF protection
- [Bobby Tables: