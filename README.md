[![Renovate](https://github.com/xdev-software/renovate-selfhosted/actions/workflows/renovate.yml/badge.svg)](https://github.com/xdev-software/renovate-selfhosted/actions/workflows/renovate.yml)

# renovate-selfhosted

## What is this?

**renovate-selfhosted** is a centralized, self-hosted [Renovate](https://docs.renovatebot.com/) bot implementation using GitHub Actions that automatically discovers and updates dependencies across multiple repositories in your organization. The added value is automated dependency management at scale without requiring individual repository setup or external services, providing you with better security, control, and cost-effectiveness compared to cloud-based solutions.

## Quick Start

Get started with automated dependency updates in 3 simple steps:

### 1. Fork or Clone This Repository
```bash
git clone https://github.com/xdev-software/renovate-selfhosted.git
cd renovate-selfhosted
```

### 2. Configure Secrets
Add the following secrets to your repository settings:
- `GH_TOKEN`: Personal Access Token with `repo` scope for creating PRs in target repositories
- `RENOVATE_GIT_PRIVATE_KEY` (optional): SSH private key for accessing private repositories

### 3. Customize Configuration
Edit `self-hosted.json5` to set your organization and preferences (replace with your actual values):
```json5
{
  "gitAuthor": "Your Renovate Bot <renovate@yourdomain.com>",  // Replace with your bot name and email
  "autodiscoverFilter": [
    "your-org/*"  // Replace with your organization or repository patterns
  ]
}
```

That's it! The workflow runs daily at 04:06 UTC, or you can trigger it manually via GitHub Actions.

## Usage

### For End Users (Repository Owners)

#### Adding Your Repository to Renovate
Your repository will be automatically discovered if it matches the patterns in `self-hosted.json5`'s `autodiscoverFilter`.

#### Configuring Renovate for Your Repository
Create a `renovate.json` or `renovate.json5` file in your repository root. Example:
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "rebaseWhen": "behind-base-branch"
}
```

See [Renovate documentation](https://docs.renovatebot.com/configuration-options/) for all configuration options.

#### What to Expect
- Renovate will scan your repository for dependencies (npm, pip, docker, etc.)
- It creates pull requests for outdated dependencies
- PRs include changelogs, release notes, and compatibility information
- You review and merge the PRs like any other pull request

### Manual Workflow Execution

You can manually trigger the Renovate workflow with custom options:

1. Go to **Actions** → **Renovate** → **Run workflow**
2. Choose options:
   - **Cache**: Enable, disable, or reset repository cache
   - **Log Level**: INFO or DEBUG for troubleshooting
   - **Repositories**: Specify exact repositories (e.g., `org/repo1,org/repo2`)

## Current State of Development

### ✅ Production Ready
This repository is **actively maintained** and **production-ready**. It currently supports:

- ✅ Automatic repository discovery via patterns
- ✅ Scheduled daily runs
- ✅ Manual workflow triggering with custom parameters
- ✅ Repository caching for improved performance
- ✅ Configurable concurrency and rate limiting
- ✅ Support for private repositories via SSH keys
- ✅ Flexible log levels for debugging

### 🚀 Current Version
- Renovate GitHub Action: `v43.0.20`
- Workflow runs on: `ubuntu-latest`

## Roadmap & Future Features

### Potential Enhancements
- 📊 **Enhanced Monitoring**: Integration with monitoring tools for better visibility into update status
- 🔔 **Notification System**: Slack/Teams notifications for failed runs or critical updates
- 📈 **Dashboard**: Web dashboard showing update statistics across all repositories
- 🔐 **Advanced Security**: Integration with vulnerability databases for security-first updates
- 🎯 **Smart Scheduling**: Adaptive scheduling based on repository activity
- 📦 **Package Grouping**: Automatic grouping of related dependency updates
- 🧪 **Pre-merge Testing**: Integration with CI/CD to automatically test updates before creating PRs
- 🔄 **Rollback Support**: Automatic rollback capabilities for problematic updates

### Epic Ideas
1. **Multi-Platform Support**: Extend beyond GitHub to GitLab, Bitbucket, and Azure DevOps
2. **AI-Powered Updates**: ML-based prioritization of updates based on impact and risk
3. **Compliance Dashboard**: Track license changes and compliance issues across dependencies
4. **Custom Update Strategies**: Repository-specific update strategies based on criticality

## Documentation

- [Developer Documentation](./DEVELOPER.md) - For contributors and customization
- [Official Renovate Docs](https://docs.renovatebot.com/) - Complete Renovate reference
- [Configuration Options](https://docs.renovatebot.com/configuration-options/) - All available settings

## Support & Troubleshooting

### Common Issues
- **No PRs created**: Check repository has a valid `renovate.json` and dependencies to update
- **Authentication errors**: Verify `GH_TOKEN` has correct permissions
- **Rate limiting**: Adjust `maxRequestsPerSecond` in `self-hosted.json5`

For detailed troubleshooting, see [DEVELOPER.md](./DEVELOPER.md) and [Renovate troubleshooting guide](https://docs.renovatebot.com/troubleshooting/).

## License

Licensed under [Apache License 2.0](./LICENSE). Copyright 2024 XDEV Software.

## Contributing

We welcome contributions! Please see [DEVELOPER.md](./DEVELOPER.md) for development guidelines.
