# 📌 Kubernetes basics for developers
*November 22, 2026 · Daily Dev Insight*

## 🧠 Overview

Kubernetes isn't just an operations tool—it's fundamentally changed how we build and deploy applications. If you're still thinking of it as "someone else's problem," you're missing out on writing more robust, scalable software. At its core, Kubernetes is a container orchestration platform that manages your application lifecycle, but the real insight is this: it forces you to think about your application as a collection of stateless, replaceable components. This mindset shift makes you a better developer, even if you never touch kubectl in production.

The learning curve is real, but you don't need to master everything at once. Start by understanding pods (your running containers), deployments (how you manage versions), and services (how components talk to each other). These three concepts will get you 80% of the way to being productive. The beauty of Kubernetes is that it provides a consistent API across any cloud provider or on-premises setup—write your manifests once, and they'll work anywhere Kubernetes runs.

Modern development increasingly means local Kubernetes environments (Minikube, Kind, or Docker Desktop's K8s) where you can iterate quickly. Gone are the days when Kubernetes was only for large-scale deployments. If your app will eventually run in containers, developing against Kubernetes locally means fewer surprises when you ship to staging or production.

## 💡 Key Concepts

- **Pods are ephemeral**: Never assume a pod will stay running or maintain state. Design for failure, use external storage for persistence, and treat pods like cattle, not pets.

- **Declarative configuration over imperative commands**: You describe the desired state (YAML manifests), and Kubernetes works to maintain it. This "reconciliation loop" is the heart of everything.

- **Services provide stable networking**: Pod IPs change constantly. Services give you a stable DNS name and load balancing across pod replicas automatically.

- **Labels and selectors are your organizing principle**: Everything in Kubernetes uses key-value labels. Services find pods via label selectors, making loose coupling trivial.

- **Health checks are mandatory**: Liveness and readiness probes tell Kubernetes when your app is healthy. Implement these endpoints early—they're critical for zero-downtime deployments.

## 🐍 Python Example

```python
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

# Health check endpoint - required for Kubernetes liveness probe
@app.route('/healthz')
def health_check():
    return jsonify({"status": "healthy"}), 200

# Readiness check - only mark ready after dependencies are available
@app.route('/ready')
def readiness_check():
    # In production, check database connections, cache availability, etc.
    return jsonify({"status": "ready"}), 200

# Main application endpoint showing pod identity
@app.route('/api/info')
def info():
    return jsonify({
        "pod_name": os.getenv('HOSTNAME', 'unknown'),
        "node_name": os.getenv('NODE_NAME', 'unknown'),
        "namespace": os.getenv('POD_NAMESPACE', 'default'),
        "version": os.getenv('APP_VERSION', '1.0.0')
    })

if __name__ == '__main__':
    # Bind to 0.0.0.0 so Kubernetes can reach the container
    # Port should match your Kubernetes service configuration
    port = int(os.getenv('PORT', 8080))
    app.run(host='0.0.0.0', port=port)

# Dockerfile companion:
# FROM python:3.11-slim
# WORKDIR /app
# COPY requirements.txt .
# RUN pip install -r requirements.txt
# COPY app.py .
# USER 1000
# CMD ["python", "app.py"]
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const app = express();

// Graceful shutdown handler - critical for zero-downtime deploys
let isShuttingDown = false;

process.on('SIGTERM', () => {
  console.log('SIGTERM received, starting graceful shutdown');
  isShuttingDown = true;
  
  // Give time for Kubernetes to remove pod from service endpoints
  setTimeout(() => {
    server.close(() => {
      console.log('Server closed, exiting');
      process.exit(0);
    });
  }, 5000);
});

// Liveness probe - is the app running?
app.get('/healthz', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).send('Shutting down');
  }
  res.status(200).send('OK');
});

// Readiness probe - can the app serve traffic?
app.get('/ready', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).send('Not ready');
  }
  // Check database, cache, external dependencies here
  res.status(200).send('Ready');
});

// Application endpoint demonstrating pod metadata
app.get('/api/metadata', (req, res) => {
  res.json({
    podName: process.env.HOSTNAME || 'unknown',
    podIP: process.env.POD_IP || 'unknown',
    namespace: process.env.POD_NAMESPACE || 'default',
    serviceAccount: process.env.SERVICE_ACCOUNT || 'default',
    uptime: process.uptime()
  });
});

const PORT = process.env.PORT || 3000;
const server = app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server listening on port ${PORT}`);
});
```

## ⚖️ When To Use / When To Avoid

**Use Kubernetes when:**
- You need to scale beyond a few servers and manual deployment becomes painful
- You're running microservices that need service discovery and load balancing
- You want infrastructure portability across clouds or on-premises
- Your team can invest in learning and maintaining K8s expertise
- You need advanced deployment strategies (canary, blue-green) built-in

**Avoid Kubernetes when:**
- You have a simple monolith that fits on one server comfortably
- Your team is small (<5 engineers) and can't dedicate time to K8s operations
- You're just starting a prototype or MVP—serverless is faster to market
- Your cloud provider's PaaS (Heroku, App Engine) meets your needs perfectly
- Operational complexity isn't justified by your scale or requirements

## 📚 Further Reading

- [Kubernetes Official Documentation - Concepts](https://kubernetes.io/docs/concepts/) - Start here for authoritative explanations of core concepts
- [Twelve-Factor App Methodology](https://12factor.net/) - Design principles that align perfectly with Kubernetes best practices
- [Kubernetes Patterns: Reusable Elements for Designing Cloud-Native Applications](https://www.oreilly.com/library/view/kubernetes-patterns/9781492050278/) - Beyond basics, learn proven architectural patterns
- [Kubernetes Python Client Documentation](https://github.com/kubernetes-client/python) - Programmatically interact with Kubernetes from Python
- [Ambassador Labs: Kubernetes Development Guide](https://www.getambassador.io/docs/telepresence/latest/quick-start/) - Tools and workflows for local Kubernetes development

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*