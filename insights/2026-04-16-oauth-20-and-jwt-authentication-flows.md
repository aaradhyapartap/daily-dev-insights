# 📌 OAuth 2.0 and JWT authentication flows
*April 16, 2026 · Daily Dev Insight*

## 🧠 Overview

OAuth 2.0 and JWT aren't competing technologies—they're complementary pieces of modern authentication architecture. OAuth 2.0 handles the authorization flow (who can access what), while JWTs are often the tokens that carry the proof of that authorization. Think of OAuth as the bouncer checking your ID at the club, and JWT as the wristband you get that proves you're allowed inside.

The real magic happens when you combine them thoughtfully. OAuth 2.0's authorization code flow gets you securely authenticated with a third-party provider, then JWTs carry your user claims and permissions across your microservices without requiring database lookups on every request. This pattern has become the backbone of modern distributed applications, from startup MVPs to enterprise systems handling millions of users.

The key insight most developers miss: JWTs are not inherently secure—their security comes from proper implementation of the OAuth flow and careful token management. Treat them like cash in your wallet, not like your driver's license.

## 💡 Key Concepts

• **Stateless vs Stateful**: JWTs enable stateless authentication—your server doesn't need to store session data. The token itself contains all necessary claims, but you sacrifice the ability to instantly revoke tokens.

• **Token Lifecycle Management**: Short-lived access tokens (15-60 minutes) paired with longer-lived refresh tokens strike the balance between security and user experience. Never store refresh tokens in localStorage.

• **Scope and Claims**: OAuth scopes define what permissions you're requesting, while JWT claims carry specific user data. Design your scopes granularly from day one—it's much harder to retrofit fine-grained permissions later.

• **PKCE is Non-Negotiable**: Proof Key for Code Exchange isn't just for mobile apps anymore. Use it for all OAuth flows, including SPAs. It prevents authorization code interception attacks.

• **Token Storage Strategy**: Access tokens can live in memory or httpOnly cookies. Refresh tokens should only ever be in httpOnly, secure, SameSite cookies. Local storage is a security anti-pattern for authentication tokens.

## 🐍 Python Example

```python
import jwt
import requests
from datetime import datetime, timedelta
from typing import Optional, Dict, Any

class OAuth2JWTManager:
    def __init__(self, client_id: str, client_secret: str, jwt_secret: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.jwt_secret = jwt_secret
    
    def exchange_code_for_tokens(self, auth_code: str, redirect_uri: str) -> Dict[str, Any]:
        """Exchange OAuth authorization code for tokens"""
        token_url = "https://oauth2.provider.com/token"
        
        payload = {
            "grant_type": "authorization_code",
            "client_id": self.client_id,
            "client_secret": self.client_secret,
            "code": auth_code,
            "redirect_uri": redirect_uri
        }
        
        response = requests.post(token_url, data=payload)
        response.raise_for_status()
        return response.json()
    
    def create_app_token(self, user_id: str, permissions: list) -> str:
        """Create internal JWT after OAuth success"""
        now = datetime.utcnow()
        payload = {
            "sub": user_id,  # Subject (user ID)
            "iat": now,      # Issued at
            "exp": now + timedelta(hours=1),  # Expires in 1 hour
            "scope": permissions,
            "iss": "my-app"  # Issuer
        }
        
        return jwt.encode(payload, self.jwt_secret, algorithm="HS256")
    
    def verify_token(self, token: str) -> Optional[Dict[str, Any]]:
        """Verify and decode JWT token"""
        try:
            payload = jwt.decode(
                token, 
                self.jwt_secret, 
                algorithms=["HS256"],
                options={"require": ["exp", "sub", "iat"]}
            )
            return payload
        except jwt.InvalidTokenError:
            return None

# Usage example
auth_manager = OAuth2JWTManager("your_client_id", "your_secret", "jwt_secret_key")

# Step 1: User comes back from OAuth provider with auth code
tokens = auth_manager.exchange_code_for_tokens("auth_code_from_callback", "http://localhost:3000/callback")

# Step 2: Create your own JWT for internal use
user_token = auth_manager.create_app_token("user123", ["read:profile", "write:posts"])

# Step 3: Verify token on subsequent requests
claims = auth_manager.verify_token(user_token)
print(f"User: {claims['sub']}, Permissions: {claims['scope']}")
```

## 🟨 JavaScript Example

```javascript
const jwt = require('jsonwebtoken');
const axios = require('axios');

class OAuth2JWTService {
  constructor(clientId, clientSecret, jwtSecret) {
    this.clientId = clientId;
    this.clientSecret = clientSecret;
    this.jwtSecret = jwtSecret;
  }

  async exchangeCodeForTokens(authCode, redirectUri) {
    const tokenUrl = 'https://oauth2.provider.com/token';
    
    try {
      const response = await axios.post(tokenUrl, {
        grant_type: 'authorization_code',
        client_id: this.clientId,
        client_secret: this.clientSecret,
        code: authCode,
        redirect_uri: redirectUri
      });
      
      return response.data;
    } catch (error) {
      throw new Error(`Token exchange failed: ${error.response?.data?.error}`);
    }
  }

  createAppToken(userId, permissions, expiresIn = '1h') {
    const payload = {
      sub: userId,
      iat: Math.floor(Date.now() / 1000),
      scope: permissions,
      iss: 'my-app'
    };

    return jwt.sign(payload, this.jwtSecret, { 
      expiresIn,
      algorithm: 'HS256'
    });
  }

  verifyToken(token) {
    try {
      return jwt.verify(token, this.jwtSecret, {
        algorithms: ['HS256'],
        issuer: 'my-app'
      });
    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        throw new Error('Token expired');
      }
      throw new Error('Invalid token');
    }
  }

  // Middleware for Express.js
  authenticateRequest() {
    return (req, res, next) => {
      const authHeader = req.headers.authorization;
      const token = authHeader?.split(' ')[1]; // Bearer token

      if (!token) {
        return res.status(401).json({ error: 'No token provided' });
      }

      try {
        const decoded = this.verifyToken(token);
        req.user = decoded;
        next();
      } catch (error) {
        return res.status(401).json({ error: error.message });
      }
    };
  }
}

// Usage in Express app
const authService = new OAuth2JWTService('client_id', 'client_secret', 'jwt_secret');

// Protected route example
app.get('/api/profile', authService.authenticateRequest(), (req, res) => {
  res.json({ 
    message: `Hello ${req.user.sub}`, 
    permissions: req.user.scope 
  });
});
```

## ⚖️ When To Use / When To Avoid

**✅ Use OAuth 2.0 + JWT when:**
• Building microservices that need to share authentication state
• Integrating with third-party services (Google, GitHub, Auth0)
• You need stateless authentication across distributed systems
• Building SPAs or mobile apps with API backends
• You want to avoid database hits on every auth