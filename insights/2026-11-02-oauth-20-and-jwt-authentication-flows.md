# 📌 OAuth 2.0 and JWT authentication flows
*November 02, 2026 · Daily Dev Insight*

## 🧠 Overview

OAuth 2.0 and JWT often get lumped together in auth discussions, but they're solving different problems. OAuth 2.0 is an **authorization framework** that lets users grant third-party applications limited access to their resources without sharing passwords. Think "Login with Google" or allowing a CI/CD tool to access your GitHub repos. JWT (JSON Web Tokens), on the other hand, is a **token format** — a compact, self-contained way to transmit information between parties as a JSON object.

Here's where it gets interesting: these two frequently work *together*. In modern implementations, OAuth 2.0 access tokens are often JWTs. This gives you the best of both worlds — OAuth's battle-tested authorization flows combined with JWT's stateless verification. The access token issued by your OAuth server can be a JWT that contains claims about the user and their permissions, eliminating the need for constant database lookups.

The critical insight senior engineers understand is this: **OAuth 2.0 handles the "who can access what" dance between services, while JWT provides the actual credential format**. You can use OAuth without JWT (opaque tokens with server-side validation), and you can use JWT without OAuth (direct JWT issuance for same-domain auth), but combining them creates a powerful, scalable authentication system.

## 💡 Key Concepts

- **OAuth 2.0 Grant Types**: Authorization Code (for web apps), Client Credentials (service-to-service), and PKCE (mobile/SPA apps). Choose based on your client type and security needs.

- **JWT Structure**: Header (algorithm metadata), Payload (claims about user/permissions), Signature (cryptographic verification). JWTs are signed, not encrypted — don't put secrets in the payload.

- **Stateless vs Stateful**: JWT enables stateless auth where servers don't maintain session storage. Trade-off: harder to revoke tokens before expiration. Solve with short expiration times and refresh token rotation.

- **The Token Lifecycle**: Access tokens (short-lived, 15-60 min) grant immediate access. Refresh tokens (long-lived, days/weeks) obtain new access tokens without re-authentication. Keep refresh tokens secure and rotating.

- **Scope and Claims**: OAuth scopes define what actions are permitted (`read:user`, `write:repo`). JWT claims describe who the subject is (`sub`, `email`, `roles`). Map OAuth scopes to JWT claims in your authorization server.

## 🐍 Python Example

```python
from datetime import datetime, timedelta
from jose import jwt, JWTError
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# In production, use environment variables and rotate regularly
SECRET_KEY = "your-256-bit-secret-key-keep-it-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(data: dict, expires_delta: timedelta = None):
    """Generate a JWT access token with claims"""
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({
        "exp": expire,
        "iat": datetime.utcnow(),
        "type": "access"
    })
    # Sign the token with your secret key
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)):
    """Verify and decode JWT from Authorization header"""
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        # Decode and verify signature in one step
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        scopes: list = payload.get("scopes", [])
        
        if user_id is None:
            raise credentials_exception
            
        return {"user_id": user_id, "scopes": scopes}
    except JWTError:
        raise credentials_exception

@app.get("/protected-resource")
async def read_protected_data(current_user: dict = Depends(get_current_user)):
    """Example protected endpoint requiring valid JWT"""
    if "read:data" not in current_user["scopes"]:
        raise HTTPException(status_code=403, detail="Insufficient permissions")
    return {"message": "Secret data", "user": current_user["user_id"]}
```

## 🟨 JavaScript Example

```javascript
const jwt = require('jsonwebtoken');
const express = require('express');
const app = express();

const SECRET_KEY = 'your-256-bit-secret-key-keep-it-secret';
const ACCESS_TOKEN_EXPIRY = '30m';

// Middleware to verify JWT from Authorization header
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN
  
  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  jwt.verify(token, SECRET_KEY, (err, payload) => {
    if (err) {
      // Token expired, invalid signature, or malformed
      return res.status(403).json({ error: 'Invalid or expired token' });
    }
    
    // Attach decoded payload to request object
    req.user = {
      userId: payload.sub,
      scopes: payload.scopes || [],
      email: payload.email
    };
    next();
  });
};

// OAuth token endpoint (simplified - use a proper OAuth library in production)
app.post('/oauth/token', express.json(), (req, res) => {
  const { grant_type, client_id, client_secret } = req.body;
  
  // Validate client credentials (check against database in production)
  if (grant_type === 'client_credentials' && client_id && client_secret) {
    const accessToken = jwt.sign(
      {
        sub: client_id,
        scopes: ['read:data', 'write:data'],
        email: 'user@example.com',
        type: 'access'
      },
      SECRET_KEY,
      { expiresIn: ACCESS_TOKEN_EXPIRY, algorithm: 'HS256' }
    );
    
    res.json({
      access_token: accessToken,
      token_type: 'Bearer',
      expires_in: 1800 // 30 minutes in seconds
    });
  } else {
    res.status(400).json({ error: 'invalid_request' });
  }
});

// Protected route requiring authentication
app.get('/api/user/profile', authenticateToken, (req, res) => {
  res.json({
    userId: req.user.userId,
    email: req.user.email,
    scopes: req.user.scopes
  });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## ⚖️ When To Use / When To Avoid

**✅ Use OAuth 2.0 + JWT When:**
- Building APIs consumed by third-party applications
- Implementing SSO across multiple services/domains
- Need stateless, horizontally scalable authentication
- Mobile apps or SPAs requiring secure token-based auth
- Microservices architecture with service-to-service auth

**❌ Avoid or Consider Alternatives When:**
- Simple monolithic app with server-side sessions (cookies are simpler)
- Need immediate token revocation (opaque tokens with central validation are better)
- Storing sensitive data in tokens (JWTs are visible to anyone with the token)
- Very high-security requirements (consider mTLS or certificate-based auth)
- Learning project — start simpler, add OAuth when you need third-party access

##