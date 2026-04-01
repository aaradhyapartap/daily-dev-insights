# 📌 API versioning strategies
*April 01, 2026 · Daily Dev Insight*

## 🧠 Overview

API versioning is like planning for divorce before you get married—nobody wants to think about it, but everyone who skips it regrets it later. As your API evolves, you'll inevitably need to make breaking changes that could crash every integration your users have built. The difference between a well-versioned API and a chaotic one is the difference between smooth migrations and angry Slack messages at 2 AM.

The real challenge isn't just picking a versioning scheme—it's designing for evolution from day one. Too many teams treat versioning as an afterthought, leading to messy URL patterns, inconsistent behavior across versions, and technical debt that compounds with every release. Smart API versioning means thinking about backward compatibility, deprecation timelines, and migration paths before you ship v1.

## 💡 Key Concepts

• **Semantic versioning applies to APIs**: Use MAJOR.MINOR.PATCH where major versions introduce breaking changes, minor versions add features, and patches fix bugs without breaking compatibility

• **Choose your versioning location wisely**: URL path versioning (`/v1/users`) is explicit and cacheable, header versioning keeps URLs clean, and query parameters offer flexibility but can be overlooked

• **Version the contract, not the implementation**: Multiple API versions can share the same underlying business logic—use transformation layers to maintain different response formats

• **Plan your deprecation strategy upfront**: Define clear timelines for version support, provide migration tools, and communicate changes early to avoid emergency updates

• **Consider versioning granularity**: You can version entire APIs, individual resources, or even specific fields—find the right balance between flexibility and complexity

## �🐍 Python Example

```python
from flask import Flask, request, jsonify
from functools import wraps
from enum import Enum

class APIVersion(Enum):
    V1 = "1.0"
    V2 = "2.0"
    V3 = "2.1"

app = Flask(__name__)

def version_handler(func):
    """Decorator to handle API versioning with automatic transformation"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        # Check version from header, URL path, or default to latest
        version = (
            request.headers.get('API-Version') or 
            request.view_args.get('version') or 
            APIVersion.V3.value
        )
        
        # Execute the main function to get raw data
        response_data = func(*args, **kwargs)
        
        # Transform response based on version
        if version == APIVersion.V1.value:
            return transform_to_v1(response_data)
        elif version == APIVersion.V2.value:
            return transform_to_v2(response_data)
        else:
            return response_data  # Latest version
            
    return wrapper

def transform_to_v1(data):
    """Transform modern response to v1 format"""
    if isinstance(data.get('user'), dict):
        # V1 used snake_case and flat structure
        user = data['user']
        return {
            'user_id': user.get('id'),
            'full_name': f"{user.get('firstName', '')} {user.get('lastName', '')}".strip(),
            'email_address': user.get('email'),
            'created': user.get('createdAt')
        }
    return data

def transform_to_v2(data):
    """Transform modern response to v2 format"""
    # V2 introduced camelCase but kept some legacy fields
    return data

@app.route('/<version>/users/<int:user_id>')
@app.route('/users/<int:user_id>')  # Versionless defaults to latest
@version_handler
def get_user(user_id, version=None):
    """Get user data - automatically versioned response"""
    # Your business logic here (stays the same across versions)
    return {
        'user': {
            'id': user_id,
            'firstName': 'John',
            'lastName': 'Doe', 
            'email': 'john@example.com',
            'createdAt': '2026-04-01T10:00:00Z',
            'profile': {'avatar': 'https://example.com/avatar.jpg'}
        }
    }
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const semver = require('semver');
const app = express();

class APIVersionManager {
    constructor() {
        this.supportedVersions = ['1.0.0', '2.0.0', '2.1.0'];
        this.deprecatedVersions = new Map([
            ['1.0.0', { sunset: '2026-12-31', message: 'Please migrate to v2.0+' }]
        ]);
    }

    parseVersion(versionString) {
        // Handle different version formats: "v1", "1.0", "1.0.0"
        const cleaned = versionString.replace(/^v/, '');
        const parts = cleaned.split('.');
        
        while (parts.length < 3) parts.push('0');
        return parts.join('.');
    }

    getCompatibleVersion(requestedVersion) {
        const parsed = this.parseVersion(requestedVersion);
        
        // Find the highest compatible version
        const compatible = this.supportedVersions
            .filter(v => semver.satisfies(v, `~${parsed}`))
            .sort(semver.rcompare)[0];
            
        return compatible || this.supportedVersions[this.supportedVersions.length - 1];
    }

    middleware() {
        return (req, res, next) => {
            // Extract version from various sources
            const requestedVersion = 
                req.headers['api-version'] ||
                req.params.version ||
                req.query.version ||
                '2.1.0'; // Default to latest

            const actualVersion = this.getCompatibleVersion(requestedVersion);
            
            // Add deprecation warnings
            const deprecation = this.deprecatedVersions.get(actualVersion);
            if (deprecation) {
                res.set('Warning', `299 - "API version ${actualVersion} is deprecated. ${deprecation.message}"`);
                res.set('Sunset', deprecation.sunset);
            }

            req.apiVersion = actualVersion;
            res.set('API-Version', actualVersion);
            next();
        };
    }
}

const versionManager = new APIVersionManager();
app.use(versionManager.middleware());

// Version-aware response transformer
function transformResponse(data, version) {
    if (semver.lt(version, '2.0.0')) {
        // V1 transformation
        return {
            user_id: data.id,
            name: data.fullName,
            email: data.emailAddress,
            created_date: data.createdAt
        };
    } else if (semver.lt(version, '2.1.0')) {
        // V2 transformation - removed nested profile
        const { profile, ...rest } = data;
        return { ...rest, avatarUrl: profile?.avatar };
    }
    
    return data; // Latest version
}

app.get('/api/:version?/users/:id', (req, res) => {
    // Simulate database fetch
    const userData = {
        id: parseInt(req.params.id),
        fullName: 'Jane Smith',
        emailAddress: 'jane@example.com',
        createdAt: '2026-04-01T10:00:00Z',
        profile: { avatar: 'https://example.com/jane.jpg' }
    };

    const transformed = transformResponse(userData, req.apiVersion);
    res.json(transformed);
});
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
• Your API has external consumers you can't control or coordinate with
• You need to maintain backward compatibility while adding features
• Breaking changes are inevitable but must be gradual
• You're building a public API or platform that others depend on

**❌ When To Avoid:**
• Internal APIs where