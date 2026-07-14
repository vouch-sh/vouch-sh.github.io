# Replace SSH Keys with Short-Lived Certificates

> Eliminate authorized_keys management. Vouch issues SSH certificates that expire in 8 hours — no key distribution, no offboarding checklist.

Source: https://vouch.sh/docs/ssh/
Last updated: 2026-07-04

---
&gt; **Windows:** SSH certificate integration is not available on Windows. See the [FAQ](/docs/faq/#does-vouch-work-on-windows) for details on Windows platform support.

Vouch replaces static SSH keys with short-lived certificates. Administrators trust a single certificate authority (CA) key, and developers receive SSH certificates that expire after 8 hours. There is no key distribution, no `authorized_keys` sprawl, and no offboarding checklist.

{{&lt; tldr &gt;}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → this page.
- **Admin, once:** [configure each SSH server](#step-2----configure-ssh-servers-for-administrators) to trust the Vouch CA public key.
- **Each developer:** `vouch setup ssh`, then `ssh user@server` just works.
{{&lt; /tldr &gt;}}

## Step 1 -- Set up the CLI (for developers)

{{&lt; role developer &gt;}}

Enable the SSH agent integration:

```bash
vouch setup ssh
```

This command configures your local SSH client to use the Vouch agent for certificate authentication. It adds the following to your `~/.ssh/config`:

```
Host *
  IdentityAgent /run/user/1000/vouch/ssh-agent.sock
```

The socket path is resolved for your platform: `$XDG_RUNTIME_DIR/vouch/ssh-agent.sock` on Linux, falling back to `~/.cache/vouch/ssh-agent.sock` where `XDG_RUNTIME_DIR` is unset (e.g. on macOS).

After setup, every `ssh` connection will automatically use your Vouch certificate when available, falling back to regular keys if needed.

### Verify the agent is running

```bash
vouch status
```

Look for the SSH agent line in the output. If the agent is not running, `vouch login` will start it automatically.

### Check your certificate

After logging in, inspect the current certificate:

```bash
vouch credential ssh
```

This obtains a certificate from the server and displays the certificate&#39;s principals, validity period, and signing CA.

---

## Step 2 -- Configure SSH servers (for administrators)

{{&lt; role admin &gt;}}

To accept Vouch certificates, each server must trust the Vouch CA public key.

You can find the CA public key on the Integrations page of the Vouch dashboard:

![Integrations page showing SSH Certificates with CA public key](/images/admin/integrations.png)

### Fetch the CA public key

Or retrieve it programmatically from the Vouch server:

```bash
curl -s https://us.vouch.sh/v1/credentials/ssh/ca | jq -r &#39;.public_key&#39;
```

Save the output -- you will need it for each method below.

---


#### CLI (manual)

On each server, add the CA public key and create a drop-in `sshd` configuration file:

```bash
# Write the CA public key
echo &#34;CONTENTS_OF_CA_PUB&#34; | sudo tee /etc/ssh/vouch_ca.pub

# Configure sshd to trust the CA (drop-in config)
# See: https://man.openbsd.org/sshd_config#TrustedUserCAKeys
# See: https://man.openbsd.org/sshd_config#AuthorizedPrincipalsFile
sudo tee /etc/ssh/sshd_config.d/99-vouch.conf &lt;&lt;&#39;SSHD&#39;
TrustedUserCAKeys /etc/ssh/vouch_ca.pub
AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u
SSHD

# Create a principals file for a specific user
sudo mkdir -p /etc/ssh/auth_principals
echo &#34;alice@example.com&#34; | sudo tee /etc/ssh/auth_principals/alice

# Restart sshd
sudo systemctl restart sshd
```

#### Ansible

```yaml
- name: Configure Vouch SSH CA trust
  hosts: all
  become: true
  vars:
    vouch_ca_pub: &#34;{{ (lookup(&#39;url&#39;, &#39;https://us.vouch.sh/v1/credentials/ssh/ca&#39;) | from_json).public_key }}&#34;

  tasks:
    - name: Write Vouch CA public key
      copy:
        content: &#34;{{ vouch_ca_pub }}&#34;
        dest: /etc/ssh/vouch_ca.pub
        owner: root
        group: root
        mode: &#34;0644&#34;

    - name: Configure sshd to trust Vouch CA
      copy:
        content: |
          TrustedUserCAKeys /etc/ssh/vouch_ca.pub
          AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u
        dest: /etc/ssh/sshd_config.d/99-vouch.conf
        owner: root
        group: root
        mode: &#34;0644&#34;
      notify: restart sshd

    - name: Create auth_principals directory
      file:
        path: /etc/ssh/auth_principals
        state: directory
        owner: root
        group: root
        mode: &#34;0755&#34;

  handlers:
    - name: restart sshd
      service:
        name: sshd
        state: restarted
```

#### Terraform (AWS EC2 user data)

```hcl
resource &#34;aws_instance&#34; &#34;example&#34; {
  ami           = &#34;ami-0abcdef1234567890&#34;
  instance_type = &#34;t3.micro&#34;

  user_data = &lt;&lt;-EOF
    #!/bin/bash
    set -e

    # Fetch and install the Vouch CA public key
    curl -s https://us.vouch.sh/v1/credentials/ssh/ca \
      | jq -r &#39;.public_key&#39; | tee /etc/ssh/vouch_ca.pub

    # Configure sshd to trust the CA (drop-in config)
    cat &gt; /etc/ssh/sshd_config.d/99-vouch.conf &lt;&lt;&#39;SSHD&#39;
    TrustedUserCAKeys /etc/ssh/vouch_ca.pub
    AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u
    SSHD

    mkdir -p /etc/ssh/auth_principals

    systemctl restart sshd
  EOF

  tags = {
    Name = &#34;vouch-ssh-example&#34;
  }
}
```



---

## Tip: understanding principals

{{&lt; role admin &gt;}}

Vouch certificates include two principals by default:

| Principal | Example | Use case |
|-----------|---------|----------|
| Email address | `alice@example.com` | Unique per-user access control |
| Username | `alice` | Matches standard Unix usernames |

When configuring `AuthorizedPrincipalsFile`, you can list either the email or the username (or both) in the principals file for each Unix account.

**Example:** To allow both `alice@example.com` and `bob@example.com` to SSH as the `deploy` user:

```bash
sudo mkdir -p /etc/ssh/auth_principals
printf &#34;alice@example.com\nbob@example.com\n&#34; | sudo tee /etc/ssh/auth_principals/deploy
```

If you do not configure `AuthorizedPrincipalsFile`, OpenSSH will accept any certificate signed by the trusted CA. This is convenient for testing but is not recommended for production.

---

## Step 3 -- Test the connection

{{&lt; role developer &gt;}}

{{&lt; session-note &gt;}}

From a developer machine:

```bash
# Log in if you have not already
vouch login

# Connect to a configured server
ssh alice@server.example.com
```

The connection should succeed without prompting for a password or key passphrase. To verify that certificate authentication was used, check the server&#39;s auth log:

```bash
# On the server
sudo grep &#34;Accepted certificate&#34; /var/log/auth.log
```

You should see an entry like:

```
Accepted publickey for alice from 192.168.1.100 port 54321 ssh2: ED25519-CERT SHA256:... ID &#34;alice@example.com&#34; serial 42 CA ED25519 SHA256:...
```

---

## How it works

1. During `vouch login`, the Vouch server signs the developer&#39;s ephemeral public key with an **Ed25519 CA** key.
2. The resulting SSH certificate is valid for **8 hours** and contains the developer&#39;s **email address** and **username** as principals. It is cached by the local Vouch SSH agent for its lifetime, so subsequent `ssh` connections reuse it without contacting the Vouch server.
3. When the developer connects to a server, the SSH client presents the certificate.
4. The server verifies the certificate was signed by the trusted CA and that one of the certificate&#39;s principals matches an allowed user.
5. The connection is established without any `authorized_keys` lookup.

---

## Troubleshooting

### &#34;Permission denied (publickey)&#34;

- Confirm you have an active Vouch session: run `vouch status`.
- Verify the Vouch agent is running and `~/.ssh/config` includes the `IdentityAgent` line.
- Check that the server has `TrustedUserCAKeys` pointing to the correct CA public key.
- If using `AuthorizedPrincipalsFile`, verify that the file for the target user contains one of the certificate&#39;s principals (email or username).

### &#34;Certificate has expired&#34;

- SSH certificates issued by Vouch are valid for 8 hours. Run `vouch login` to get a fresh certificate.

### Agent not found

- Run `vouch login` to start the agent, or restart it with `vouch setup ssh`.
- Verify the socket exists at the path your SSH config points to: `grep IdentityAgent ~/.ssh/config`, then `ls -la` that path (`$XDG_RUNTIME_DIR/vouch/ssh-agent.sock`, or `~/.cache/vouch/ssh-agent.sock` on macOS).

### Server rejects the certificate even though it was signed by the right CA

- Check `AuthorizedPrincipalsFile` permissions. The file and its parent directory must be owned by root and not writable by group or others.
- Ensure the principals file is in the correct location (`/etc/ssh/auth_principals/&lt;username&gt;`).
- Review `/var/log/auth.log` (or `/var/log/secure` on RHEL-based systems) for detailed error messages.

### Connection falls back to password authentication

- The server may not have `TrustedUserCAKeys` configured, or `sshd` may not have been restarted after the configuration change.
- Run `ssh -v user@server` to see which authentication methods are attempted. Look for `Offering public key: ... ED25519-CERT` in the debug output.

---

## IDE Remote Development

Because Vouch configures `~/.ssh/config` with its agent, tools that build on top of SSH work automatically:

- **VS Code Remote-SSH** -- Open remote folders and terminals on any server trusted by the Vouch CA. No additional extension configuration is needed.
- **JetBrains Gateway** -- Connect to remote development environments using the same SSH certificate.
- **scp / rsync / sftp** -- File transfers use the Vouch SSH agent transparently.

As long as the Vouch agent is running and you have an active session, any tool that uses the system SSH client will authenticate with your Vouch certificate.

---

## Related guides

- [Getting Started](/docs/getting-started/) -- Install the CLI and enroll your YubiKey.
- [AWS Integration](/docs/aws/) -- Federate into AWS with OIDC for temporary STS credentials.
- [Amazon EKS](/docs/eks/) -- Authenticate to Kubernetes clusters running on EKS.
- [Security Model](/docs/security/) -- How Vouch protects credentials at every layer.
