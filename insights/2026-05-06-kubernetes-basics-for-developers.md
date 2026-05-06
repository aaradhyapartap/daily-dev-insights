# 📌 Kubernetes basics for developers
*May 06, 2026 · Daily Dev Insight*

## 🧠 Overview

Kubernetes isn't just infrastructure magic—it's a developer's best friend for building resilient, scalable applications. Think of it as your application's operating system in the cloud. While many developers see K8s as an ops concern, understanding its fundamentals will make you write better applications and debug production issues faster.

The real power of Kubernetes lies in its declarative approach: you describe what you want (5 replicas of this service, exposed on port 80), and K8s figures out how to make it happen. This abstraction layer means your code doesn't need to worry about which specific server it's running on, how to handle failures, or how to scale up during traffic spikes. You write your application logic, package it in a container, and let Kubernetes handle the complexity of distributed systems.

## 💡 Key Concepts

• **Pods**: The smallest deployable unit—usually one container plus shared storage/network. Think of it as a "wrapper" around your application instance
• **Deployments**: Manage multiple replicas of your application, handle rolling updates, and ensure desired state (if one pod crashes, spin up a replacement)
• **Services**: Stable network endpoints that route traffic to your pods, even as individual pods come and go
• **ConfigMaps & Secrets**: External configuration and sensitive data management, keeping your application code environment-agnostic
• **Namespaces**: Logical isolation within a cluster—like having separate "dev", "staging", and "prod" environments in the same infrastructure

## 🐍 Python Example

```python
# kubernetes_deploy.py - Deploy a Flask app with proper health checks
from kubernetes import client, config
import yaml

def create_flask_deployment():
    # Load kubeconfig (works in-cluster or locally)
    try:
        config.load_incluster_config()  # When running inside K8s
    except:
        config.load_kube_config()  # Local development
    
    apps_v1 = client.AppsV1Api()
    
    # Define deployment with health checks and resource limits
    deployment = {
        'apiVersion': 'apps/v1',
        'kind': 'Deployment',
        'metadata': {'name': 'flask-app', 'namespace': 'default'},
        'spec': {
            'replicas': 3,
            'selector': {'matchLabels': {'app': 'flask-app'}},
            'template': {
                'metadata': {'labels': {'app': 'flask-app'}},
                'spec': {
                    'containers': [{
                        'name': 'flask-app',
                        'image': 'myregistry/flask-app:v1.2.0',
                        'ports': [{'containerPort': 5000}],
                        # Critical: Always define resource limits
                        'resources': {
                            'requests': {'memory': '128Mi', 'cpu': '100m'},
                            'limits': {'memory': '256Mi', 'cpu': '500m'}
                        },
                        # Health checks prevent bad deployments
                        'livenessProbe': {
                            'httpGet': {'path': '/health', 'port': 5000},
                            'initialDelaySeconds': 30,
                            'periodSeconds': 10
                        },
                        'readinessProbe': {
                            'httpGet': {'path': '/ready', 'port': 5000},
                            'initialDelaySeconds': 5,
                            'periodSeconds': 5
                        },
                        # Environment config from ConfigMap
                        'envFrom': [{'configMapRef': {'name': 'flask-config'}}]
                    }]
                }
            }
        }
    }
    
    # Apply the deployment
    try:
        apps_v1.create_namespaced_deployment(
            namespace="default", 
            body=deployment
        )
        print("✅ Deployment created successfully")
    except client.exceptions.ApiException as e:
        if e.status == 409:  # Already exists
            apps_v1.patch_namespaced_deployment(
                name="flask-app",
                namespace="default",
                body=deployment
            )
            print("📦 Deployment updated")
        else:
            raise e

if __name__ == "__main__":
    create_flask_deployment()
```

## 🟨 JavaScript Example

```javascript
// k8s-service-discovery.js - Service discovery pattern for Node.js apps
const k8s = require('@kubernetes/client-node');
const express = require('express');

class KubernetesServiceDiscovery {
    constructor() {
        this.kc = new k8s.KubeConfig();
        // Auto-detect if we're in cluster or local development
        process.env.KUBERNETES_SERVICE_HOST 
            ? this.kc.loadFromCluster() 
            : this.kc.loadFromDefault();
        
        this.k8sApi = this.kc.makeApiClient(k8s.CoreV1Api);
        this.serviceCache = new Map();
    }
    
    // Discover service endpoints dynamically
    async getServiceEndpoints(serviceName, namespace = 'default') {
        const cacheKey = `${namespace}/${serviceName}`;
        
        try {
            const response = await this.k8sApi.readNamespacedEndpoints(
                serviceName, 
                namespace
            );
            
            const endpoints = [];
            response.body.subsets?.forEach(subset => {
                const port = subset.ports?.[0]?.port;
                subset.addresses?.forEach(addr => {
                    endpoints.push(`http://${addr.ip}:${port}`);
                });
            });
            
            this.serviceCache.set(cacheKey, endpoints);
            return endpoints;
            
        } catch (error) {
            console.warn(`Service discovery failed for ${serviceName}:`, error.message);
            return this.serviceCache.get(cacheKey) || [];
        }
    }
    
    // Health check endpoint that K8s probes will use
    setupHealthChecks(app) {
        app.get('/health', (req, res) => {
            // Check external dependencies, database connections, etc.
            res.status(200).json({ 
                status: 'healthy', 
                timestamp: new Date().toISOString(),
                uptime: process.uptime() 
            });
        });
        
        app.get('/ready', async (req, res) => {
            try {
                // Verify we can discover required services
                const dbEndpoints = await this.getServiceEndpoints('postgres-service');
                const redisEndpoints = await this.getServiceEndpoints('redis-service');
                
                if (dbEndpoints.length > 0 && redisEndpoints.length > 0) {
                    res.status(200).json({ status: 'ready' });
                } else {
                    res.status(503).json({ status: 'not ready', reason: 'dependencies unavailable' });
                }
            } catch (error) {
                res.status(503).json({ status: 'error', message: error.message });
            }
        });
    }
}

// Usage example
const app = express();
const serviceDiscovery = new KubernetesServiceDiscovery();

// Set up health checks (required for proper K8s integration)
serviceDiscovery.setupHealthChecks(app);

// Example API endpoint that uses service discovery
app.get('/api/data', async (req, res) => {
    const dbEndpoints = await serviceDiscovery.getServiceEndpoints('postgres-service');
    // Use one of the discovered endpoints for your database call
    res.json({ message: 'Service discovery working!', endpoints: dbEndpoints });
});

app.listen(3000, () => console.log('🚀 App ready for Kubernetes!'));
```

## ⚖️ When To Use / When To Avoid

**✅ Use Kubernetes when:**
• You have multiple microservices that need orchestration
• You need automatic scaling, rolling updates, and self-healing
• You're running in multiple environments (dev