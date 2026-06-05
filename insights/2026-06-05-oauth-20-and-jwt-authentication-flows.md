# 📌 OAuth 2.0 and JWT authentication flows
*June 05, 2026 · Daily Dev Insight*

## 🧠 Overview

OAuth 2.0 and JWT tokens work beautifully together, but they solve different problems. OAuth 2.0 is an authorization framework that lets users grant limited access to their resources without sharing passwords. JWT (JSON Web Tokens) is a compact, self-contained way to securely transmit information between parties. When combined, OAuth issues JWTs as access tokens, creating a stateless, scalable authentication system.

The magic happens in the flow: a user authorizes your app through an OAuth provider (like GitHub or Google), receives a JWT access token, and your application can then make authenticated requests using that token. The JWT contains encoded user information and permissions, eliminating the need for database lookups on every request. This pattern has become the backbone of modern API authentication because it scales horizontally and works seamlessly across microservices.

## 💡 Key Concepts

• **Authorization vs Authentication**: OAuth handles authorization (what can you access), while JWT often carries authentication data (who you are) within the token payload
• **Stateless tokens**: JWTs are self-contained, meaning your servers don't need to store session state or make database calls to validate tokens
• **Token expiration**: Short-lived access tokens (15-60 minutes) paired with longer-lived refresh tokens provide security without constant re-authentication
• **Scope-based permissions**: OAuth scopes define granular permissions that get encoded into JWT claims, enabling fine-grained access control
• **Signature verification**: JWTs are cryptographically signed, allowing any service to verify authenticity without calling back to the issuer

## 🐍 Python Example

```python
import jwt
import requests
from datetime import datetime, timedelta
from flask import Flask, request, jsonify

app = Flask(__name__)
SECRET_KEY = "your-secret-key"
OAUTH_CLIENT_ID = "your-github-client-id"
OAUTH_CLIENT_SECRET = "your-github-client-secret"

def create_jwt_token(user_data, scopes):
    """Create a JWT token with user data and OAuth scopes"""
    payload = {
        'user_id': user_data['id'],
        'username': user_data['login'],
        'email': user_data.get('email'),
        'scopes': scopes,
        'exp': datetime.utcnow() + timedelta(hours=1),  # 1 hour expiry
        'iat': datetime.utcnow(),
        'iss': 'your-app-name'
    }
    return jwt.encode(payload, SECRET_KEY, algorithm='HS256')

@app.route('/oauth/callback')
def oauth_callback():
    """Handle OAuth callback and exchange code for token"""
    code = request.args.get('code')
    
    # Exchange authorization code for access token
    token_response = requests.post('https://github.com/login/oauth/access_token', {
        'client_id': OAUTH_CLIENT_ID,
        'client_secret': OAUTH_CLIENT_SECRET,
        'code': code
    }, headers={'Accept': 'application/json'})
    
    access_token = token_response.json()['access_token']
    scopes = token_response.json().get('scope', '').split(',')
    
    # Get user data from GitHub API
    user_response = requests.get('https://api.github.com/user',
        headers={'Authorization': f'Bearer {access_token}'})
    user_data = user_response.json()
    
    # Create our own JWT token with user data and scopes
    jwt_token = create_jwt_token(user_data, scopes)
    
    return jsonify({'token': jwt_token, 'user': user_data['login']})

def verify_jwt_token(token):
    """Verify and decode JWT token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
        return payload
    except jwt.ExpiredSignatureError:
        return None

@app.route('/protected')
def protected_route():
    """Protected route that requires valid JWT"""
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return jsonify({'error': 'Missing or invalid authorization header'}), 401
    
    token = auth_header.split(' ')[1]
    payload = verify_jwt_token(token)
    
    if not payload:
        return jsonify({'error': 'Invalid or expired token'}), 401
    
    return jsonify({
        'message': f'Hello {payload["username"]}!',
        'scopes': payload['scopes']
    })
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const axios = require('axios');

const app = express();
const SECRET_KEY = process.env.JWT_SECRET || 'your-secret-key';
const GITHUB_CLIENT_ID = process.env.GITHUB_CLIENT_ID;
const GITHUB_CLIENT_SECRET = process.env.GITHUB_CLIENT_SECRET;

// Middleware to verify JWT tokens
const authenticateToken = (req, res, next) => {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN
    
    if (!token) {
        return res.status(401).json({ error: 'Access token required' });
    }
    
    jwt.verify(token, SECRET_KEY, (err, user) => {
        if (err) {
            return res.status(403).json({ error: 'Invalid or expired token' });
        }
        req.user = user;
        next();
    });
};

// OAuth callback handler
app.get('/auth/github/callback', async (req, res) => {
    const { code } = req.query;
    
    try {
        // Exchange code for access token
        const tokenResponse = await axios.post('https://github.com/login/oauth/access_token', {
            client_id: GITHUB_CLIENT_ID,
            client_secret: GITHUB_CLIENT_SECRET,
            code: code
        }, {
            headers: { 'Accept': 'application/json' }
        });
        
        const { access_token, scope } = tokenResponse.data;
        
        // Fetch user data
        const userResponse = await axios.get('https://api.github.com/user', {
            headers: { 'Authorization': `Bearer ${access_token}` }
        });
        
        // Create JWT with user data and OAuth scopes
        const jwtPayload = {
            userId: userResponse.data.id,
            username: userResponse.data.login,
            email: userResponse.data.email,
            scopes: scope ? scope.split(',') : [],
            githubToken: access_token // Store for GitHub API calls
        };
        
        const jwtToken = jwt.sign(jwtPayload, SECRET_KEY, { 
            expiresIn: '1h',
            issuer: 'your-app-name'
        });
        
        res.json({ 
            token: jwtToken, 
            user: userResponse.data.login,
            expiresIn: 3600 
        });
        
    } catch (error) {
        res.status(500).json({ error: 'OAuth flow failed' });
    }
});

// Protected route using JWT middleware
app.get('/api/profile', authenticateToken, async (req, res) => {
    try {
        // Use stored GitHub token for API calls
        const githubResponse = await axios.get('https://api.github.com/user', {
            headers: { 'Authorization': `Bearer ${req.user.githubToken}` }
        });
        
        res.json({
            profile: githubResponse.data,
            tokenScopes: req.user.scopes,
            tokenExpiry: new Date(req.user.exp * 1000)
        });
    } catch (error) {
        res.status(500).json({ error: 'Failed to fetch profile' });
    }
});

app.