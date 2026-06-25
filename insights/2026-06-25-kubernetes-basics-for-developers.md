# 📌 Kubernetes basics for developers
*June 25, 2026 · Daily Dev Insight*

## 🧠 Overview

Kubernetes has evolved from a container orchestration buzzword into the de facto standard for running production applications at scale. But here's the thing most tutorials won't tell you upfront: you don't need to be a DevOps wizard to benefit from K8s. As a developer, understanding Kubernetes fundamentals means you can write applications that are inherently more scalable, resilient, and production-ready from day one.

At its core, Kubernetes is about declarative infrastructure. Instead of scripting deployment steps ("first do this, then do that"), you describe your desired state ("I want 3 replicas of this service, always running"), and Kubernetes continuously works to make reality match your specification. This shift in thinking changes how you design applications—suddenly, stateless services, health checks, and graceful shutdowns aren't nice-to-haves, they're essential patterns.

The learning curve is real, but the payoff is substantial. Once you grasp pods, deployments, services, and basic resource management, you'll understand why your app crashed in staging, why traffic isn't reaching your new feature, and how to debug production issues without SSH-ing into servers at 2 AM. Let's break down what actually matters for day-to-day development.

## 💡 Key Concepts

- **Pods are ephemeral**: The smallest deployable unit in K8s is a pod (one or more containers). Pods are disposable and replaceable—never store state in them. Design your apps to be killed and restarted at any moment.

- **Deployments manage desired state**: A Deployment wraps your pods with replica management, rolling updates, and rollback capabilities. This is what you'll use 90% of the time to run applications.

- **Services provide stable networking**: Pod IPs change constantly. Services give you a stable DNS name and load balancing across pod replicas. Understanding ClusterIP, NodePort, and LoadBalancer types is crucial.

- **ConfigMaps and Secrets externalize configuration**: Never bake credentials or environment-specific config into images. K8s provides first-class primitives for injecting configuration at runtime.

- **Resource limits prevent noisy neighbors**: Always set CPU and memory requests/limits. Without them, one misbehaving pod can starve everything else on the node.

## �🐍 Python Example

```python
# kubernetes-health-check-app.py
# A Flask app demonstrating K8s-ready health endpoints

from flask import Flask, jsonify
import os
import time
import signal
import sys

app = Flask(__name__)

# Track app readiness (simulates DB connection, etc.)
ready = False
shutdown_in_progress = False

@app.route('/healthz')
def liveness():
    """Liveness probe - is the app running?"""
    if shutdown_in_progress:
        return jsonify({"status": "shutting down"}), 503
    return jsonify({"status": "alive"}), 200

@app.route('/readyz')
def readiness():
    """Readiness probe - can the app handle traffic?"""
    if ready and not shutdown_in_progress:
        return jsonify({"status": "ready"}), 200
    return jsonify({"status": "not ready"}), 503

@app.route('/api/data')
def get_data():
    """Your actual business logic"""
    pod_name = os.getenv('POD_NAME', 'unknown')
    return jsonify({
        "message": "Hello from Kubernetes!",
        "pod": pod_name,
        "version": "1.0.0"
    })

def graceful_shutdown(signum, frame):
    """Handle SIGTERM from Kubernetes gracefully"""
    global shutdown_in_progress
    print("Received SIGTERM, starting graceful shutdown...")
    shutdown_in_progress = True
    # Give active requests time to complete
    time.sleep(5)
    sys.exit(0)

if __name__ == '__main__':
    # Register shutdown handler
    signal.signal(signal.SIGTERM, graceful_shutdown)
    
    # Simulate startup time (DB connections, cache warming, etc.)
    time.sleep(2)
    ready = True
    
    # K8s will inject POD_NAME via downward API
    port = int(os.getenv('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

## 🟨 JavaScript Example

```javascript
// kubernetes-client-example.js
// Interact with K8s API using the official client

const k8s = require('@kubernetes/client-node');
const express = require('express');

// Load K8s config (works both in-cluster and locally with kubeconfig)
const kc = new k8s.KubeConfig();
if (process.env.KUBERNETES_SERVICE_HOST) {
  kc.loadFromCluster(); // Running inside K8s
} else {
  kc.loadFromDefault(); // Local development
}

const k8sApi = kc.makeApiClient(k8s.CoreV1Api);
const app = express();

// Example: List all pods in current namespace
app.get('/api/pods', async (req, res) => {
  try {
    const namespace = process.env.POD_NAMESPACE || 'default';
    const response = await k8sApi.listNamespacedPod(namespace);
    
    const pods = response.body.items.map(pod => ({
      name: pod.metadata.name,
      status: pod.status.phase,
      restarts: pod.status.containerStatuses?.[0]?.restartCount || 0,
      node: pod.spec.nodeName
    }));
    
    res.json({ pods, count: pods.length });
  } catch (err) {
    console.error('K8s API error:', err);
    res.status(500).json({ error: 'Failed to fetch pods' });
  }
});

// Watch for pod events (useful for monitoring/automation)
app.get('/api/watch-pods', async (req, res) => {
  const namespace = process.env.POD_NAMESPACE || 'default';
  const watch = new k8s.Watch(kc);
  
  res.setHeader('Content-Type', 'text/event-stream');
  
  watch.watch(
    `/api/v1/namespaces/${namespace}/pods`,
    {},
    (type, pod) => {
      res.write(`data: ${JSON.stringify({
        type,
        name: pod.metadata.name,
        status: pod.status.phase
      })}\n\n`);
    },
    (err) => console.error('Watch error:', err)
  );
  
  req.on('close', () => watch.abort());
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`K8s dashboard running on port ${PORT}`);
});
```

## ⚖️ When To Use / When To Avoid

**When To Use:**
- You need to scale horizontally across multiple instances
- Your application requires high availability and zero-downtime deployments
- You're running microservices that need service discovery and load balancing
- You want infrastructure portability across cloud providers
- Your team is growing and needs standardized deployment practices

**When To Avoid:**
- You're running a simple monolith with predictable, low traffic (Platform-as-a-Service might be simpler)
- Your team is 1-2 developers without dedicated ops support (operational overhead is real)
- You have stateful, tightly-coupled legacy applications that can't be containerized easily
- Cost is a primary concern and you're not utilizing the cluster efficiently (managed K8s isn't cheap)

## 📚 Further Reading

- [Kubernetes Official Documentation - Concepts](https://kubernetes.io/docs/concepts/) — Start here for authoritative explanations of all core primitives
- [Kubernetes Patterns eBook](https://www.redhat.com/en/resources/oreilly-kubernetes-patterns-scalable-apps) — Real-world patterns for designing cloud-native apps
- [Python Kubernetes Client Documentation](https://github.com/kubernetes-client/python) — Official Python library for K8s API interaction
- [12 