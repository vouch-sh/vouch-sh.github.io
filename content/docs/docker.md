---
title: "Authenticate to Container Registries without Stored Passwords"
linkTitle: "Docker Registries"
description: "Stop running docker login and storing plaintext credentials. Vouch generates registry tokens on demand for ECR and GHCR."
weight: 6
subtitle: "Authenticate to container registries using Vouch"
params:
  docsGroup: code
---

Container registry credentials are a frequent source of leaks. Docker stores them in plaintext in `~/.docker/config.json`, and ECR's `get-login-password` tokens require a cron job or wrapper script to refresh every 12 hours. If you've ever committed a `.docker/config.json` to a dotfiles repo, those credentials are permanently exposed.

Vouch's [credential helper](https://docs.docker.com/engine/reference/commandline/login/#credential-helpers) generates tokens on demand -- no stored passwords, no refresh scripts, and full auditability through [ECR authentication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html) logs. After a single `vouch login`, you can pull and push container images to supported registries without running `docker login` or managing any secrets.

## How it works

When Docker needs to authenticate to a container registry, it calls the Vouch credential helper instead of looking up stored passwords:

1. **Docker requests credentials** -- The Docker daemon calls the `docker-credential-vouch` helper when it needs to authenticate to a configured registry.
2. **Vouch exchanges your session** -- The credential helper contacts the Vouch server and exchanges your active hardware-backed session for registry-specific credentials.
3. **Short-lived token issued** -- The Vouch server returns a temporary token appropriate for the target registry (an AWS STS token for ECR, or a GitHub token for GHCR).
4. **Docker authenticates** -- The token is passed back to Docker and used for the current pull or push operation.

Key characteristics:

- **Short-lived tokens** -- Credentials are generated on demand and expire quickly. They are never persisted to disk.
- **No `docker login` required** -- The credential helper handles authentication transparently.
- **Multiple registries** -- You can configure Vouch for multiple registries simultaneously.

---

## Supported Registries

| Registry | Domain Pattern | Token Type |
|---|---|---|
| **AWS ECR** | `<account-id>.dkr.ecr.<region>.amazonaws.com` | AWS STS temporary credentials |
| **GitHub Container Registry** | `ghcr.io` | GitHub installation access token |

---

## Prerequisites

Before configuring the Docker integration, make sure you have:

- The **Vouch CLI** installed and enrolled (see [Getting Started](/docs/getting-started/))
- **Docker** installed and running
- For **ECR**: Your organization administrator has configured AWS IAM to trust the Vouch OIDC provider (see [AWS Integration](/docs/aws/))
- For **GHCR**: Your organization administrator has connected a GitHub organization to the Vouch server (see [GitHub Integration](/docs/github/))

---

## Step 1 -- Configure Docker Credential Helper

Run the setup command to install the Vouch Docker credential helper:

```
vouch setup docker
```

This prints the Docker configuration that will be added. To apply it automatically for a specific registry:

```bash
# Configure for GitHub Container Registry
vouch setup docker --configure ghcr.io

# Configure for AWS ECR
vouch setup docker --configure 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

The command updates your `~/.docker/config.json` to register the Vouch credential helper for the specified registry:

```json
{
  "credHelpers": {
    "ghcr.io": "vouch",
    "123456789012.dkr.ecr.us-east-1.amazonaws.com": "vouch"
  }
}
```

You can configure multiple registries by running the command once for each registry domain.

---

## Step 2 -- Authenticate

If you have not already logged in today, authenticate with your YubiKey:

```
vouch login
```

Your session lasts for 8 hours. All Docker operations during that window use the session automatically.

---

## Step 3 -- Use Docker normally

With the credential helper configured and an active session, Docker commands work without any extra flags or manual login:

```bash
# Pull from GitHub Container Registry
docker pull ghcr.io/your-org/your-image:latest

# Pull from AWS ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/your-repo:latest

# Push to GitHub Container Registry
docker push ghcr.io/your-org/your-image:v1.2.3

# Push to AWS ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/your-repo:v1.2.3
```

Vouch handles authentication transparently. You do not need to run `docker login` or `aws ecr get-login-password`.

---

## Registry-Specific Setup

### AWS ECR

For ECR authentication, Vouch uses your AWS integration to obtain temporary STS credentials, which are then exchanged for an ECR authorization token.

**IAM Policy** -- The IAM role assumed by Vouch must include ECR permissions. At minimum, the role needs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "*"
    }
  ]
}
```

For push access, add these additional actions:

```json
{
  "Effect": "Allow",
  "Action": [
    "ecr:PutImage",
    "ecr:InitiateLayerUpload",
    "ecr:UploadLayerPart",
    "ecr:CompleteLayerUpload"
  ],
  "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/*"
}
```

See the [AWS Integration](/docs/aws/) guide for full IAM configuration details.

### GitHub Container Registry (GHCR)

For GHCR authentication, Vouch uses the same GitHub App integration used for Git access. The token issued is scoped to the packages your organization has granted access to.

**Token scope** -- The GitHub App must have the `packages:read` permission (and `packages:write` if you need push access). Your organization administrator can configure this when installing the Vouch GitHub App.

See the [GitHub Integration](/docs/github/) guide for GitHub App setup details.

---

## Troubleshooting

### docker-credential-vouch not found

```
error getting credentials - err: exec: "docker-credential-vouch": executable file not found in $PATH
```

The `docker-credential-vouch` binary is not in your `PATH`. This binary is included with the Vouch CLI installation. Verify:

1. The Vouch CLI is installed: `vouch --version`
2. The credential helper binary exists: `which docker-credential-vouch`
3. If the binary is missing, reinstall the Vouch CLI or ensure the installation directory is in your `PATH`.

### Authentication failed

```
Error response from daemon: Head "https://ghcr.io/v2/...": denied
```

- Verify you have an active Vouch session: `vouch login`
- Check that the credential helper is configured for the registry: inspect `~/.docker/config.json` and confirm the registry appears in `credHelpers`.
- Ensure the Vouch agent is running: `vouch status`

### AWS not configured

```
error: AWS integration is not configured for this Vouch server
```

Your organization administrator has not set up the AWS OIDC provider. Ask them to complete the [AWS Integration](/docs/aws/) setup before using ECR through Vouch.

### Unsupported registry

```
error: registry "registry.example.com" is not supported by Vouch
```

Vouch currently supports AWS ECR and GitHub Container Registry (ghcr.io). Other registries are not yet supported. Check the [supported registries](#supported-registries) table above.

### Credentials not being used

If Docker appears to ignore the Vouch credential helper and prompts for a username and password:

1. Check for conflicting `credsStore` settings in `~/.docker/config.json`. A top-level `credsStore` may override individual `credHelpers` entries.
2. Remove or rename any conflicting credential store:
   ```json
   {
     "credsStore": "",
     "credHelpers": {
       "ghcr.io": "vouch"
     }
   }
   ```
3. Ensure there are no cached credentials for the registry. Run `docker logout <registry>` to clear any stored credentials, then retry.

---

## Helm & Compatible Tools

### Helm

Helm 3.8+ supports OCI registries for chart storage. Because Helm reads the Docker credential store, charts hosted in ECR or GHCR authenticate through Vouch automatically:

```bash
# Push a chart to ECR
helm push my-chart-0.1.0.tgz oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/charts

# Pull a chart from ECR
helm pull oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/charts/my-chart --version 0.1.0

# Install directly from OCI
helm install my-release oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/charts/my-chart
```

### Other compatible tools

Any tool that reads `~/.docker/config.json` credHelpers will use Vouch automatically:

- **crane** -- Image manipulation without a Docker daemon
- **skopeo** -- Copy images between registries
- **Podman** -- Daemonless container engine
- **buildah** -- OCI image builder
- **ORAS** -- OCI artifacts (Wasm modules, ML models, signatures)
