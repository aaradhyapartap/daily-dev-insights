# 📌 API versioning strategies
*October 18, 2026 · Daily Dev Insight*

## 🧠 Overview

API versioning is one of those topics that seems simple until you're knee-deep in production issues because you broke backward compatibility for 10,000 clients. The fundamental challenge is this: your API needs to evolve with new features and improvements, but existing clients expect the contract they built against to remain stable. Without a solid versioning strategy, you're choosing between stagnation and chaos.

The three most common approaches are URL versioning (`/v1/users`), header versioning (`Accept: application/vnd.api+json; version=1`), and query parameter versioning (`/users?version=1`). Each has trade-offs around discoverability, caching, and routing complexity. URL versioning wins on simplicity and explicitness—you can see the version at a glance in logs and documentation. Header versioning is more "RESTful" purist but harder to test with simple curl commands. Query parameters are the ugly middle child that most teams avoid.

The real insight? Pick one strategy early, communicate deprecation timelines clearly (typically 12-18 months minimum), and never break compatibility within a major version. Your future self—and your API consumers—will thank you.

## 💡 Key Concepts

- **Semantic versioning for APIs**: Use major.minor.patch where major changes break compatibility, minor adds features backward-compatibly, and patch fixes bugs. Communicate this clearly in your API documentation.

- **Version scope matters**: You can version the entire API (`/v2/*`), individual endpoints, or even specific fields. Whole-API versioning is clearest but forces clients to upgrade everything at once. Endpoint-level versioning offers flexibility but increases maintenance complexity.

- **Deprecation is a communication problem**: Implement sunset headers (`Sunset: Sat, 31 Dec 2027 23:59:59 GMT`), send deprecation warnings in responses, and maintain clear migration guides. Never surprise your users.

- **Support multiple versions temporarily, not forever**: Running 3+ API versions simultaneously is a maintenance nightmare. Plan for 2 concurrent versions max, with clear sunset dates.

- **Default version strategy**: Never auto-upgrade clients to the latest version. Require explicit version selection and return a clear error if no version is specified.

## �🐍 Python Example

```python
from flask import Flask, request, jsonify, make_response
from datetime import datetime, timedelta
from functools import wraps

app = Flask(__name__)

# Version registry with deprecation dates
API_VERSIONS = {
    'v1': {'deprecated': True, 'sunset': '2027-12-31'},
    'v2': {'deprecated': False, 'sunset': None},
}

def versioned_route(version):
    """Decorator to handle API versioning with deprecation warnings"""
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            version_info = API_VERSIONS.get(version, {})
            
            # Build response
            response = make_response(f(*args, **kwargs))
            
            # Add deprecation headers if needed
            if version_info.get('deprecated'):
                sunset_date = version_info.get('sunset')
                response.headers['Deprecation'] = 'true'
                response.headers['Sunset'] = sunset_date
                response.headers['Link'] = '</v2/users>; rel="successor-version"'
            
            response.headers['API-Version'] = version
            return response
        return wrapper
    return decorator

@app.route('/v1/users/<int:user_id>')
@versioned_route('v1')
def get_user_v1(user_id):
    """Legacy version - returns simple user object"""
    return jsonify({
        'id': user_id,
        'name': 'John Doe',
        'email': 'john@example.com'
    })

@app.route('/v2/users/<int:user_id>')
@versioned_route('v2')
def get_user_v2(user_id):
    """New version - returns enriched user object with metadata"""
    return jsonify({
        'id': user_id,
        'name': 'John Doe',
        'email': 'john@example.com',
        'profile': {
            'avatar_url': 'https://cdn.example.com/avatars/1.jpg',
            'created_at': '2024-01-15T10:30:00Z'
        },
        'preferences': {
            'timezone': 'UTC',
            'language': 'en'
        }
    })

if __name__ == '__main__':
    app.run(debug=True)
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const app = express();

// Middleware to extract and validate API version from URL
const apiVersionMiddleware = (req, res, next) => {
  const versionMatch = req.path.match(/^\/v(\d+)\//);
  
  if (!versionMatch) {
    return res.status(400).json({
      error: 'API version required',
      message: 'Please specify API version in URL (e.g., /v1/ or /v2/)'
    });
  }
  
  req.apiVersion = parseInt(versionMatch[1]);
  next();
};

app.use(apiVersionMiddleware);

// Version-specific user service
class UserService {
  static getUser(id, version) {
    const baseUser = {
      id,
      name: 'Jane Smith',
      email: 'jane@example.com'
    };
    
    // V1: Simple response
    if (version === 1) {
      return baseUser;
    }
    
    // V2: Enhanced response with breaking changes
    // Note: email moved to contact object
    if (version === 2) {
      return {
        id,
        name: 'Jane Smith',
        contact: {
          email: 'jane@example.com',
          phone: '+1-555-0123'
        },
        metadata: {
          createdAt: '2024-03-20T14:22:00Z',
          lastLogin: '2026-10-18T09:15:00Z'
        }
      };
    }
  }
}

app.get('/v:version(\\d+)/users/:id', (req, res) => {
  const { version } = req;
  const userId = parseInt(req.params.id);
  
  // Add deprecation warning for v1
  if (version === 1) {
    res.set({
      'Deprecation': 'true',
      'Sunset': 'Sun, 31 Dec 2027 23:59:59 GMT',
      'Link': '</v2/users>; rel="successor-version"'
    });
  }
  
  const user = UserService.getUser(userId, version);
  res.json(user);
});

app.listen(3000, () => {
  console.log('API server running on port 3000');
});
```

## ⚖️ When To Use / When To Avoid

**When to version your API:**
- ✅ You have external consumers who build against your API
- ✅ Making breaking changes (field removal, type changes, behavior changes)
- ✅ Your API is public or serves multiple independent teams
- ✅ You need to maintain backward compatibility during migrations

**When to avoid versioning (or use alternatives):**
- ❌ Internal microservices where you control all clients (use feature flags instead)
- ❌ Adding new optional fields (backward-compatible, no version bump needed)
- ❌ You can coordinate simultaneous deployment of API and all clients
- ❌ Pre-1.0 development phase (use date-based versioning or stay in beta)

## 📚 Further Reading

- [REST API Versioning Strategies - Microsoft Azure Architecture](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design#versioning-a-restful-web-api) - Comprehensive guide from Microsoft on versioning patterns
- [Semantic Versioning 2.0.0 Specification](https://sem