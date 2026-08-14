# 📌 Kubernetes basics for developers
*August 14, 2026 · Daily Dev Insight*

## 🧠 Overview

Kubernetes isn't just infrastructure anymore—it's become the runtime environment where your code lives and breathes. As a developer in 2026, understanding K8s basics means you can debug production issues faster, write deployment configs alongside your code, and actually understand why your app crashed at 3 AM (spoiler: it's usually resource limits or liveness probes).

Think of Kubernetes as an operating system for distributed applications. Just like you don't need to understand kernel internals to write desktop apps, you don't need to master every K8s component. But knowing the fundamentals—pods, deployments, services, and basic configuration—transforms you from someone who "throws code over the wall" to someone who owns their application's entire lifecycle.

The real power of K8s for developers isn't the orchestration magic—it's the declarative configuration model. You describe what you want (3 replicas of this container, accessible internally on port 8080), and K8s figures out how to make it happen. This shift from imperative scripts to declarative manifests is the mental model that takes time to internalize, but once it clicks, you'll wonder how you ever managed deployments any other way.

## 💡 Key Concepts

- **Pods are ephemeral**: Never write to local disk expecting it to persist, never hardcode pod IPs. Pods die and restart constantly—design for it from day one.

- **ConfigMaps and Secrets separate config from code**: Stop baking environment variables into Docker images. Use ConfigMaps for non-sensitive config and Secrets for credentials, mounted as volumes or env vars.

- **Services provide stable networking**: Pods come and go, but Services give you a stable DNS name and IP for accessing a set of pods. Learn the difference between ClusterIP, NodePort, and LoadBalancer.

- **Resource requests vs limits**: Requests guarantee minimum resources; limits set maximums. Set both, or watch your pod get OOMKilled during the next traffic spike.

- **Health checks keep apps reliable**: Liveness probes restart unhealthy containers, readiness probes remove pods from service rotation. Implement both correctly, or K8s can't help you self-heal.

## 🐍 Python Example

```python
# kubernetes_app.py - A Flask app with proper K8s health endpoints
from flask import Flask, jsonify
import os
import signal
import sys

app = Flask(__name__)

# Graceful shutdown handler for SIGTERM (K8s sends this before killing pods)
class GracefulShutdown:
    def __init__(self):
        self.ready = True
        signal.signal(signal.SIGTERM, self.handle_sigterm)
    
    def handle_sigterm(self, signum, frame):
        print("SIGTERM received, draining connections...")
        self.ready = False
        # Give yourself 25 seconds to finish requests (K8s default terminationGracePeriodSeconds is 30)
        # Do cleanup here: close DB connections, finish processing, etc.
        sys.exit(0)

shutdown_handler = GracefulShutdown()

@app.route('/healthz')
def liveness():
    """Liveness probe - is the app running at all?"""
    return jsonify({"status": "alive"}), 200

@app.route('/readyz')
def readiness():
    """Readiness probe - should we receive traffic?"""
    if not shutdown_handler.ready:
        return jsonify({"status": "shutting down"}), 503
    
    # Check dependencies: database, cache, etc.
    # Return 503 if not ready to serve traffic
    return jsonify({"status": "ready"}), 200

@app.route('/')
def hello():
    pod_name = os.getenv('HOSTNAME', 'unknown')
    namespace = os.getenv('POD_NAMESPACE', 'default')
    return jsonify({
        "message": "Hello from Kubernetes!",
        "pod": pod_name,
        "namespace": namespace
    })

if __name__ == '__main__':
    # Never run with debug=True in production
    app.run(host='0.0.0.0', port=8080)
```

## 🟨 JavaScript Example

```javascript
// server.js - Express server with K8s-aware patterns
const express = require('express');
const app = express();

// Track server state for graceful shutdown
let isShuttingDown = false;
const activeConnections = new Set();

// Middleware to track active requests
app.use((req, res, next) => {
  const connection = { req, res };
  activeConnections.add(connection);
  res.on('finish', () => activeConnections.delete(connection));
  next();
});

// Liveness probe - basic health check
app.get('/healthz', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// Readiness probe - can we handle traffic?
app.get('/readyz', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'not ready' });
  }
  // Add checks: DB connection, external APIs, etc.
  res.status(200).json({ status: 'ready' });
});

app.get('/api/data', (req, res) => {
  const podInfo = {
    hostname: process.env.HOSTNAME || 'unknown',
    namespace: process.env.POD_NAMESPACE || 'default',
    nodeIP: process.env.NODE_IP || 'unknown'
  };
  res.json({ data: 'Your data here', pod: podInfo });
});

const server = app.listen(8080, () => {
  console.log('Server listening on port 8080');
});

// Graceful shutdown on SIGTERM (K8s sends this before killing pod)
process.on('SIGTERM', () => {
  console.log('SIGTERM received, starting graceful shutdown');
  isShuttingDown = true;
  
  // Stop accepting new connections
  server.close(() => {
    console.log('Server closed to new connections');
  });
  
  // Give existing requests time to complete (30s default in K8s)
  setTimeout(() => {
    console.log(`Forcing shutdown. ${activeConnections.size} connections still active.`);
    process.exit(0);
  }, 25000);
});
```

## ⚖️ When To Use / When To Avoid

**Use Kubernetes when:**
- You need horizontal scaling across multiple containers/services
- You want declarative infrastructure and GitOps workflows
- High availability and self-healing are requirements
- You're building microservices with complex deployment patterns
- Your team has or can acquire K8s operational knowledge

**Avoid Kubernetes when:**
- You have a simple monolith that runs fine on a single VM
- Your team is 2 people and learning curve outweighs benefits
- Serverless platforms (Lambda, Cloud Run) meet your needs more simply
- You need bare-metal performance with minimal abstraction overhead
- Your organization lacks infrastructure automation maturity

## 📚 Further Reading

- [Kubernetes Documentation - Pods](https://kubernetes.io/docs/concepts/workloads/pods/) - Official deep dive into the fundamental K8s building block
- [The Twelve-Factor App](https://12factor.net/) - Design principles that make apps thrive in K8s environments
- [Kubernetes Best Practices](https://learnk8s.io/production-best-practices) - Comprehensive checklist for production-ready deployments
- [CNCF Landscape](https://landscape.cncf.io/) - Explore the broader cloud-native ecosystem around Kubernetes
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) - Essential commands every developer should bookmark

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*