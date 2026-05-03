# 📌 Monorepo vs polyrepo structure
*May 03, 2026 · Daily Dev Insight*

## 🧠 Overview

The eternal debate between monorepo and polyrepo structures isn't just about code organization—it's fundamentally about team dynamics, deployment strategies, and long-term maintainability. A monorepo houses multiple related projects in a single version-controlled repository, while polyrepo distributes each project into its own repository. This choice cascades through your entire development workflow, affecting everything from dependency management to CI/CD pipelines.

What makes this decision particularly nuanced is that both approaches have evolved significantly. Modern monorepo tooling like Nx, Lerna, and Bazel has solved many traditional pain points around build performance and selective testing. Meanwhile, polyrepo strategies have matured with better package management, semantic versioning, and microservice orchestration. The "right" choice increasingly depends on your team's collaboration patterns and the coupling between your services.

The key insight most teams miss: this isn't a binary choice. Hybrid approaches—like using monorepos for tightly coupled services while keeping distinct products in separate repos—often provide the best of both worlds.

## 💡 Key Concepts

• **Dependency boundaries**: Monorepos make refactoring across services trivial but can lead to tight coupling; polyrepos enforce loose coupling but make coordinated changes painful

• **Build and test efficiency**: Modern monorepo tools enable incremental builds and affected testing, while polyrepos require careful dependency caching strategies

• **Team autonomy vs coordination**: Polyrepos maximize team independence and release cadences, monorepos optimize for cross-team collaboration and consistency

• **Tooling complexity**: Monorepos require sophisticated tooling for scaling but provide unified developer experience; polyrepos have simpler per-repo tooling but complex inter-repo coordination

• **Security and access control**: Polyrepos offer granular access control per project; monorepos typically use folder-based permissions with less isolation

## 🐍 Python Example

```python
# monorepo_manager.py - Tool for managing Python monorepo dependencies
import subprocess
import json
import os
from pathlib import Path
from typing import Dict, List, Set

class MonorepoManager:
    def __init__(self, root_path: str):
        self.root = Path(root_path)
        self.services = self._discover_services()
    
    def _discover_services(self) -> Dict[str, Path]:
        """Discover all services with pyproject.toml files"""
        services = {}
        for toml_file in self.root.rglob("pyproject.toml"):
            if "services/" in str(toml_file.parent):
                service_name = toml_file.parent.name
                services[service_name] = toml_file.parent
        return services
    
    def get_dependency_graph(self) -> Dict[str, Set[str]]:
        """Build internal dependency graph between services"""
        graph = {}
        
        for service_name, service_path in self.services.items():
            graph[service_name] = set()
            
            # Parse pyproject.toml for internal dependencies
            toml_path = service_path / "pyproject.toml"
            with open(toml_path) as f:
                content = f.read()
                
            # Simple parsing for dependencies starting with our org
            for other_service in self.services:
                if f'myorg-{other_service}' in content:
                    graph[service_name].add(other_service)
        
        return graph
    
    def get_affected_services(self, changed_files: List[str]) -> Set[str]:
        """Get services affected by file changes (for selective testing)"""
        affected = set()
        dep_graph = self.get_dependency_graph()
        
        # Direct changes
        for file_path in changed_files:
            for service_name, service_path in self.services.items():
                if str(service_path) in file_path:
                    affected.add(service_name)
        
        # Propagate through dependency graph
        changed = True
        while changed:
            changed = False
            for service, deps in dep_graph.items():
                if deps & affected and service not in affected:
                    affected.add(service)
                    changed = True
        
        return affected

# Usage example
if __name__ == "__main__":
    manager = MonorepoManager("/workspace/backend-monorepo")
    changed_files = ["services/user-service/src/models.py"]
    affected = manager.get_affected_services(changed_files)
    print(f"Need to test: {affected}")
```

## 🟨 JavaScript Example

```javascript
// polyrepo-coordinator.js - Automate polyrepo dependency updates
const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');
const semver = require('semver');

class PolyrepoCoordinator {
  constructor(repoConfigs) {
    this.repos = repoConfigs; // Array of {name, path, dependencies}
  }

  /**
   * Check for outdated internal dependencies across repos
   */
  findOutdatedDependencies() {
    const versions = this.getCurrentVersions();
    const outdated = {};

    this.repos.forEach(repo => {
      const packageJson = JSON.parse(
        fs.readFileSync(path.join(repo.path, 'package.json'), 'utf8')
      );
      
      outdated[repo.name] = [];
      
      // Check dependencies against latest versions
      Object.entries(packageJson.dependencies || {}).forEach(([dep, currentVersion]) => {
        if (versions[dep] && semver.lt(
          semver.coerce(currentVersion), 
          versions[dep]
        )) {
          outdated[repo.name].push({
            package: dep,
            current: currentVersion,
            latest: versions[dep]
          });
        }
      });
    });

    return outdated;
  }

  /**
   * Get current published versions of all internal packages
   */
  getCurrentVersions() {
    const versions = {};
    
    this.repos.forEach(repo => {
      try {
        const packageJson = JSON.parse(
          fs.readFileSync(path.join(repo.path, 'package.json'), 'utf8')
        );
        versions[packageJson.name] = packageJson.version;
      } catch (error) {
        console.warn(`Could not read package.json for ${repo.name}`);
      }
    });

    return versions;
  }

  /**
   * Coordinate a breaking change across multiple repos
   */
  async coordinateBreakingChange(packageName, newVersion, migrationSteps) {
    console.log(`Coordinating ${packageName} update to ${newVersion}`);
    
    // 1. Update the source package
    const sourceRepo = this.repos.find(r => r.name === packageName);
    this.updatePackageVersion(sourceRepo.path, newVersion);
    
    // 2. Find all consumers
    const consumers = this.repos.filter(repo => 
      repo.dependencies?.includes(packageName)
    );

    // 3. Create update branches and PRs
    for (const consumer of consumers) {
      const branchName = `update-${packageName}-${newVersion}`;
      
      try {
        // Create branch
        execSync(`git checkout -b ${branchName}`, { 
          cwd: consumer.path 
        });
        
        // Run migration steps
        migrationSteps.forEach(step => {
          console.log(`Running: ${step} in ${consumer.name}`);
          execSync(step, { cwd: consumer.path });
        });
        
        // Update dependency version
        this.updateDependencyVersion(consumer.path, packageName, newVersion);
        
        console.log(`Updated ${consumer.name} for ${packageName}@${newVersion}`);
        
      } catch (error) {
        console.error(`Failed to update ${consumer.name}:`, error.message);
      }
    }
  }

  updatePackageVersion(repoPath, version) {
    const packagePath =