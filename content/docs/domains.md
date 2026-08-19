---
title: "Email Domains & Issuer Subdomains"
linkTitle: "Email Domains"
description: "Verify additional email domains for your organization and claim a dedicated OIDC issuer subdomain with its own signing keys for AWS federation."
weight: 6
subtitle: "Prove domain ownership, route enrollments, and isolate your organization's OIDC issuer"
params:
  docsGroup: admin
---

An organization is created from the email domain of its first enrollee. That domain is the organization's **primary domain** and cannot be changed. If your users have email addresses on more than one domain — an acquisition, a rebrand, a regional subsidiary — add those as **additional domains** so their enrollments attach to the same organization.

Manage them on the **Email Domains** page (`/admin/domains`).

---

## Why verification exists

Claiming a domain determines which organization a user's enrollment joins, and therefore which administrators can act on that user. Without proof of ownership, anyone could claim `competitor.com`, wait for one of that company's employees to enroll, and take administrative control of the account.

So an added domain does nothing until it is verified. Until then it takes no part in matching users at login — users on that domain continue to enroll exactly as if you had never added it.

---

## Adding and verifying a domain

1. **Add the domain** on `/admin/domains`. The server generates a random token and the entry enters the **Pending** state.

2. **Publish a DNS TXT record** at:

   ```
   _vouch-verification.<your-domain>
   ```

   with the token shown in the UI as its value. For `example.com`, that is `_vouch-verification.example.com`.

3. **Click Verify.** The server performs a DNS TXT lookup and marks the entry **Verified** if any record at that name matches the token. If the lookup fails or nothing matches, the entry stays pending and you can retry after fixing DNS.

Once verified, new enrollments from that domain join your organization. An organization may hold up to **10** additional domains on top of its primary domain.

> **Leave the TXT record published.** Verification is not a one-time check — the server re-verifies the record periodically, and removing it will eventually unverify the domain (see below).

---

## Domain states

| State | Meaning | Counts for login matching |
|---|---|---|
| **Pending** | Added, TXT record never yet observed | No |
| **Verified** | Ownership confirmed | Yes |
| **Unverified** | Was verified, then failed re-verification repeatedly | No |

Only **Verified** entries — plus the primary domain — make up the organization's owned domain set. That set controls three things:

- **Enrollment routing** — new enrollments from an owned domain join your organization.
- **SCIM provisioning** — an identity provider token can only create users whose email domain is in the owned set. See [SCIM Provisioning](/docs/scim/#domain-validation).
- **Issuer subdomain eligibility** — the labels you may claim as an issuer subdomain are derived from your verified domains (see below).

---

## Ongoing re-verification

A background task re-checks the DNS TXT record of every verified additional domain, at most once every **24 hours** per domain. After **3 consecutive failures**, the entry flips to **Unverified**:

- New logins stop attaching to your organization for that domain.
- **Users who already enrolled keep their organization membership.** They are not orphaned, deactivated, or removed — a DNS outage must not evict your existing users.
- An `org_domain_unverified` audit event is recorded.

A single successful check resets the failure counter to zero, so a brief DNS blip costs nothing.

Stale entries are garbage-collected automatically: a **Pending** entry that never verifies is deleted after **7 days**, and an **Unverified** entry after **14 days** (audited as `org_domain_expired`). Deletion only removes the claim; it never touches users.

---

## Removing a domain

Remove a domain from `/admin/domains`. This unclaims it — future enrollments from that domain no longer join your organization — and records `org_domain_removed`. Existing users keep their organization membership, exactly as with unverification.

---

## Issuer subdomains

By default, every organization's OIDC federation tokens are issued under the shared instance URL (for the US instance, `https://us.vouch.sh`). An organization can instead claim a **dedicated issuer subdomain** — for example, `acme-com` → `https://acme-com.us.vouch.sh` — with its own signing key set, so that a token issued for your organization does not verify against any other organization's JWKS, and vice versa.

Manage it on the **Issuer Subdomain** page (`/admin/subdomain`).

### Claiming a subdomain

The labels you may claim are derived from the registrable apex of your verified domains: verifying `acme.com` makes the label `acme-com` claimable. Claiming publishes the subdomain's own OIDC discovery document and JWKS — the page shows the issuer URL and the discovery URL that AWS IAM consumes.

### Effect on AWS federation

Issuer selection is automatic and fail-closed: once a subdomain is claimed, **all** of your organization's AWS OIDC tokens carry the subdomain as `iss` and `aud` — they never fall back to the shared issuer. Your [IAM OIDC provider](/docs/aws/#step-1--register-the-vouch-oidc-provider) must therefore point at the subdomain URL, not the shared instance URL. Claim the subdomain and update the IAM provider together, or role assumption will fail until they match.

### Signing key management

The subdomain page includes a key panel for the organization's signing keys:

| Action | Behavior |
|---|---|
| **Rotate** | Staged: a new key is published in the JWKS for **24 hours** before it starts signing, so relying parties (AWS JWKS caches, OIDC discovery caches) see the new key ID before the switch. The previous key remains published for verifying already-issued tokens. |
| **Revoke previous keys** | Retires the prior key set. Gated by a token-drain window so a key cannot be dropped while tokens it signed are still live. |
| **Emergency rotate** | Replaces the whole key set at once, for suspected key compromise. Outstanding tokens signed with the old keys stop verifying immediately. |

### Releasing a subdomain

Releasing the subdomain reverts your organization's tokens to the shared instance issuer. Any IAM OIDC provider still pointing at the subdomain URL will stop matching newly issued tokens — update it back to the shared instance URL when you release.

---

## Audit events

Every domain and subdomain action is recorded:

| Event | Trigger |
|---|---|
| `org_domain_added` | Domain added, entering pending |
| `org_domain_verified` | TXT record matched, entry verified |
| `org_domain_removed` | Administrator removed the domain |
| `org_domain_unverified` | Re-verification failed 3 times in a row |
| `org_domain_expired` | Garbage-collected as a stale pending or unverified entry |
| `org_subdomain_claimed` / `org_subdomain_released` | Issuer subdomain lifecycle |
| `org_issuer_key_rotated` / `org_issuer_key_revoked` / `org_issuer_key_emergency_rotation` | Signing key lifecycle (one event per algorithm) |

These are administrative events and are retained indefinitely — see [Audit Log Export](/docs/audit-export/).

---

## Troubleshooting

**Verify fails but the record looks correct.** Check propagation with `dig +short TXT _vouch-verification.example.com`. The value must match the token exactly. If your DNS provider appends the zone name automatically, entering `_vouch-verification.example.com` in the `example.com` zone yields `_vouch-verification.example.com.example.com`.

**A domain unverified itself.** The TXT record was unreachable on 3 consecutive daily checks. Republish it and click Verify again. Existing users on that domain were unaffected.

**A domain disappeared from the list.** It was pending for more than 7 days, or unverified for more than 14, and was garbage-collected. Re-add it to get a fresh token.

**Role assumption fails after claiming a subdomain.** Your tokens now carry the subdomain issuer, but the IAM OIDC provider still points at the shared instance URL. Update the provider (and trust policy conditions) to the subdomain URL shown on `/admin/subdomain`.
