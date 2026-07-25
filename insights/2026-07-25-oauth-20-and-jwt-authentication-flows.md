# 📌 OAuth 2.0 and JWT authentication flows
*July 25, 2026 · Daily Dev Insight*

## 🧠 Overview

OAuth 2.0 and JWT aren't competing technologies—they're complementary pieces of modern authentication architecture that solve different problems. OAuth 2.0 is an **authorization framework** that lets users grant third-party applications limited access to their resources without sharing credentials. Think "Sign in with Google" or giving a mobile app access to your GitHub repos. JWT (JSON Web Tokens), on the other hand, is a **token format**—a compact, self-contained way to securely transmit information between parties.

Here's where they intersect beautifully: OAuth 2.0 defines *how* tokens are issued and exchanged, while JWT defines *what* those tokens look like and how they're structured. When you implement OAuth 2.0, you'll often use JWTs as your access tokens because they're stateless—the server doesn't need to hit a database on every request to validate them. The JWT carries all the necessary claims (user ID, permissions, expiration) right in the token itself, cryptographically signed to prevent tampering.

The confusion many developers face is thinking OAuth 2.0 is just "login with social media." In reality, it's a flexible framework with multiple grant types (authorization code, client credentials, refresh tokens) designed for different scenarios. Whether you're building a microservices architecture, a mobile app, or integrating with third-party APIs, understanding when to use which flow—and how to properly validate and handle JWTs—is crucial for building secure, scalable authentication systems.

## 💡 Key Concepts

- **OAuth 2.0 is about delegation, not authentication**: It answers "what can this app do on behalf of the user?" not "who is this user?" (though OpenID Connect extends OAuth for authentication)
- **JWTs are stateless but not riskless**: No database lookup needed for validation, but they can't be revoked without additional infrastructure (blacklists, short expiration times, refresh token rotation)
- **Never store sensitive data in JWT payload**: The payload is base64-encoded, not encrypted—anyone can decode and read it. Use it for non-sensitive claims only
- **The Authorization Code flow with PKCE is the gold standard**: For any client that can't securely store secrets (SPAs, mobile apps), PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks
- **Access tokens should be short-lived, refresh tokens long-lived**: Think 15-minute access tokens with hourly refresh tokens—limits damage from token theft while maintaining good UX

## 🐍 Python Example

```python
from datetime import datetime, timedelta
from jose import jwt, JWTError
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()

# Configuration
SECRET_KEY = "your-secret-key-min-256-bits"  # Use env vars in production!
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def create_access_token(data: dict, expires_delta: timedelta = None):
    """Generate a JWT access token with claims and expiration"""
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    
    to_encode.update({
        "exp": expire,
        "iat": datetime.utcnow(),  # Issued at
        "type": "access"
    })
    
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)):
    """Dependency to validate JWT and extract user from token"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        # Decode and verify signature
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        
        if user_id is None:
            raise credentials_exception
            
        # Additional validation
        if payload.get("type") != "access":
            raise credentials_exception
            
        return {"user_id": user_id, "scopes": payload.get("scopes", [])}
    
    except JWTError:
        raise credentials_exception

@app.post("/token")
async def login(username: str, password: str):
    """Authenticate user and issue JWT token"""
    # Verify credentials (use proper password hashing in production!)
    if username != "demo" or password != "password":
        raise HTTPException(status_code=400, detail="Incorrect credentials")
    
    access_token = create_access_token(
        data={"sub": username, "scopes": ["read", "write"]},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/protected")
async def protected_route(current_user: dict = Depends(get_current_user)):
    """Example protected endpoint requiring valid JWT"""
    return {"message": f"Hello {current_user['user_id']}", "scopes": current_user['scopes']}
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const app = express();

app.use(express.json());

const SECRET_KEY = 'your-secret-key-min-256-bits'; // Use env vars!
const ACCESS_TOKEN_EXPIRY = '15m';
const REFRESH_TOKEN_EXPIRY = '7d';

// In-memory store (use Redis in production)
const refreshTokens = new Set();

// Generate access and refresh token pair
app.post('/oauth/token', (req, res) => {
  const { username, password, grant_type, refresh_token } = req.body;
  
  // Handle refresh token grant
  if (grant_type === 'refresh_token') {
    if (!refreshTokens.has(refresh_token)) {
      return res.status(401).json({ error: 'invalid_grant' });
    }
    
    try {
      const decoded = jwt.verify(refresh_token, SECRET_KEY);
      const newAccessToken = generateAccessToken({ sub: decoded.sub });
      
      return res.json({ 
        access_token: newAccessToken,
        token_type: 'Bearer',
        expires_in: 900 
      });
    } catch (err) {
      return res.status(401).json({ error: 'invalid_token' });
    }
  }
  
  // Handle password grant (simplified - use proper auth!)
  if (username === 'demo' && password === 'password') {
    const accessToken = generateAccessToken({ sub: username, scope: 'read write' });
    const refreshToken = generateRefreshToken({ sub: username });
    
    refreshTokens.add(refreshToken); // Store for validation
    
    return res.json({
      access_token: accessToken,
      refresh_token: refreshToken,
      token_type: 'Bearer',
      expires_in: 900
    });
  }
  
  res.status(400).json({ error: 'invalid_grant' });
});

// Middleware to validate JWT
function authenticateToken(req, res, next) {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN
  
  if (!token) return res.sendStatus(401);
  
  jwt.verify(token, SECRET_KEY, (err, user) => {
    if (err) return res.status(403).json({ error: 'invalid_token' });
    req.user = user;
    next();
  });
}

// Protected endpoint
app.get('/api/profile', authenticateToken, (req, res) => {
  res.json({ 
    user: req.user.sub,