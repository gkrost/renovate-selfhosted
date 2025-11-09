# Developer Documentation

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [Configuration Deep Dive](#configuration-deep-dive)
- [Workflow Mechanics](#workflow-mechanics)
- [Customization Guide](#customization-guide)
- [Troubleshooting & Debugging](#troubleshooting--debugging)
- [Best Practices](#best-practices)
- [Contributing](#contributing)

## Architecture Overview

### High-Level Design
```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Actions Workflow                  │
│                    (Scheduled/Manual Trigger)                │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              Renovate Bot (Docker Container)                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  1. Autodiscover Repositories                         │  │
│  │  2. Clone Repository (with cache)                     │  │
│  │  3. Scan for Dependencies                             │  │
│  │  4. Check for Updates                                 │  │
│  │  5. Create/Update Pull Requests                       │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              Target Repositories (Discovered)                │
│  - Repositories matching autodiscoverFilter                  │
│  - PRs created for dependency updates                        │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

1. **GitHub Actions Workflow** (`renovate.yml`)
   - Triggers: Scheduled (daily) or manual
   - Manages caching for performance
   - Configures Renovate bot execution

2. **Renovate Bot** (via `renovatebot/github-action`)
   - Docker-based execution
   - Handles dependency detection and updates
   - Creates pull requests in target repositories

3. **Configuration Files**
   - `self-hosted.json5`: Global Renovate configuration
   - `renovate.json5`: Example repository-specific config

## Repository Structure

```
renovate-selfhosted/
├── .github/
│   └── workflows/
│       └── renovate.yml          # Main GitHub Actions workflow
├── LICENSE                        # Apache 2.0 license
├── README.md                      # End-user documentation
├── DEVELOPER.md                   # This file (developer docs)
├── renovate.json5                 # Example repo-specific config
└── self-hosted.json5              # Global Renovate configuration
```

### File Purposes

- **renovate.yml**: Defines the GitHub Actions workflow with scheduling, caching, and execution logic
- **self-hosted.json5**: Central configuration for the Renovate bot (author, filters, rate limits)
- **renovate.json5**: Example configuration for individual repositories

## Configuration Deep Dive

### Global Configuration (`self-hosted.json5`)

```json5
{
  // Bot identity in commits and PRs
  "gitAuthor": "XDEV Renovate Bot <renovate@xdev-software.de>",
  
  // Disable onboarding PRs (assumes repos already configured)
  "onboarding": false,
  
  // GitHub platform
  "platform": "github",
  
  // Rate limiting (0 = unlimited)
  "prHourlyLimit": 0,        // PRs per hour
  "prConcurrentLimit": 10,    // Concurrent PRs
  
  // Automatically discover repositories
  "autodiscover": true,
  "autodiscoverFilter": [
    "xdev-software/*",        // All repos in xdev-software org
    "RapidClipse/*"           // All repos in RapidClipse org
  ],
  
  // Performance and rate limiting for API calls
  "hostRules": [
    {
      "concurrentRequestLimit": 500,
      "maxRequestsPerSecond": 200,
      "enableHttp2": true
    }
  ]
}
```

#### Key Configuration Options

| Option | Description | Recommended Value |
|--------|-------------|-------------------|
| `gitAuthor` | Identity for commits | `"YourBot <email@domain.com>"` |
| `onboarding` | Create onboarding PR | `false` (for pre-configured repos) |
| `prHourlyLimit` | Max PRs per hour | `0` (unlimited) or `5-10` |
| `prConcurrentLimit` | Concurrent open PRs | `10` |
| `autodiscoverFilter` | Org/repo patterns | `["your-org/*"]` |
| `concurrentRequestLimit` | API concurrency | `500` |
| `maxRequestsPerSecond` | API rate limit | `200` (adjust based on token) |

### Repository Configuration (`renovate.json5`)

Individual repositories should contain their own `renovate.json` or `renovate.json5`:

```json5
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  
  // Rebase when behind base branch
  "rebaseWhen": "behind-base-branch",
  
  // Additional options (examples)
  "extends": ["config:base"],
  "timezone": "America/New_York",
  "schedule": ["before 5am on Monday"],
  "packageRules": [
    {
      "matchPackagePatterns": ["^@types/"],
      "groupName": "TypeScript definitions"
    }
  ]
}
```

See [Renovate Configuration Options](https://docs.renovatebot.com/configuration-options/) for all possibilities.

## Workflow Mechanics

### Scheduled Execution

The workflow runs daily at 04:06 UTC:
```yaml
schedule:
  - cron: '6 4 * * *'
```

**Why 04:06?** Off-peak hours, slightly offset from common times (00:00, 06:00) to reduce GitHub Actions load.

### Manual Execution

Trigger manually with custom parameters:
```yaml
workflow_dispatch:
  inputs:
    repoCache: [enabled|disabled|reset]
    logLevel: [INFO|DEBUG]
    repositories: "org/repo1,org/repo2"
```

### Caching Strategy

```yaml
# Cache directory used by Renovate
cache_dir: /tmp/renovate/cache/renovate/repository
cache_key: renovate-cache
```

**Cache Lifecycle:**
1. **Restore**: Load previous cache (if exists)
2. **Fix Permissions**: Adjust ownership for Docker container (UID 12021)
3. **Execute**: Run Renovate with cache
4. **Delete Old**: Remove previous cache
5. **Save New**: Store updated cache

**Benefits:**
- Faster repository cloning
- Reduced bandwidth usage
- Improved execution time (3-5x faster)

**Cache Reset:** Use manual trigger with `repoCache: reset` to clear corrupted cache.

### Permission Handling

The workflow requires specific permissions:
```yaml
permissions: 
  actions: write    # Delete old cache
  contents: read    # Read repository files
```

Target repositories need:
- `GH_TOKEN` with `repo` scope (create PRs, read repos)
- Optional: `RENOVATE_GIT_PRIVATE_KEY` for private repos

## Customization Guide

### Change Scheduling

Modify the cron expression in `renovate.yml`:
```yaml
schedule:
  - cron: '0 2 * * 1'  # 02:00 UTC every Monday
  - cron: '0 14 * * 5' # 14:00 UTC every Friday
```

Use [crontab.guru](https://crontab.guru/) to generate expressions.

### Adjust Rate Limits

For GitHub Enterprise or to be more conservative:
```json5
"hostRules": [
  {
    "concurrentRequestLimit": 100,      // Reduce concurrency
    "maxRequestsPerSecond": 50,         // Slower rate
    "enableHttp2": true
  }
]
```

### Add Repository Filters

Include or exclude specific repositories:
```json5
"autodiscoverFilter": [
  "myorg/*",              // All repos
  "!myorg/archived-*",    // Exclude archived
  "otherorg/specific-repo" // Specific repo
]
```

### Change Renovate Version

Update the action version in `renovate.yml`:
```yaml
- uses: renovatebot/github-action@v43.0.20  # Check releases for latest version
```

Check [renovatebot/github-action releases](https://github.com/renovatebot/github-action/releases).

### Enable Dependency Dashboard

Add to `self-hosted.json5`:
```json5
"dependencyDashboard": true,
"dependencyDashboardTitle": "Dependency Dashboard"
```

This creates a single issue in each repository showing all updates.

### Add Custom Package Rules

Filter updates by package type:
```json5
"packageRules": [
  {
    "matchDepTypes": ["devDependencies"],
    "automerge": true
  },
  {
    "matchPackagePatterns": ["^eslint"],
    "groupName": "ESLint packages"
  }
]
```

## Troubleshooting & Debugging

### Enable DEBUG Logging

1. **Manual Trigger**: Use workflow_dispatch with `logLevel: DEBUG`
2. **Default**: Add to `self-hosted.json5`:
   ```json5
   "logLevel": "debug"
   ```

### Common Issues

#### No Pull Requests Created

**Symptoms:** Workflow runs successfully, but no PRs appear

**Causes & Solutions:**
1. **No dependencies found**: Ensure repositories have `package.json`, `requirements.txt`, etc.
2. **No updates available**: All dependencies are already current
3. **Configuration error**: Check `renovate.json` syntax
4. **Rate limited**: Check logs for rate limit messages

**Debug Steps:**
```bash
# Check workflow logs
gh run view <run-id> --log

# Look for specific repository processing
gh run view <run-id> --log | grep "repository=your-org/your-repo"
```

#### Authentication Failures

**Symptoms:** "Authentication failed" or "403 Forbidden" errors

**Solutions:**
1. Verify `GH_TOKEN` has correct scopes:
   - ✅ `repo` (full control)
   - ✅ `workflow` (if updating workflow files)
2. Check token hasn't expired
3. Ensure bot has access to target repositories

#### Cache Permission Errors

**Symptoms:** "Permission denied" in cache directory

**Solution:** The workflow automatically fixes permissions, but if issues persist:
```yaml
# Add to workflow (already included)
- name: Try fix cache permissions
  run: sudo chown -R 12021:0 /tmp/renovate/
```

#### Rate Limiting

**Symptoms:** "rate limit exceeded" errors

**Solutions:**
1. Reduce `maxRequestsPerSecond` in `self-hosted.json5`
2. Add delay between repositories:
   ```json5
   "repositoryDelay": 5000  // 5000ms (5 seconds) between repos
   ```
3. Use authenticated GitHub App instead of PAT

### Advanced Debugging

#### Test Configuration Locally

Run Renovate locally with Docker:
```bash
export GITHUB_TOKEN="your-token"

docker run --rm \
  -v $(pwd)/self-hosted.json5:/usr/src/app/config.json5 \
  -e RENOVATE_TOKEN=$GITHUB_TOKEN \
  -e RENOVATE_REPOSITORIES=your-org/test-repo \
  -e LOG_LEVEL=debug \
  renovate/renovate:latest
```

#### Validate Configuration

Use Renovate's config validator:
```bash
npx --yes --package renovate -- renovate-config-validator
```

#### Analyze Workflow Runs

Check past runs programmatically:
```bash
# List recent runs
gh run list --workflow=renovate.yml

# View specific run with logs
gh run view <run-id> --log-failed

# Download artifacts (if any)
gh run download <run-id>
```

## Best Practices

### Security

1. **Use secrets for all tokens**: Never hardcode tokens in config files
2. **Limit token scope**: Only grant necessary permissions
3. **Rotate tokens regularly**: Update `GH_TOKEN` every 90 days
4. **Review PRs before merging**: Even automated updates need human review
5. **Enable branch protection**: Require reviews on critical repositories

### Performance

1. **Use caching**: Don't disable cache unless necessary
2. **Limit autodiscovery**: Be specific with `autodiscoverFilter`
3. **Batch updates**: Group related dependencies
4. **Schedule off-peak**: Run during low-activity hours

### Maintenance

1. **Monitor workflow runs**: Check for failures weekly
2. **Update Renovate action**: Keep action version current
3. **Review bot PRs**: Ensure quality and relevance
4. **Document custom rules**: Comment complex configurations
5. **Test changes**: Use manual trigger to test config changes

### Repository Setup

1. **Consistent config**: Use similar `renovate.json` across repos
2. **Enable automerge selectively**: Only for non-breaking updates
3. **Set up CI/CD**: Ensure tests run on Renovate PRs
4. **Use dependency dashboard**: Enable for visibility
5. **Group updates**: Reduce PR noise with package rules

## Contributing

### Making Changes

1. **Fork the repository**
2. **Create a feature branch**: `git checkout -b feature/my-enhancement`
3. **Test your changes**: Use manual workflow trigger
4. **Document changes**: Update README.md and this file
5. **Submit PR**: Describe the change and its purpose

### Code Review Checklist

- [ ] Configuration syntax is valid (JSON5)
- [ ] Changes are documented in README.md/DEVELOPER.md
- [ ] Workflow runs successfully with changes
- [ ] No secrets or sensitive data committed
- [ ] Follows existing code style and conventions

### Testing Changes

Before submitting:
1. Test with manual trigger
2. Verify cache functionality
3. Check logs for errors
4. Test with both INFO and DEBUG log levels
5. Ensure PRs are created correctly

### Release Process

This repository doesn't have formal releases, but major changes should:
1. Update Renovate action version if needed
2. Document breaking changes in PR
3. Test with multiple repositories
4. Notify users of required configuration changes

## Additional Resources

- [Renovate Documentation](https://docs.renovatebot.com/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [JSON5 Specification](https://json5.org/)
- [Cron Expression Generator](https://crontab.guru/)
- [Renovate GitHub Action](https://github.com/renovatebot/github-action)

## Support

For issues or questions:
1. Check [Renovate Troubleshooting](https://docs.renovatebot.com/troubleshooting/)
2. Review workflow logs
3. Open an issue in this repository
4. Consult [Renovate Community](https://github.com/renovatebot/renovate/discussions)
