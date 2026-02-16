---
title: "Replace SSH Keys with Short-Lived Certificates"
linkTitle: "SSH Certificates"
description: "Eliminate authorized_keys management. Vouch issues SSH certificates that expire in 8 hours — no key distribution, no offboarding checklist."
weight: 3
subtitle: "Configure SSH servers to trust Vouch certificates for passwordless authentication"
params:
  docsGroup: infra
---

Managing SSH access with traditional public keys is one of the most painful parts of growing a team. Every new hire means copying keys to every server. Every departure means hunting down and removing keys you hope you can find. Keys never expire, and there is no way to know who used them.

Vouch eliminates this entirely. Administrators trust a single certificate authority (CA) key, and developers receive short-lived SSH certificates that expire after 8 hours. There is no key distribution, no `authorized_keys` sprawl, and no offboarding checklist. Developers authenticate with `vouch login` and then `ssh` into any trusted server without passwords or key copying.

## How it works

1. During `vouch login`, the Vouch server signs the developer's ephemeral public key with an **Ed25519 CA** key.
2. The resulting SSH certificate is valid for **8 hours** and contains the developer's **email address** and **username** as principals.
3. When the developer connects to a server, the SSH client presents the certificate.
4. The server verifies the certificate was signed by the trusted CA and that one of the certificate's principals matches an allowed user.
5. The connection is established without any `authorized_keys` lookup.

Because certificates expire after 8 hours, a compromised certificate is useless the next day. There is nothing to revoke and nothing to rotate.

---

## Step 1 -- Set up the CLI (for developers)

After completing the [Getting Started](/docs/getting-started/) guide, enable the SSH agent integration:

```bash
vouch setup ssh
```

This command configures your local SSH client to use the Vouch agent for certificate authentication. It adds the following to your `~/.ssh/config`:

```
Host *
  IdentityAgent ~/.vouch/ssh-agent.sock
```

After setup, every `ssh` connection will automatically use your Vouch certificate when available, falling back to regular keys if needed.

### Verify the agent is running

```bash
vouch status
```

Look for the SSH agent line in the output. If the agent is not running, `vouch login` will start it automatically.

### Check your certificate

After logging in, inspect the current certificate:

```bash
vouch ssh show
```

This displays the certificate's principals, validity period, and signing CA.

---

## Step 2 -- Configure SSH servers (for administrators)

To accept Vouch certificates, each server must trust the Vouch CA public key. Below are three approaches for deploying this configuration.

### Fetch the CA public key

First, retrieve your organization's CA public key from the Vouch server:

```bash
curl -s https://{{< instance-url >}}/ssh/ca.pub
```

Save the output -- you will need it for each method below.

---

### CLI (manual)

On each server, add the CA public key and configure `sshd`:

```bash
# Write the CA public key
echo "CONTENTS_OF_CA_PUB" | sudo tee /etc/ssh/vouch_ca.pub

# Tell sshd to trust certificates signed by this CA
# See: https://man.openbsd.org/sshd_config#TrustedUserCAKeys
echo "TrustedUserCAKeys /etc/ssh/vouch_ca.pub" | sudo tee -a /etc/ssh/sshd_config

# Optionally set AuthorizedPrincipalsFile to control which principals are allowed
# See: https://man.openbsd.org/sshd_config#AuthorizedPrincipalsFile
echo "AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u" | sudo tee -a /etc/ssh/sshd_config

# Create a principals file for a specific user
sudo mkdir -p /etc/ssh/auth_principals
echo "alice@example.com" | sudo tee /etc/ssh/auth_principals/alice

# Restart sshd
sudo systemctl restart sshd
```

---

### Ansible

```yaml
- name: Configure Vouch SSH CA trust
  hosts: all
  become: true
  vars:
    vouch_ca_pub: "{{ lookup('url', 'https://{{< instance-url >}}/ssh/ca.pub') }}"

  tasks:
    - name: Write Vouch CA public key
      copy:
        content: "{{ vouch_ca_pub }}"
        dest: /etc/ssh/vouch_ca.pub
        owner: root
        group: root
        mode: "0644"

    - name: Trust Vouch CA for user certificates
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^TrustedUserCAKeys"
        line: "TrustedUserCAKeys /etc/ssh/vouch_ca.pub"
      notify: restart sshd

    - name: Set AuthorizedPrincipalsFile
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^AuthorizedPrincipalsFile"
        line: "AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u"
      notify: restart sshd

    - name: Create auth_principals directory
      file:
        path: /etc/ssh/auth_principals
        state: directory
        owner: root
        group: root
        mode: "0755"

  handlers:
    - name: restart sshd
      service:
        name: sshd
        state: restarted
```

---

### Terraform (AWS EC2 user data)

```hcl
resource "aws_instance" "example" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    set -e

    # Fetch and install the Vouch CA public key
    curl -s https://{{< instance-url >}}/ssh/ca.pub \
      | tee /etc/ssh/vouch_ca.pub

    # Configure sshd to trust the CA
    echo "TrustedUserCAKeys /etc/ssh/vouch_ca.pub" >> /etc/ssh/sshd_config
    echo "AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u" >> /etc/ssh/sshd_config

    mkdir -p /etc/ssh/auth_principals

    systemctl restart sshd
  EOF

  tags = {
    Name = "vouch-ssh-example"
  }
}
```

---

## Tip: understanding principals

Vouch certificates include two principals by default:

| Principal | Example | Use case |
|-----------|---------|----------|
| Email address | `alice@example.com` | Unique per-user access control |
| Username | `alice` | Matches standard Unix usernames |

When configuring `AuthorizedPrincipalsFile`, you can list either the email or the username (or both) in the principals file for each Unix account.

**Example:** To allow both `alice@example.com` and `bob@example.com` to SSH as the `deploy` user:

```bash
sudo mkdir -p /etc/ssh/auth_principals
printf "alice@example.com\nbob@example.com\n" | sudo tee /etc/ssh/auth_principals/deploy
```

If you do not configure `AuthorizedPrincipalsFile`, OpenSSH will accept any certificate signed by the trusted CA. This is convenient for testing but is not recommended for production.

---

## Step 3 -- Test the connection

From a developer machine with an active Vouch session:

```bash
# Log in if you have not already
vouch login

# Connect to a configured server
ssh alice@server.example.com
```

The connection should succeed without prompting for a password or key passphrase. To verify that certificate authentication was used, check the server's auth log:

```bash
# On the server
sudo grep "Accepted certificate" /var/log/auth.log
```

You should see an entry like:

```
Accepted publickey for alice from 192.168.1.100 port 54321 ssh2: ED25519-CERT SHA256:... ID "alice@example.com" serial 42 CA ED25519 SHA256:...
```

---

## How certificate authentication works

Here is the full flow in detail:

1. **Login** -- The developer runs `vouch login`, authenticates with their YubiKey (PIN + touch), and receives a signed SSH certificate from the Vouch server's Ed25519 CA.

2. **Connection** -- The developer runs `ssh user@server`. The SSH client, configured to use the Vouch agent, presents the certificate to the server.

3. **CA verification** -- The server checks that the certificate is signed by a CA listed in `TrustedUserCAKeys`. If the signature is invalid or the CA is not trusted, the connection is rejected.

4. **Principal matching** -- The server checks whether any principal in the certificate matches an entry in the `AuthorizedPrincipalsFile` for the target Unix user. If no principals file is configured, any valid certificate is accepted.

5. **Session established** -- If both checks pass, the SSH session is established. The certificate's validity period (8 hours from login) is enforced -- expired certificates are rejected at step 3.

---

## Troubleshooting

### "Permission denied (publickey)"

- Confirm you have an active Vouch session: run `vouch status`.
- Verify the Vouch agent is running and `~/.ssh/config` includes the `IdentityAgent` line.
- Check that the server has `TrustedUserCAKeys` pointing to the correct CA public key.
- If using `AuthorizedPrincipalsFile`, verify that the file for the target user contains one of the certificate's principals (email or username).

### "Certificate has expired"

- SSH certificates issued by Vouch are valid for 8 hours. Run `vouch login` to get a fresh certificate.

### Agent not found

- Run `vouch login` to start the agent, or restart it with `vouch setup ssh`.
- Verify the socket path exists: `ls -la ~/.vouch/ssh-agent.sock`.

### Server rejects the certificate even though it was signed by the right CA

- Check `AuthorizedPrincipalsFile` permissions. The file and its parent directory must be owned by root and not writable by group or others.
- Ensure the principals file is in the correct location (`/etc/ssh/auth_principals/<username>`).
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
