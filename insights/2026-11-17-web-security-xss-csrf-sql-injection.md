# 📌 Web security: XSS, CSRF, SQL injection
*November 17, 2026 · Daily Dev Insight*

## 🧠 Overview

Web security vulnerabilities are like leaving your house keys under the doormat—everyone knows to check there. XSS (Cross-Site Scripting), CSRF (Cross-Site Request Forgery), and SQL injection remain in OWASP's top 10 not because they're sophisticated, but because developers still make the same preventable mistakes. These attacks exploit the fundamental trust relationships in web applications: trust in user input, trust in the browser, and trust in database queries.

Here's the reality: most developers understand these concepts theoretically but fail to implement defenses consistently. You might sanitize inputs in your authentication flow but forget about that obscure admin endpoint. You might use parameterized queries in production code but write raw SQL in migration scripts. The difference between a secure application and a breached one often comes down to muscle memory—making defensive coding your default, not an afterthought.

The good news? Modern frameworks have made these attacks harder to execute if you use them correctly. The bad news? One custom implementation, one "quick fix," or one copied Stack Overflow snippet without understanding context can undo all those protections. Let's look at how to defend against these attacks with practical, production-ready code.

## 💡 Key Concepts

- **XSS exploits trust in content**: When you render user input as HTML/JavaScript without sanitization, attackers can inject malicious scripts that run in victims' browsers, stealing cookies, session tokens, or performing actions on their behalf.

- **CSRF exploits trust in the user**: Browsers automatically send cookies with requests, so attackers can trick authenticated users into making unwanted requests to your application from malicious sites. Defense requires verifying request origin through CSRF tokens.

- **SQL injection exploits trust in queries**: Concatenating user input directly into SQL queries lets attackers manipulate database logic, potentially reading, modifying, or deleting data. Parameterized queries separate code from data, making injection impossible.

- **Defense in depth matters**: No single technique prevents all attacks. Combine input validation, output encoding, CSP headers, parameterized queries, CSRF tokens, and principle of least privilege for comprehensive protection.

- **Context determines encoding**: HTML context needs HTML encoding, JavaScript context needs JavaScript escaping, and URL context needs URL encoding. Using the wrong escape method can still leave you vulnerable.

## 🐍 Python Example

```python
from flask import Flask, request, render_template_string, session
import secrets
import sqlite3
from markupsafe import escape
from functools import wraps

app = Flask(__name__)
app.secret_key = secrets.token_hex(32)

# CSRF protection decorator
def csrf_protect(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if request.method == "POST":
            token = session.get('csrf_token')
            if not token or token != request.form.get('csrf_token'):
                return "CSRF validation failed", 403
        return f(*args, **kwargs)
    return decorated_function

@app.route('/comment', methods=['GET', 'POST'])
@csrf_protect
def handle_comment():
    if request.method == 'GET':
        # Generate CSRF token for form
        session['csrf_token'] = secrets.token_hex(16)
        return render_template_string('''
            <form method="post">
                <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
                <input name="comment" placeholder="Your comment">
                <button>Submit</button>
            </form>
        ''', csrf_token=session['csrf_token'])
    
    # XSS prevention: escape user input before rendering
    user_comment = escape(request.form.get('comment', ''))
    
    # SQL injection prevention: use parameterized queries
    conn = sqlite3.connect('comments.db')
    cursor = conn.cursor()
    
    # SAFE: Parameters are properly escaped by the driver
    cursor.execute(
        "INSERT INTO comments (text, user_id) VALUES (?, ?)",
        (user_comment, session.get('user_id', 'anonymous'))
    )
    
    # UNSAFE (NEVER DO THIS):
    # cursor.execute(f"INSERT INTO comments (text) VALUES ('{user_comment}')")
    
    conn.commit()
    conn.close()
    
    return f"Comment saved: {user_comment}"

if __name__ == '__main__':
    app.run(debug=False)  # Never run debug=True in production
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const csrf = require('csurf');
const cookieParser = require('cookie-parser');
const { Pool } = require('pg');
const DOMPurify = require('isomorphic-dompurify');

const app = express();
const csrfProtection = csrf({ cookie: true });
const db = new Pool({ connectionString: process.env.DATABASE_URL });

app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());

// XSS prevention: Content Security Policy header
app.use((req, res, next) => {
  res.setHeader(
    "Content-Security-Policy",
    "default-src 'self'; script-src 'self'; object-src 'none';"
  );
  next();
});

app.get('/profile', csrfProtection, (req, res) => {
  res.send(`
    <form method="POST" action="/update-profile">
      <input type="hidden" name="_csrf" value="${req.csrfToken()}">
      <input name="bio" placeholder="Your bio">
      <button>Update</button>
    </form>
  `);
});

app.post('/update-profile', csrfProtection, async (req, res) => {
  const userId = req.session?.userId;
  const userBio = req.body.bio;
  
  // XSS prevention: sanitize HTML input
  const sanitizedBio = DOMPurify.sanitize(userBio, { 
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong'],
    ALLOWED_ATTR: [] 
  });
  
  try {
    // SQL injection prevention: parameterized query
    await db.query(
      'UPDATE users SET bio = $1 WHERE id = $2',
      [sanitizedBio, userId]  // Parameters safely escaped
    );
    
    res.send(`Profile updated: ${sanitizedBio}`);
  } catch (error) {
    res.status(500).send('Update failed');
  }
});

app.listen(3000);
```

## ⚖️ When To Use / When To Avoid

**Always Use These Defenses:**
- ✅ Parameterized queries for ALL database operations (no exceptions)
- ✅ CSRF tokens on state-changing endpoints (POST, PUT, DELETE)
- ✅ Output encoding/escaping when rendering user content
- ✅ CSP headers to restrict script sources
- ✅ HTTPOnly and Secure flags on session cookies

**Don't Rely On:**
- ❌ Client-side validation alone (always validate server-side)
- ❌ Blacklist filtering for XSS (attackers find bypasses)
- ❌ Referer header checking for CSRF (easily spoofed)
- ❌ "Sanitizing" SQL inputs manually (use parameterized queries)
- ❌ Security through obscurity (assume attackers know your stack)

## 📚 Further Reading

- [OWASP Top 10 Web Application Security Risks](https://owasp.org/www-project-top-ten/) - Comprehensive guide to the most critical security risks including detailed prevention strategies
- [Content Security Policy Reference on MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) - Deep dive into CSP headers for XSS prevention
- [PostgreSQL Documentation: SQL Injection Prevention](https://www.postgresql.org/docs/current/sql-prepare.html) - Database-level security best practices and parameterized query examples
- [Flask Security Considerations](https://flask.palletsprojects.com/en