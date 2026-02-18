---
title: "Making AWS CodeArtifact Work for Every Package Manager"
description: "CodeArtifact token rotation is the #1 complaint. Vouch eliminates the cron jobs and makes CodeArtifact as easy as any public registry."
date: 2026-02-12
---

AWS CodeArtifact is a managed package repository that supports npm, pip, Maven, Cargo, NuGet, and more. It integrates with IAM for access control and stores packages in your own AWS account. For teams that need private packages without running Artifactory or Nexus, it's an attractive option.

But the developer experience has a problem: **token rotation**.

## The token problem

CodeArtifact uses authorization tokens that expire after 12 hours by default. To get a token, you run:

```bash
aws codeartifact get-authorization-token \
  --domain my-domain \
  --domain-owner 123456789012 \
  --query authorizationToken \
  --output text
```

This gives you a bearer token that you then configure in your package manager:

```bash
# npm
npm config set //my-domain-123456789012.d.codeartifact.us-east-1.amazonaws.com/npm/my-repo/:_authToken=$TOKEN

# pip
pip install my-package --index-url https://aws:$TOKEN@my-domain-123456789012.d.codeartifact.us-east-1.amazonaws.com/pypi/my-repo/simple/

# Cargo
# (even more complicated -- requires editing credentials.toml)
```

Every 12 hours, the token expires. Every package manager has a different configuration mechanism. The result is:

1. **Cron jobs** to refresh tokens. These break silently when AWS credentials expire.
2. **Wrapper scripts** like `refresh-codeartifact.sh` that every developer has to remember to run.
3. **Stale tokens** that cause cryptic authentication failures in the middle of `npm install`.
4. **Teams giving up** and using npmjs.com, PyPI, or crates.io public registries instead of private ones.

## What teams actually do

In practice, most teams solve the token rotation problem with one of these patterns:

**The cron job:** A shell script that runs every 6 hours, fetches a new token, and updates `.npmrc` or `pip.conf`. This works until someone's AWS credentials expire, the cron job fails silently, and `npm install` breaks at 3 PM.

**The login script:** A function in `.zshrc` that fetches a new token at shell startup. This works until someone opens a new terminal tab and forgets to run it, or keeps a terminal open for more than 12 hours.

**The CI/CD workaround:** A pipeline step that fetches a fresh token before every `npm install`. This works for CI but doesn't help developers running builds locally.

None of these are good. Package managers should authenticate transparently, the way `git push` works with GitHub.

## Vouch makes it transparent

With Vouch, CodeArtifact authentication works the same way AWS CLI authentication works -- through a credential helper that fetches tokens on demand:

```bash
# One-time setup
vouch setup codeartifact --tool cargo --repository my-repo --domain my-domain --domain-owner 123456789012

# Daily use
vouch login           # One YubiKey tap, 8-hour session
cargo build           # Token fetched automatically
pip install my-pkg    # Token fetched automatically
```

### How it works

1. When a package manager needs to authenticate, Vouch's credential helper intercepts the request.
2. Vouch exchanges your active hardware-backed session for temporary STS credentials.
3. It calls `codeartifact:GetAuthorizationToken` to get a fresh CodeArtifact token.
4. The token is returned to the package manager for the current operation.

No cron jobs. No wrapper scripts. No stale tokens. The token is fetched on demand, every time.

### Per-package-manager behavior

| Package manager | Token model | Developer experience |
|---|---|---|
| **Cargo** | Dynamic (fetched on demand via credential provider) | Fully transparent. `cargo build` just works. |
| **pip** | Dynamic (embedded in index URL via credential helper) | Fully transparent. `pip install` just works. |
| **npm** | Static (written to `.npmrc`, 12h expiry) | Re-run `vouch setup codeartifact --tool npm` when token expires. |
| **Maven** | Environment variable (`CODEARTIFACT_AUTH_TOKEN`) | Use `vouch exec --type codeartifact -- mvn deploy`. |

Cargo and pip get the best experience because their credential systems support dynamic token fetching. npm's `.npmrc` format requires a static token, but even there, refreshing is a single command instead of a manual multi-step process.

## The broader picture

CodeArtifact token rotation is a symptom of a larger problem: AWS services that use IAM authentication are harder to use than their third-party equivalents, not because of features, but because of credential UX.

A startup choosing between CodeArtifact and a hosted npm registry shouldn't have to factor in "how hard is credential management." With Vouch, they don't.

The same `vouch login` that handles CodeArtifact tokens also provides:

- **AWS CLI credentials** via `credential_process`
- **CodeCommit Git auth** via SigV4 signing
- **ECR Docker auth** via `credential-helper`
- **SSH certificates** via the SSH agent
- **GitHub tokens** via installation token exchange

One authentication event, one tool, every credential handled.

## Getting started

See the [CodeArtifact integration guide](https://vouch.sh/docs/codeartifact/) for setup instructions covering Cargo, pip, npm, and Maven.
