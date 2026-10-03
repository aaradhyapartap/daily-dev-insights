# 📌 Kubernetes basics for developers
*October 03, 2026 · Daily Dev Insight*

## 🧠 Overview

Kubernetes isn't just infrastructure—it's the contract between your code and production. As a developer, you don't need to be a cluster admin, but understanding pods, services, and deployments will fundamentally change how you think about building resilient applications. The days of "works on my machine" are over; Kubernetes forces you to think about twelve-factor principles, health checks, and graceful shutdowns from day one.

The mental shift is this: you're no longer deploying *to a server*, you're declaring *desired state*. Your application becomes a set of YAML manifests describing what should exist, and Kubernetes continuously reconciles reality with your intent. This declarative model means your deployment configuration is code, versioned alongside your application, and reproducible across environments.

What trips up most developers is overcomplicating their first steps. Start with the primitives—pods (containers running together), services (stable networking), and deployments (rolling updates and replicas). Master these three, and you'll handle 90% of real-world scenarios. The ecosystem has hundreds of custom resources and operators, but they're all built on these foundations.

## 💡 Key Concepts

- **Pods are ephemeral**: Never rely on a pod's IP or filesystem persistence. Pods die and resurrect constantly. Design for statelessness or use proper StatefulSets and PersistentVolumes.

- **Services provide stable endpoints**: While pod IPs change, a Service gives you a consistent DNS name (`my-app.default.svc.cluster.local`) that load-balances across healthy pods.

- **Health checks aren't optional**: Liveness probes tell Kubernetes when to restart your container; readiness probes control traffic flow. Implement `/healthz` and `/ready` endpoints in every service.

- **Resource limits prevent noisy neighbors**: Always set CPU/memory requests (guaranteed) and limits (maximum). Without them, one misbehaving pod can starve your entire node.

- **ConfigMaps and Secrets externalize configuration**: Hardcoded values make your containers environment-specific. Use ConfigMaps for non-sensitive data and Secrets for credentials, mounted as files or environment variables.

## 🐍 Python Example

```python
from flask import Flask, jsonify
import os
import signal
import sys
import time

app = Flask(__name__)

# Track application health
is_ready = False
shutdown_initiated = False

@app.route('/healthz')
def health_check():
    """Liveness probe - is the app running?"""
    if shutdown_initiated:
        return jsonify({"status": "shutting down"}), 503
    return jsonify({"status": "healthy"}), 200

@app.route('/ready')
def readiness_check():
    """Readiness probe - can the app serve traffic?"""
    if not is_ready or shutdown_initiated:
        return jsonify({"status": "not ready"}), 503
    return jsonify({"status": "ready"}), 200

@app.route('/api/data')
def get_data():
    """Example endpoint that uses configuration"""
    environment = os.getenv('ENVIRONMENT', 'unknown')
    database_url = os.getenv('DATABASE_URL', 'not-configured')
    return jsonify({
        "environment": environment,
        "database": database_url.split('@')[-1],  # Hide credentials
        "version": os.getenv('APP_VERSION', '1.0.0')
    })

def graceful_shutdown(signum, frame):
    """Handle SIGTERM from Kubernetes for zero-downtime deployments"""
    global shutdown_initiated
    print("Received shutdown signal, draining connections...", flush=True)
    shutdown_initiated = True
    time.sleep(5)  # Allow in-flight requests to complete
    sys.exit(0)

if __name__ == '__main__':
    # Register signal handler for graceful shutdown
    signal.signal(signal.SIGTERM, graceful_shutdown)
    
    # Simulate startup tasks (DB connection, cache warming, etc.)
    time.sleep(2)
    is_ready = True
    
    print("Application ready to serve traffic", flush=True)
    app.run(host='0.0.0.0', port=8080)
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const app = express();

let isReady = false;
let isShuttingDown = false;

// Liveness probe endpoint
app.get('/healthz', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting down' });
  }
  res.json({ status: 'healthy' });
});

// Readiness probe endpoint
app.get('/ready', (req, res) => {
  if (!isReady || isShuttingDown) {
    return res.status(503).json({ status: 'not ready' });
  }
  res.json({ status: 'ready' });
});

// Example business logic using injected config
app.get('/api/config', (req, res) => {
  res.json({
    environment: process.env.NODE_ENV || 'development',
    apiTimeout: process.env.API_TIMEOUT_MS || '5000',
    featureFlags: process.env.FEATURE_FLAGS?.split(',') || [],
    podName: process.env.HOSTNAME // Kubernetes sets this automatically
  });
});

const PORT = process.env.PORT || 3000;
const server = app.listen(PORT, async () => {
  console.log(`Server starting on port ${PORT}...`);
  
  // Simulate async initialization (DB connections, etc.)
  await new Promise(resolve => setTimeout(resolve, 3000));
  
  isReady = true;
  console.log('Application ready to accept traffic');
});

// Graceful shutdown handler for SIGTERM
process.on('SIGTERM', () => {
  console.log('SIGTERM received, starting graceful shutdown');
  isShuttingDown = true;
  
  server.close(() => {
    console.log('HTTP server closed, exiting process');
    process.exit(0);
  });
  
  // Force shutdown after 30 seconds
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
});
```

## ⚖️ When To Use / When To Avoid

**Use Kubernetes when:**
- You need horizontal scaling and self-healing capabilities
- Running microservices that require service discovery
- Managing multiple environments (dev/staging/prod) with identical infrastructure
- Your team has outgrown Heroku/Platform-as-a-Service limitations
- You need advanced deployment strategies (blue/green, canary)

**Avoid Kubernetes when:**
- You're a solo developer or small team without ops support
- Running a simple monolith that doesn't need horizontal scaling
- Your infrastructure budget is limited (managed Kubernetes isn't cheap)
- You don't have CI/CD pipelines in place yet
- Serverless platforms (Lambda, Cloud Run) already meet your needs

## 📚 Further Reading

- [Kubernetes Official Documentation - Concepts](https://kubernetes.io/docs/concepts/) — Start with "Workloads" and "Services"
- [The Twelve-Factor App Methodology](https://12factor.net/) — Essential reading for cloud-native development
- [Kubernetes Best Practices by Google](https://cloud.google.com/blog/products/containers-kubernetes/your-guide-kubernetes-best-practices) — Production-ready patterns
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) — Bookmark this for daily debugging
- [learnk8s.io Production Best Practices](https://learnk8s.io/production-best-practices) — Comprehensive checklist for production readiness

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*