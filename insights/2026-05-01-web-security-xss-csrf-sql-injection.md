# 📌 Web security: XSS, CSRF, SQL injection
*May 01, 2026 · Daily Dev Insight*

## 🧠 Overview

Web security vulnerabilities like XSS (Cross-Site Scripting), CSRF (Cross-Site Request Forgery), and SQL injection remain the most common attack vectors despite being well-understood for decades. These aren't just theoretical concerns—they're actively exploited in the wild and can lead to complete application compromise, data breaches, and regulatory nightmares.

The frustrating reality is that these vulnerabilities persist not because they're technically complex to prevent, but because developers often implement security as an afterthought or rely on incomplete solutions. Modern frameworks provide excellent built-in protections, but they require deliberate activation and proper configuration. Understanding the attack mechanisms isn't just academic—it's essential for recognizing when your defenses might have gaps.

The good news? With proper input validation, output encoding, parameterized queries, and CSRF tokens, you can eliminate 90% of these risks. The key is building security into your development workflow from day one, not bolting it on during the security audit.

## 💡 Key Concepts

• **XSS exploits trust in user input** - Malicious scripts execute in victims' browsers when applications fail to properly sanitize or encode user-provided content before rendering it in HTML
• **CSRF exploits trust in user identity** - Attackers trick authenticated users into unknowingly submitting malicious requests by leveraging existing session cookies or authentication tokens
• **SQL injection exploits trust in data queries** - Unsanitized input gets interpreted as SQL commands rather than data, allowing attackers to manipulate database queries and access unauthorized information
• **Defense in depth is essential** - No single security measure is foolproof; combine input validation, output encoding, parameterized queries, CSP headers, and CSRF tokens for comprehensive protection
• **Framework defaults matter** - Modern web frameworks often include security features, but they must be explicitly enabled and properly configured to be effective

## 🐍 Python Example

```python
from flask import Flask, request, render_template_string, session, redirect
import sqlite3
import secrets
from markupsafe import escape
from functools import wraps

app = Flask(__name__)
app.secret_key = secrets.token_hex(16)

def csrf_protect(f):
    """Decorator to protect routes from CSRF attacks"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if request.method == 'POST':
            token = request.form.get('csrf_token')
            if not token or token != session.get('csrf_token'):
                return "CSRF token missing or invalid", 403
        return f(*args, **kwargs)
    return decorated_function

def get_db():
    """Get database connection with proper configuration"""
    conn = sqlite3.connect('app.db')
    conn.execute('CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)')
    return conn

@app.route('/')
def index():
    # Generate CSRF token for forms
    if 'csrf_token' not in session:
        session['csrf_token'] = secrets.token_hex(16)
    
    # Safe way to display user input - using escape()
    user_name = request.args.get('name', '')
    safe_name = escape(user_name)  # Prevents XSS
    
    template = '''
    <h1>Hello {{ name }}!</h1>
    <form method="POST" action="/submit">
        <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
        <input type="text" name="email" placeholder="Enter email">
        <button type="submit">Submit</button>
    </form>
    '''
    return render_template_string(template, name=safe_name, csrf_token=session['csrf_token'])

@app.route('/submit', methods=['POST'])
@csrf_protect
def submit_data():
    """Secure form submission with SQL injection protection"""
    email = request.form.get('email', '').strip()
    
    if not email:
        return "Email required", 400
    
    # Safe database query using parameterized statements
    conn = get_db()
    try:
        # This prevents SQL injection by separating SQL code from data
        conn.execute("INSERT INTO users (name, email) VALUES (?, ?)", 
                    ("Anonymous", email))
        conn.commit()
        return "User created successfully!"
    except sqlite3.Error as e:
        return f"Database error occurred", 500
    finally:
        conn.close()

if __name__ == '__main__':
    app.run(debug=False)  # Never run debug=True in production
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const csrf = require('csurf');
const cookieParser = require('cookie-parser');
const session = require('express-session');
const { body, validationResult } = require('express-validator');
const mysql = require('mysql2/promise');
const he = require('he'); // HTML entity encoder

const app = express();

// Security middleware setup
app.use(cookieParser());
app.use(express.urlencoded({ extended: true }));
app.use(session({
    secret: process.env.SESSION_SECRET || 'dev-secret-change-in-prod',
    resave: false,
    saveUninitialized: false,
    cookie: { 
        secure: process.env.NODE_ENV === 'production', // HTTPS only in prod
        httpOnly: true, // Prevents XSS access to cookies
        maxAge: 3600000 // 1 hour
    }
}));

// CSRF protection middleware
const csrfProtection = csrf({ cookie: true });
app.use(csrfProtection);

// Database connection pool
const dbPool = mysql.createPool({
    host: process.env.DB_HOST || 'localhost',
    user: process.env.DB_USER || 'root',
    password: process.env.DB_PASSWORD || '',
    database: process.env.DB_NAME || 'testdb',
    waitForConnections: true,
    connectionLimit: 10
});

// Content Security Policy middleware to prevent XSS
app.use((req, res, next) => {
    res.setHeader('Content-Security-Policy', 
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'");
    next();
});

app.get('/', (req, res) => {
    const userComment = req.query.comment || '';
    // Proper output encoding prevents XSS
    const safeComment = he.encode(userComment);
    
    res.send(`
        <h1>Comment: ${safeComment}</h1>
        <form method="POST" action="/comment">
            <input type="hidden" name="_csrf" value="${req.csrfToken()}">
            <textarea name="comment" placeholder="Enter comment"></textarea>
            <button type="submit">Submit Comment</button>
        </form>
    `);
});

app.post('/comment', 
    // Input validation middleware
    body('comment').trim().isLength({ min: 1, max: 500 }).escape(),
    async (req, res) => {
        try {
            const errors = validationResult(req);
            if (!errors.isEmpty()) {
                return res.status(400).json({ errors: errors.array() });
            }

            const { comment } = req.body;
            
            // Parameterized query prevents SQL injection
            const [result] = await dbPool.execute(
                'INSERT INTO comments (content, created_at) VALUES (?, NOW())',
                [comment]
            );
            
            res.redirect('/?success=1');
        } catch (error) {
            console.error('Database error:', error);
            res.status(500).send('Internal server error');
        }
    }
);

app.listen(3000, () => console.log('Server running on port 3000'));
```

## ⚖️ When To Use / When To Avoid

**Always implement these protections when:**
• Building any web application that accepts user input
• Working with databases or external APIs
• Handling authentication and