# 📌 Monorepo vs polyrepo structure
*September 30, 2026 · Daily Dev Insight*

## 🧠 Overview

The monorepo versus polyrepo debate isn't just about where you store code—it's about how your team collaborates, ships features, and scales engineering culture. A monorepo houses multiple projects in a single repository with shared tooling and dependencies, while polyrepo distributes each project into its own repository with independent versioning and deployment pipelines.

The decision fundamentally shapes your development workflow. Monorepos excel at atomic cross-project changes and consistent tooling, making them favorites at Google, Meta, and Microsoft. You can refactor a shared library and update all consumers in one pull request. Polyrepos, championed by teams at Netflix and Amazon, provide strong boundaries, independent deployment cycles, and clearer ownership models. Each service owns its destiny without being blocked by unrelated changes.

Neither approach is universally superior—the right choice depends on team size, deployment frequency, and organizational structure. Small teams benefit from monorepo simplicity, while distributed organizations often need polyrepo independence. The key is understanding the trade-offs and tooling requirements before committing to either architecture.

## 💡 Key Concepts

- **Dependency management diverges dramatically**: Monorepos use workspace features (npm workspaces, Poetry) for shared internal packages, while polyrepos publish packages to registries and version them independently
- **CI/CD complexity inverts at scale**: Small monorepos have simpler CI, but large ones require sophisticated change detection and selective testing; polyrepos start complex but scale linearly
- **Code discovery vs. encapsulation**: Monorepos make finding and reusing code trivial but risk tight coupling; polyrepos enforce boundaries but can lead to code duplication
- **Tooling investment is non-negotiable**: Large monorepos demand custom build systems (Bazel, Nx, Turborepo), while polyrepos need robust package management and versioning strategies
- **Team autonomy follows structure**: Polyrepos enable independent team velocity but require strong API contracts; monorepos facilitate collaboration but can create bottlenecks

## 🐍 Python Example

```python
# Monorepo workspace manager for Python projects
# This script detects changed packages and runs tests selectively

import subprocess
import json
from pathlib import Path
from typing import Set

class MonorepoManager:
    def __init__(self, repo_root: Path):
        self.repo_root = repo_root
        self.packages_dir = repo_root / "packages"
    
    def get_changed_files(self, base_branch: str = "main") -> Set[str]:
        """Get files changed compared to base branch."""
        result = subprocess.run(
            ["git", "diff", "--name-only", f"{base_branch}...HEAD"],
            capture_output=True,
            text=True,
            cwd=self.repo_root
        )
        return set(result.stdout.strip().split("\n"))
    
    def get_affected_packages(self, changed_files: Set[str]) -> Set[str]:
        """Determine which packages are affected by changes."""
        affected = set()
        
        for file_path in changed_files:
            # Check if file is in a package directory
            if file_path.startswith("packages/"):
                package = file_path.split("/")[1]
                affected.add(package)
        
        # Also check dependency graph for downstream impacts
        for package in list(affected):
            affected.update(self.get_dependents(package))
        
        return affected
    
    def get_dependents(self, package: str) -> Set[str]:
        """Find packages that depend on the given package."""
        dependents = set()
        
        for pkg_dir in self.packages_dir.iterdir():
            if not pkg_dir.is_dir():
                continue
            
            pyproject = pkg_dir / "pyproject.toml"
            if pyproject.exists():
                # Simple dependency check (production would parse TOML)
                content = pyproject.read_text()
                if f'"{package}"' in content:
                    dependents.add(pkg_dir.name)
        
        return dependents

# Usage in CI pipeline
if __name__ == "__main__":
    manager = MonorepoManager(Path.cwd())
    changed = manager.get_changed_files()
    affected = manager.get_affected_packages(changed)
    
    print(f"Testing packages: {', '.join(affected)}")
    # Run tests only for affected packages
```

## 🟨 JavaScript Example

```javascript
// Polyrepo package versioning and release automation
// This script handles semantic versioning and cross-repo updates

const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

class PolyrepoReleaseManager {
  constructor(packagePath) {
    this.packagePath = packagePath;
    this.packageJson = JSON.parse(
      fs.readFileSync(path.join(packagePath, 'package.json'), 'utf8')
    );
  }

  // Analyze commits to determine version bump
  determineVersionBump() {
    const commits = execSync('git log $(git describe --tags --abbrev=0)..HEAD --oneline', {
      cwd: this.packagePath,
      encoding: 'utf8'
    });

    if (commits.match(/BREAKING CHANGE/)) return 'major';
    if (commits.match(/^feat:/m)) return 'minor';
    if (commits.match(/^fix:/m)) return 'patch';
    
    return null; // No release needed
  }

  // Bump version in package.json
  bumpVersion(type) {
    const [major, minor, patch] = this.packageJson.version.split('.').map(Number);
    
    const newVersion = type === 'major' 
      ? `${major + 1}.0.0`
      : type === 'minor'
      ? `${major}.${minor + 1}.0`
      : `${major}.${minor}.${patch + 1}`;

    this.packageJson.version = newVersion;
    
    fs.writeFileSync(
      path.join(this.packagePath, 'package.json'),
      JSON.stringify(this.packageJson, null, 2) + '\n'
    );

    return newVersion;
  }

  // Create git tag and publish to registry
  release(version) {
    execSync(`git tag v${version}`, { cwd: this.packagePath });
    execSync('git push --tags', { cwd: this.packagePath });
    execSync('npm publish --access public', { cwd: this.packagePath });
    
    console.log(`✅ Released ${this.packageJson.name}@${version}`);
  }
}

// Usage in CI/CD
const manager = new PolyrepoReleaseManager(process.cwd());
const bumpType = manager.determineVersionBump();

if (bumpType) {
  const newVersion = manager.bumpVersion(bumpType);
  manager.release(newVersion);
} else {
  console.log('No release necessary');
}
```

## ⚖️ When To Use / When To Avoid

**Use Monorepo when:**
- Team size is under 100 engineers
- You need atomic cross-project refactoring
- Sharing code between projects is frequent
- Consistent tooling and standards are priorities
- You can invest in build optimization tools

**Use Polyrepo when:**
- Multiple independent teams own different services
- Projects have different release cadences
- Access control and security boundaries are critical
- Teams need autonomy over their tech stack
- Your organization is geographically distributed

## 📚 Further Reading

- [Google's monorepo philosophy and tools](https://research.google/pubs/pub45424/) — Research paper on managing massive monorepos at scale
- [Nx documentation on monorepo tooling](https://nx.dev/concepts/more-concepts/why-monorepos) — Modern build system for TypeScript and JavaScript monorepos
- [Bazel build system guide](https://bazel.build/start) — Google's open-source build tool designed for monorepo performance
- [Microsoft's Rush Stack for polyrepo management