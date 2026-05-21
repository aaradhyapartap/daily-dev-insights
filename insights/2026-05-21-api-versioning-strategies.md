# 📌 API versioning strategies
*May 21, 2026 · Daily Dev Insight*

## 🧠 Overview

API versioning is the art of evolving your API without breaking everyone who depends on it. It's like renovating a house while people are still living in it—you need a solid strategy to avoid chaos. The challenge isn't just technical; it's about managing the human contract between your API and its consumers over time.

The best versioning strategy depends on your API's maturity, your team's release cadence, and how much control you have over client updates. Whether you choose URL versioning (`/v1/users`), header versioning (`API-Version: 2023-05-21`), or content negotiation, the key is consistency and clear communication about deprecation timelines.

Remember: versioning is a promise to your users. Every version you ship is a commitment to maintain backward compatibility for a reasonable period. Choose a strategy that your team can actually sustain, not just what looks elegant on paper.

## 💡 Key Concepts

• **Semantic versioning for APIs**: Major versions for breaking changes, minor for new features, patch for bug fixes—but be stricter than library semver since API changes affect external systems
• **Deprecation windows**: Always announce breaking changes well in advance (6-12 months minimum) and provide migration paths with clear timelines
• **Version negotiation**: Let clients specify which version they understand, whether through URLs, headers, or content types—but pick one method and stick with it
• **Backward compatibility layers**: Maintain adapters that translate between versions when possible, allowing gradual migrations instead of forced upgrades
• **Sunset headers**: Use HTTP headers like `Sunset` and `Deprecation` to programmatically communicate version lifecycle information

## 🐍 Python Example

```python
from flask import Flask, request, jsonify
from datetime import datetime, timedelta
from functools import wraps

app = Flask(__name__)

# Version detection decorator
def versioned_api(supported_versions):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            # Check URL path version first
            version = request.view_args.get('version', 'v1')
            
            # Fall back to header-based versioning
            if not version.startswith('v'):
                version = request.headers.get('API-Version', 'v1')
            
            if version not in supported_versions:
                return jsonify({
                    'error': f'Unsupported API version: {version}',
                    'supported_versions': supported_versions
                }), 400
            
            # Add deprecation warning for old versions
            response = f(version=version, *args, **kwargs)
            if version == 'v1':
                # Set sunset date 6 months from now
                sunset_date = datetime.now() + timedelta(days=180)
                response.headers['Sunset'] = sunset_date.strftime('%a, %d %b %Y %H:%M:%S GMT')
                response.headers['Deprecation'] = 'true'
                response.headers['Link'] = '</api/v2/docs>; rel="successor-version"'
            
            return response
        return wrapper
    return decorator

@app.route('/api/<version>/users/<int:user_id>', methods=['GET'])
@versioned_api(['v1', 'v2'])
def get_user(user_id, version):
    # Simulate user data
    base_user = {
        'id': user_id,
        'name': 'John Doe',
        'email': 'john@example.com',
        'created_at': '2024-01-15T10:30:00Z'
    }
    
    if version == 'v1':
        # Legacy format - flatten structure
        return jsonify({
            'user_id': base_user['id'],
            'user_name': base_user['name'],
            'user_email': base_user['email']
        })
    
    elif version == 'v2':
        # New format - nested structure with additional metadata
        return jsonify({
            'data': base_user,
            'meta': {
                'version': 'v2',
                'deprecated_fields': None
            }
        })

if __name__ == '__main__':
    app.run(debug=True)
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const semver = require('semver');
const app = express();

// Middleware for API version handling
const versionMiddleware = (req, res, next) => {
    // Extract version from URL, header, or default
    let version = req.params.version || 
                  req.headers['accept-version'] || 
                  req.headers['api-version'] || 
                  '1.0.0';
    
    // Normalize version format
    if (!version.match(/^\d+\.\d+\.\d+$/)) {
        version = version.replace('v', '') + '.0.0';
    }
    
    req.apiVersion = semver.clean(version);
    
    // Add version info to response headers
    res.setHeader('API-Version', req.apiVersion);
    
    // Check if version is supported
    const supportedVersions = ['1.0.0', '1.1.0', '2.0.0'];
    const isSupported = supportedVersions.some(v => 
        semver.satisfies(req.apiVersion, `^${v.split('.')[0]}.0.0`)
    );
    
    if (!isSupported) {
        return res.status(400).json({
            error: 'Unsupported API version',
            requested: req.apiVersion,
            supported: supportedVersions
        });
    }
    
    next();
};

// Version-aware response transformer
const transformResponse = (data, version) => {
    const majorVersion = semver.major(version);
    
    switch (majorVersion) {
        case 1:
            // Legacy format
            return {
                status: 'success',
                result: data,
                timestamp: new Date().toISOString()
            };
        
        case 2:
            // Modern format with JSON:API compliance
            return {
                data: data,
                jsonapi: { version: '1.0' },
                meta: {
                    api_version: version,
                    generated_at: new Date().toISOString()
                }
            };
        
        default:
            return data;
    }
};

app.use(express.json());
app.use('/api/:version?', versionMiddleware);

app.get('/api/:version?/products/:id', async (req, res) => {
    try {
        // Simulate database fetch
        const product = {
            id: parseInt(req.params.id),
            name: 'Awesome Widget',
            price: 29.99,
            category: 'widgets'
        };
        
        // Add deprecation headers for v1
        if (semver.major(req.apiVersion) === 1) {
            const sixMonthsFromNow = new Date();
            sixMonthsFromNow.setMonth(sixMonthsFromNow.getMonth() + 6);
            
            res.setHeader('Deprecation', 'true');
            res.setHeader('Sunset', sixMonthsFromNow.toUTCString());
            res.setHeader('Link', '</api/v2/docs>; rel="successor-version"');
        }
        
        const response = transformResponse(product, req.apiVersion);
        res.json(response);
        
    } catch (error) {
        res.status(500).json({ error: 'Internal server error' });
    }
});

app.listen(3000, () => {
    console.log('Versioned API server running on port 3000');
});
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- Your API has external consumers you can't control or coordinate with
- You're making breaking changes to established endpoints
- You need to maintain SLA commitments while evolving functionality
- Your API is part of a platform or marketplace ecosystem