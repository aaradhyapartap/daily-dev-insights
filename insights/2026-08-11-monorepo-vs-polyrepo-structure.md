# 📌 Monorepo vs polyrepo structure
*August 11, 2026 · Daily Dev Insight*

## 🧠 Overview

The monorepo versus polyrepo debate isn't just about where you store code—it's about how your team communicates, ships features, and manages complexity at scale. A monorepo houses multiple projects in a single repository (think Google's legendary ~2 billion line codebase), while polyrepos split each project into its own repository. Both approaches are valid, but they optimize for fundamentally different problems.

The real insight here is that your choice reflects your organizational structure more than your technical needs. Monorepos excel when you need atomic cross-project changes, shared tooling, and a single source of truth. They force coordination and visibility but can become slow and unwieldy without proper tooling. Polyrepos offer independence and clear boundaries, making them ideal for microservices or distributed teams, but at the cost of potential version hell and duplicated effort.

Here's what most teams get wrong: they choose based on hype rather than their actual collaboration patterns. If your frontend team never talks to your backend team, a monorepo won't magically fix that—it'll just make merges more painful. The best architecture is the one that matches how your humans already work together.

## 💡 Key Concepts

- **Atomic changes**: Monorepos allow you to update multiple projects in a single commit, ensuring consistency across your stack without complex versioning
- **Tooling investment**: Monorepos demand sophisticated build systems (Bazel, Nx, Turborepo) to avoid rebuilding everything on every change
- **Ownership boundaries**: Polyrepos enforce clear ownership through repository permissions, while monorepos use CODEOWNERS files and trust
- **Dependency management**: Monorepos typically use a single version of each dependency (avoiding "dependency hell"), while polyrepos allow each project to evolve independently
- **CI/CD complexity**: Monorepos need intelligent change detection to test only affected projects; polyrepos have simpler CI but harder cross-repo testing

## 🐍 Python Example

```python
# monorepo-tools/affected_projects.py
# Detects which projects changed in a monorepo commit for smart CI/CD

import subprocess
import json
from pathlib import Path
from typing import Set, Dict, List

def get_changed_files(base_branch: str = "main") -> Set[str]:
    """Get all files changed compared to base branch."""
    result = subprocess.run(
        ["git", "diff", "--name-only", f"origin/{base_branch}...HEAD"],
        capture_output=True,
        text=True,
        check=True
    )
    return set(result.stdout.strip().split("\n"))

def load_project_config() -> Dict[str, List[str]]:
    """Load project definitions from monorepo config."""
    config_path = Path("monorepo.json")
    with open(config_path) as f:
        return json.load(f)["projects"]

def find_affected_projects(changed_files: Set[str], 
                          projects: Dict[str, List[str]]) -> Set[str]:
    """Determine which projects are affected by file changes."""
    affected = set()
    
    for project_name, project_paths in projects.items():
        for changed_file in changed_files:
            # Check if any changed file is under this project's paths
            if any(changed_file.startswith(path) for path in project_paths):
                affected.add(project_name)
                break
    
    return affected

def main():
    """Run affected project detection for CI optimization."""
    changed_files = get_changed_files()
    projects = load_project_config()
    affected = find_affected_projects(changed_files, projects)
    
    print(f"Changed files: {len(changed_files)}")
    print(f"Affected projects: {', '.join(sorted(affected))}")
    
    # Output for CI system to consume
    with open("affected-projects.txt", "w") as f:
        f.write("\n".join(sorted(affected)))

if __name__ == "__main__":
    main()
```

## 🟨 JavaScript Example

```javascript
// polyrepo-tools/sync-versions.js
// Keeps package versions synchronized across polyrepo projects

const fs = require('fs').promises;
const path = require('path');
const { Octokit } = require('@octokit/rest');

class PolyrepoVersionManager {
  constructor(githubToken, org) {
    this.octokit = new Octokit({ auth: githubToken });
    this.org = org;
  }

  async getRepositories(pattern) {
    // Get all repos matching a naming pattern
    const { data } = await this.octokit.repos.listForOrg({
      org: this.org,
      type: 'sources'
    });
    
    return data.filter(repo => 
      new RegExp(pattern).test(repo.name)
    );
  }

  async updatePackageVersion(repo, dependency, newVersion) {
    // Read package.json from each repo
    const { data } = await this.octokit.repos.getContent({
      owner: this.org,
      repo: repo.name,
      path: 'package.json'
    });

    const packageJson = JSON.parse(
      Buffer.from(data.content, 'base64').toString()
    );

    // Update the dependency version
    if (packageJson.dependencies?.[dependency]) {
      packageJson.dependencies[dependency] = newVersion;
      
      // Create a PR with the change
      console.log(`Updating ${dependency} to ${newVersion} in ${repo.name}`);
      return packageJson;
    }
  }

  async syncDependency(repoPattern, dependency, targetVersion) {
    const repos = await this.getRepositories(repoPattern);
    const updates = [];

    for (const repo of repos) {
      const result = await this.updatePackageVersion(
        repo, 
        dependency, 
        targetVersion
      );
      if (result) updates.push(repo.name);
    }

    return updates;
  }
}

// Usage
const manager = new PolyrepoVersionManager(
  process.env.GITHUB_TOKEN,
  'your-org'
);

manager.syncDependency('service-.*', 'lodash', '^4.17.21')
  .then(updated => console.log(`Updated ${updated.length} repos`));
```

## ⚖️ When To Use / When To Avoid

**Use Monorepo when:**
- You need frequent cross-project refactoring (shared libraries, design systems)
- Your team is co-located or tightly coordinated
- You have the resources to invest in build tooling (Bazel, Nx, Turborepo)
- Code reuse and consistency are top priorities

**Use Polyrepo when:**
- Projects have genuinely independent lifecycles and release schedules
- Teams are autonomous with clear service boundaries (microservices architecture)
- You need strict access control per project
- Onboarding simplicity matters (new devs clone only what they need)

**Avoid Monorepo when:**
- Your CI/CD can't handle smart incremental builds
- You have very large binary assets (media, ML models)
- Teams actively resist collaboration

**Avoid Polyrepo when:**
- You're constantly updating shared dependencies across repos
- Breaking changes cascade through multiple repositories
- You spend more time managing versions than writing code

## 📚 Further Reading

- [Google's approach to monorepo development](https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/fulltext) – The canonical case study from the team managing the world's largest monorepo
- [Nx monorepo documentation](https://nx.dev/concepts/more-concepts/why-monorepos) – Modern tooling designed specifically for JavaScript/TypeScript monorepos
- [ThoughtWorks Technology Radar on monorepos](https://www.thoughtworks.com/radar/techniques/monorepos) – Balanced industry perspective on when monorepos make sense
- [Git submodules documentation](https://git-