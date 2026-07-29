# Authenticate to Container Registries without Stored Passwords

> Stop running docker login and storing plaintext credentials. Vouch generates registry tokens on demand for ECR and GHCR.

Source: https://vouch.sh/docs/docker/
Last updated: 2026-07-04

---
Vouch&#39;s [credential helper](https://docs.docker.com/engine/reference/commandline/login/#credential-helpers) generates registry tokens on demand -- no stored passwords, no refresh scripts, and no `docker login`. After a single `vouch login`, Docker pulls and pushes to supported registries authenticate automatically.

{{&lt; tldr &gt;}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → this page; the ECR path also requires the [AWS integration](/docs/aws/).
- **Admin, once:** grant the [registry-specific permissions](#registry-specific-setup) -- ECR actions on the Vouch IAM role, or `packages:read` on the GitHub App.
- **Each developer:** `vouch setup docker --configure &lt;registry&gt;`, then `docker pull` and `docker push` just work.
{{&lt; /tldr &gt;}}

## Supported Registries

| Registry | Domain Pattern | Token Type |
|---|---|---|
| **AWS ECR** | `&lt;account-id&gt;.dkr.ecr.&lt;region&gt;.amazonaws.com` | AWS STS temporary credentials |
| **GitHub Container Registry** | `ghcr.io` | GitHub installation access token |

---

## Prerequisites

{{&lt; role admin &gt;}}

Before developers can configure the Docker integration:

- For **ECR**: AWS IAM must be configured to trust the Vouch OIDC provider (see [AWS Integration](/docs/aws/))
- For **GHCR**: A GitHub organization must be connected to the Vouch server (see [GitHub Integration](/docs/github/))

---

## Step 1 -- Configure Docker Credential Helper

{{&lt; role developer &gt;}}

With Docker installed and running, run the setup command to install the Vouch Docker credential helper:

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
  &#34;credHelpers&#34;: {
    &#34;ghcr.io&#34;: &#34;vouch&#34;,
    &#34;123456789012.dkr.ecr.us-east-1.amazonaws.com&#34;: &#34;vouch&#34;
  }
}
```

You can configure multiple registries by running the command once for each registry domain.

---

## Step 2 -- Use Docker normally

{{&lt; role developer &gt;}}

{{&lt; session-note &gt;}}

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

{{&lt; role admin &gt;}}

### AWS ECR

For ECR authentication, Vouch uses your AWS integration to obtain temporary STS credentials, which are then exchanged for an ECR authorization token.

**IAM Policy** -- The IAM role assumed by Vouch must include ECR permissions. At minimum, the role needs:

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Action&#34;: [
        &#34;ecr:GetAuthorizationToken&#34;,
        &#34;ecr:BatchGetImage&#34;,
        &#34;ecr:GetDownloadUrlForLayer&#34;,
        &#34;ecr:BatchCheckLayerAvailability&#34;
      ],
      &#34;Resource&#34;: &#34;*&#34;
    }
  ]
}
```

For push access, add these additional actions:

```json
{
  &#34;Effect&#34;: &#34;Allow&#34;,
  &#34;Action&#34;: [
    &#34;ecr:PutImage&#34;,
    &#34;ecr:InitiateLayerUpload&#34;,
    &#34;ecr:UploadLayerPart&#34;,
    &#34;ecr:CompleteLayerUpload&#34;
  ],
  &#34;Resource&#34;: &#34;arn:aws:ecr:us-east-1:123456789012:repository/*&#34;
}
```

See the [AWS Integration](/docs/aws/) guide for full IAM configuration details.

### GitHub Container Registry (GHCR)

For GHCR authentication, Vouch uses the same GitHub App integration used for Git access. The token issued is scoped to the packages your organization has granted access to.

**Token scope** -- The GitHub App must have the `packages:read` permission (and `packages:write` if you need push access). Your organization administrator can configure this when installing the Vouch GitHub App.

See the [GitHub Integration](/docs/github/) guide for GitHub App setup details.

---

## How it works

1. **Docker requests credentials** -- The Docker daemon calls the `docker-credential-vouch` helper when it needs to authenticate to a configured registry.
2. **Vouch exchanges your session** -- The credential helper contacts the Vouch server and exchanges your active hardware-backed session for registry-specific credentials.
3. **Short-lived token issued** -- The Vouch server returns a temporary token appropriate for the target registry (an AWS STS token for ECR, or a GitHub token for GHCR). Tokens are generated on demand and never persisted to disk.
4. **Docker authenticates** -- The token is passed back to Docker and used for the current pull or push operation.

---

## Troubleshooting

### docker-credential-vouch not found

```
error getting credentials - err: exec: &#34;docker-credential-vouch&#34;: executable file not found in $PATH
```

The `docker-credential-vouch` binary is not in your `PATH`. This binary is included with the Vouch CLI installation. Verify:

1. The Vouch CLI is installed: `vouch --version`
2. The credential helper binary exists: `which docker-credential-vouch`
3. If the binary is missing, reinstall the Vouch CLI or ensure the installation directory is in your `PATH`.

### Authentication failed

```
Error response from daemon: Head &#34;https://ghcr.io/v2/...&#34;: denied
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
error: registry &#34;registry.example.com&#34; is not supported by Vouch
```

Vouch currently supports AWS ECR and GitHub Container Registry (ghcr.io). Other registries are not yet supported. Check the [supported registries](#supported-registries) table above.

### Credentials not being used

If Docker appears to ignore the Vouch credential helper and prompts for a username and password:

1. Check for conflicting `credsStore` settings in `~/.docker/config.json`. A top-level `credsStore` may override individual `credHelpers` entries.
2. Remove or rename any conflicting credential store:
   ```json
   {
     &#34;credsStore&#34;: &#34;&#34;,
     &#34;credHelpers&#34;: {
       &#34;ghcr.io&#34;: &#34;vouch&#34;
     }
   }
   ```
3. Ensure there are no cached credentials for the registry. Run `docker logout &lt;registry&gt;` to clear any stored credentials, then retry.

---

## Helm &amp; Compatible Tools

### Helm

Helm 3.8&#43; supports OCI registries for chart storage. Because Helm reads the Docker credential store, charts hosted in ECR or GHCR authenticate through Vouch automatically:

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
