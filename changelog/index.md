# Changelog

> Major features in recent Vouch releases: PROXY protocol support, connection limits, RFC 9421 ECDSA signature encoding, mTLS certificate chain validation, SCIM PUT support, device-flow client authentication, enforcement of registered grant and response types, a DNS-over-HTTPS resolver fix, mandatory attestation for hardware key registration, SLSA Build Level 3 provenance, a Cedar-based policy engine, and an audit events API with OCSF export.

Source: https://vouch.sh/changelog/
Last updated: 2026-10-02

---
Highlights from recent Vouch releases. For the complete list of changes in every
release, including bug fixes, dependency updates, and internal refactoring,
see the [GitHub releases page](https://github.com/vouch-sh/vouch/releases).

## [v2026.10.1](https://github.com/vouch-sh/vouch/releases/tag/v2026.10.1) - October 2, 2026

- **HTTP message signatures use the RFC 9421 ECDSA encoding**: **Breaking:**
  [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) section 3.3.4 defines an
  `ecdsa-p256-sha256` signature as a 64-octet `r||s` value. The server and CLI
  signed and verified DER, so a client built on a conforming library received
  a 401 on every signed `/v1/*` request. Both now use `r||s`. A v2026.10.1
  server rejects requests from an earlier CLI. Upgrade the server and the CLI
  together.
- **The server accepts the PROXY protocol**: a TCP proxy that relays TLS
  without terminating it (Envoy or Istio passthrough, nginx `stream`, HAProxy
  `mode tcp`) made every request appear to come from the proxy, so all users
  shared one rate-limit bucket and audit events recorded the proxy&#39;s address.
  With `VOUCH_PROXY_PROTOCOL=true`, the HTTPS and mTLS listeners require a
  [PROXY protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)
  v2 header from a peer in `VOUCH_TRUSTED_PROXIES` and use the header&#39;s source
  address for rate limiting and audit. The server refuses to start with the
  switch on and no trusted proxies configured. Port 80 serves `/health/ready`
  for probes that cannot send the header.
- **Every listener limits connections and their duration**: the server closes
  a connection that does not finish its TLS handshake within 5 seconds or holds
  no request for 10 seconds. Before this release, one client that never sent a
  ClientHello on the mTLS port blocked every new mTLS connection.
  `VOUCH_MAX_CONNECTIONS` (default 10,000) caps open connections in total, and
  `VOUCH_MAX_CONNECTIONS_PER_IP` (default 64) caps them per client. The request
  timeout drops from 30 seconds to 10.
- **Rate limits count an IPv6 /64 as one client**: the request rate limiters
  keyed each IPv6 address separately, so one host could rotate addresses
  within its /64 for a new allowance per address. The rate limiters and
  connection caps count each /64 as one client and prune idle clients every
  minute. The mTLS listener ignores `X-Forwarded-For`, and
  `VOUCH_TRUSTED_PROXIES` applies to the HTTPS port only.
- **Dashboard applications can use `private_key_jwt` without FAPI**: a
  Standard-profile web or service application can authenticate with a private
  key instead of a client secret. The server issues no secret and does not
  sender-constrain its tokens, so the application can forward them to
  bearer-only APIs. `POST /api/v1/applications` takes the same choice in
  `token_endpoint_auth_method`.
- **Tokens are accepted only where they are meant to be used**: the session
  cookie accepts only a browser session token issued to the deployment itself.
  An access token issued to an OAuth client or narrowed to a resource reads as
  signed out. Token exchange keeps a narrowed subject token narrowed and
  rejects an actor token from a different organization than the subject.
  `/v1/*` refuses an unbound token under the `DPoP` scheme, and every endpoint
  refuses a request with more than one `DPoP` header
  ([RFC 9449](https://www.rfc-editor.org/rfc/rfc9449) section 4.3).
- **Deactivation and key deletion apply on every endpoint**: `/v1/auth/status`,
  `/v1/keys`, and the applications API accepted a deactivated user&#39;s live
  token. The server now refuses a deactivated account wherever it reads a
  token. A session whose hardware key was deleted is refused on `/v1/*`,
  introspection, and token exchange, as it already was at userinfo. Deleting
  or deactivating a client&#39;s owner revokes the client&#39;s
  [RFC 7592](https://www.rfc-editor.org/rfc/rfc7592) registration access token.
- **The CLI sends a credential only to the server it belongs to**: the CLI
  sent the stored session token to whatever server `--server` or
  `VOUCH_SERVER` named. It now refuses unless the stored session belongs to
  that server. `vouch setup anthropic` and `vouch setup openai` refuse a
  `--token-endpoint` that is not HTTPS, except plain HTTP to loopback.
- **The agent ties cached credentials to a live session**: the agent drops
  every cached credential and SSH certificate when a new session replaces the
  old one, including the same user signing in to another server. It stops
  serving them when the session expires, and it serves an SSH certificate only
  when the certificate names the session&#39;s user and server. The agent no
  longer restores its own session at startup and makes no outbound HTTP
  requests; the CLI restores the session with a DPoP-bound request.
- **Policies deny when a rule fails to evaluate**: Cedar skips a policy whose
  condition errors at runtime, so a custom `forbid` that overflowed on
  client-supplied device posture allowed the request. An allow that carries an
  evaluation error is now a deny attributed to the failing policy. Linking a
  GitHub App installation no longer counts as a GitHub credential in policy
  history, and the server queries 24-hour history only for decisions a
  temporal rule covers.
- **The SSRF guard follows the IANA special-purpose registries**: the guard on
  client-controlled `jwks_uri` and `request_uri` fetches allowed IPv6 addresses
  that embed a private IPv4 address, such as the NAT64 address
  `64:ff9b::a00:1` (10.0.0.1). The guard classifies NAT64, 6to4, Teredo, and
  IPv4-mapped addresses by the IPv4 address they carry, and refuses every block
  the IANA registries mark as not globally reachable. Client registration
  refuses RSA keys outside 2048 to 8192 bits.
- **Protocol fixes**: the server rejects a signed request whose body was
  stripped in transit, because it checks `Content-Digest` whenever the signer
  sent one. A client that presents a Basic header and a body `client_secret`
  together is rejected ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)
  section 2.3). A pushed authorization request with a Request Object reads its
  parameters only from that object
  ([RFC 9101](https://www.rfc-editor.org/rfc/rfc9101) section 6.3). A SAML
  assertion must name the service provider in every `AudienceRestriction`.
- **AWS CLI fixes**: `vouch setup aws --discover` dropped an assignment whose
  profile name collided with another. It now writes a profile for every
  assignment, adding the account ID and then a hash to the name when needed.
  `vouch credential rds` reads the region only from a real RDS endpoint
  suffix. The RDS and EKS credential caches key on region, and the Redshift
  cache also keys on database and duration.
- **Audit and configuration fixes**: every audit row written for a request
  records the client IP and User-Agent. Rows for authorization-code token
  issuance, RP-initiated logout, and GitHub App linking omitted them. An
  empty value (`VAR=&#34;&#34;` or `&#34;&#34;` in the S3 configuration document) loads as
  unset for every optional setting, so an empty S3 value no longer overrides
  the environment.

## [v2026.9.5](https://github.com/vouch-sh/vouch/releases/tag/v2026.9.5) - September 24, 2026

- **mTLS client authentication validates the certificate**: a
  `tls_client_auth` client authenticated on a subject DN or SAN match alone, so
  a self-signed certificate carrying a registered DN authenticated as that
  client. The server now validates the certificate chain
  ([RFC 8705](https://www.rfc-editor.org/rfc/rfc8705) section 2.1). A
  `self_signed_tls_client_auth` client matches only the leaf of its registered
  `x5c` chain.
- **SCIM supports PUT and follows RFC 7644 PATCH and filter rules**: `PUT` on
  `/Users/{id}` and `/Groups/{id}` replaces the resource. One
  [RFC 7644](https://www.rfc-editor.org/rfc/rfc7644) parser reads every
  filter. An unrecognized filter no longer returns every user or group, a
  compound filter evaluates every clause, and escaped and URN-qualified values
  match. **Access-affecting:** a PATCH `remove` on `members` with no filter and
  no value empties the group, a group `PUT` without `members` removes every
  member, and a user `PUT` without `active` sets the user active.
- **Client credentials cannot be reused or downgraded**: a `private_key_jwt`
  assertion&#39;s `jti` is spent when it authenticates, so a request rejected
  later cannot leave the assertion reusable at another endpoint. The FIDO2
  assertion grant is bound to the client that started the ceremony. A client
  registered for `private_key_jwt` can no longer mint a shared secret.
- **FIPS-approved key exchange on Linux**: the server, CLI, and agent offer
  only FIPS-approved TLS key-exchange groups on Linux. Bare X25519 is no longer
  offered; `X25519MLKEM768` remains the preferred group.
- **The agent pairs each session with its own server**: the agent refused a
  plain-HTTP server URL but still stored the new token beside the previous
  session&#39;s URL, so the CLI could send a development token to production. The
  agent now stores a session only with its own URL. `VOUCH_ALLOW_INSECURE`
  set to `false`, `0`, or an empty string no longer enables plain HTTP.
- **Deactivation covers applications**: a deactivated user cannot register a
  client, and an organization application whose creator is deactivated
  transfers to an active organization admin.
- **GitHub installation linking checks the caller**: linking an installation
  requires an organization admin whose GitHub account has access to it.
  Replaying the install callback no longer creates a duplicate installation,
  and installation webhooks update every matching record.
- **Writes re-check their preconditions**: a GitHub token refresh cannot
  restore a revoked token or revert a concurrent re-link, and
  [RFC 7592](https://www.rfc-editor.org/rfc/rfc7592) updates and deletes
  re-check their premise inside the write. An RFC 7592 `GET` now returns the
  required `registration_access_token`.
- **Audit log fixes**: revocations, logouts of expired sessions, and custom
  policy toggles record the state the server committed. The audit API ignores
  blank filters.

## [v2026.9.4](https://github.com/vouch-sh/vouch/releases/tag/v2026.9.4) - September 15, 2026

- **The device flow authenticates its client at both endpoints**: `/oauth/device`
  and the device-code grant run the same client authentication as every other
  grant ([RFC 8628](https://www.rfc-editor.org/rfc/rfc8628) sections 3.1 and
  3.4). A request without credentials is rejected, and redemption is bound to
  the client the device code was issued to. `/oauth/revoke` and
  `/oauth/introspect` now verify the certificate of a client registered for
  [mTLS](https://www.rfc-editor.org/rfc/rfc8705) authentication, which they had
  accepted on its `client_id` alone. Enrolling requires the v2026.9.4 CLI or
  later.
- **Workload Identity Federation works again for CLI-registered FAPI clients**:
  `vouch credential openai` and `vouch credential anthropic` send the
  [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693) token-exchange grant, but
  the CLI&#39;s Financial-grade API
  ([FAPI](https://openid.net/specs/fapi-2_0-security-profile.html)) client
  registration left that grant out. The server rejected those requests with
  `unauthorized_client`. New registrations declare every grant the client uses.
  Repair an enrolled client by running `vouch login` or `vouch enroll`.
- **Registered grant types and response types are enforced**: the
  authorization-code and device-code grants check the client&#39;s registered
  `grant_types` before the single-use code is consumed
  ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749) section 5.2), so a client
  registered for one grant cannot redeem another. The authorization endpoint
  checks registered `response_types`, including on the
  [JAR](https://www.rfc-editor.org/rfc/rfc9101) paths. Rejections render through
  the `response_mode` the request negotiated instead of falling back to a query
  redirect.
- **Applications created in the dashboard resolve grants from their type**: a
  self-service application stores no grant list, which resolved to
  [RFC 7591](https://www.rfc-editor.org/rfc/rfc7591) section 2&#39;s
  `authorization_code` default. A Native application could not use the device
  flow, and a Service application could not use client credentials. Grants now
  follow the application type. An
  [RFC 7592](https://www.rfc-editor.org/rfc/rfc7592) PUT that restates the
  `response_types` the server itself issued is also accepted again.
- **A client is held to the authentication method it registered**: a native
  client issued a per-instance secret
  ([RFC 8252](https://www.rfc-editor.org/rfc/rfc8252) section 8.4) had its
  method rewritten to `none` on read, so it could authenticate without the
  secret. Client type now comes from the registered method. The secret-rotation
  endpoints use the same rule, so the owner of a native or SPA client that holds
  a secret can rotate a compromised one.
- **Subject DN matching accepts multi-valued RDNs**: `tls_client_auth_subject_dn`
  comparison treats `&#43;` as the multi-valued RDN separator
  ([RFC 4514](https://www.rfc-editor.org/rfc/rfc4514) section 2.3) and tolerates
  spaces around `=`. A subject that is one multi-valued RDN parses instead of
  falling back to exact string comparison. Matching is not loosened: a DN of two
  or more RDNs still authenticates only in the order
  [RFC 8705](https://www.rfc-editor.org/rfc/rfc8705) section 2.1.2 requires, and
  a value that matches only in reverse logs a warning naming `-nameopt rfc2253`.
- **Every path that removes an admin enforces the last-admin floor**: &#34;at least
  one active admin per organization&#34; covered demote and deactivate. It did not
  cover the admin UI&#39;s remove-member action, SCIM `DELETE /Users/{id}`, or a
  SCIM update setting `active=false`. One `UsersWrite` token could remove every
  admin in sequence, and SCIM `DELETE` left the organization no way back in. All
  three paths now take the same guard in one transaction, so concurrent removals
  collide instead of each seeing the other as the surviving admin.
- **Hardware key counters compare across the full `u32` range**: WebAuthn
  `signCount` is a `u32` stored in a signed column, and the monotonic maximum
  compared it as a signed value. A counter stopped advancing at 2^31-1 for the
  life of the credential, and a key past that point lost its baseline to any
  lower count, weakening clone detection. The read and the comparison now run in
  `u32` space.
- **Expiry checks use the instant the request arrived**: both ID token
  generators stamped `iat` and `exp` from a later clock read, so an ID token
  could outlive the access token issued with it. Nine database helpers that
  compare a stored `expires_at` each stamped their own clock, which could reject
  a record that was live on arrival. All of them take the arrival instant.
- **AWS messages name the config file they read**: the CLI derives the AWS
  config directory from the resolved path instead of assuming `~/.aws`. The four
  messages that report a profile lookup name that path, so setting
  `AWS_CONFIG_FILE` no longer sends an operator to a file that does not hold the
  profile.

## [v2026.9.3](https://github.com/vouch-sh/vouch/releases/tag/v2026.9.3) - September 12, 2026

- **DNS-over-HTTPS works again**: v2026.9.2 shipped `hickory-resolver` 0.26.2,
  which carried a DNSSEC validation regression. The CLI and agent always
  validate DNSSEC when
  [DNS-over-HTTPS](https://www.rfc-editor.org/rfc/rfc8484) is enabled, and a
  signed response that fails to validate fails the lookup, so credential
  flows could fail to resolve the Vouch server. This release pins 0.26.3,
  where the upstream fix landed. Only v2026.9.2 is affected, and only if you
  turned DNS-over-HTTPS on with `VOUCH_DOH` or the `network.dns_over_https`
  config field. The server resolves through system DNS and is unaffected.
  Upgrade from v2026.9.2.
- **Foreign tool config files are written where those tools read them**: the
  CLI resolves pip, uv, AWS, and Docker configuration paths by each tool&#39;s own
  documented search order instead of building paths from `$HOME`. pip&#39;s
  existence-gated macOS chain and its `%APPDATA%\pip\pip.ini` location on
  Windows are honored, `PIP_CONFIG_FILE` and `UV_CONFIG_FILE` are treated as
  filenames rather than directories, and `AWS_CONFIG_FILE` and `DOCKER_CONFIG`
  are respected. Helper binaries install to `$XDG_BIN_HOME` when it is set.
- **SCIM filters and deactivation fixes**: an attribute name is matched only at
  the start of a filter expression, so a filter naming one attribute no longer
  matches another that contains it. Member filter operators match regardless of
  case. A PATCH that toggles `active` revokes credentials based on the net
  result of the whole request rather than on each operation, so a request that
  ends with the user still active no longer revokes their credentials.
- **Grants and revocation are scoped to what the request named**: the token
  endpoint enforces the client&#39;s registered `grant_types` on the token-exchange
  and FIDO2 assertion grants, revoking a machine-to-machine token revokes that
  token instead of every session the client holds, and the revocation handler
  records the proof `jti` before it revokes. The admin UI path rejects
  sender-constrained tokens, and
  [RFC 9728](https://www.rfc-editor.org/rfc/rfc9728) protected-resource metadata
  no longer advertises the SCIM endpoints.
- **Client registration and mTLS matching fixes**: a registration response
  echoes the [RFC 8705](https://www.rfc-editor.org/rfc/rfc8705) mTLS metadata
  the application registered, subject DN comparison ignores spaces after commas
  so a certificate matches however its issuer formatted the DN, and a JWK
  marked `use: &#34;enc&#34;` is rejected for
  [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) signature verification.
  Resuming a pending authorization compares `max_age` at full precision.
- **SAML canonicalization and assertion validation**: exclusive
  canonicalization emits an empty default-namespace declaration on prefixed
  elements when `#default` appears in the `InclusiveNamespaces` prefix list, and
  the server requires `SubjectConfirmationData.NotOnOrAfter` on every bearer
  confirmation instead of accepting a confirmation with no expiry.
- **Hardware key counters start at the registered value**: a newly registered
  key stores the `signCount` from its registration ceremony, so the first
  assertion is compared against the right starting counter. An enrollment
  callback that cannot read the user&#39;s authenticator list fails closed.
- **Concurrent admin changes cannot break organization invariants**: org-admin
  changes serialize on the organization row, toggling a preconfigured policy
  cannot overwrite a concurrent change, an org-scoped application create or
  update fails if its owner is deleted mid-request, and removing an additional
  domain invalidates the affected sessions in the cache. Renaming a hardware
  key and deleting an application both record audit events.
- **Token exchange honors the actor&#39;s logout**: the
  `logout_invalidates_exchange` policy check evaluates against the user named by
  the [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693) actor token, so an
  actor who has logged out cannot exchange a token on another user&#39;s behalf.

## [v2026.9.2](https://github.com/vouch-sh/vouch/releases/tag/v2026.9.2) - September 11, 2026

- **Token exchange honors the resource allowlist and the subject token&#39;s
  lifetime**: [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693) token
  exchange checks the requested `resource` values against the client&#39;s
  allowlist, including on the path that resumes a pending authorization. An
  exchanged access token never outlives the subject token it came from, and a
  subject token with no remaining lifetime yields no token at all.
- **DPoP proofs cannot be replayed inside the clock-skew window**: the server
  keeps each proof&#39;s `jti` for as long as the proof remains valid under the
  allowed clock skew, so a replayed proof is rejected for its entire life
  rather than accepted in the window between the retention expiring and the
  proof going stale. Assertion replay records now expire with the assertion
  they cover. Protected-resource metadata reports
  `dpop_bound_access_tokens_required` as `false`, because an
  [mTLS](https://www.rfc-editor.org/rfc/rfc8705) certificate satisfies the
  sender-constraint requirement as well as a
  [DPoP](https://www.rfc-editor.org/rfc/rfc9449) proof does.
- **Sessions prove hardware possession before they authorize anything**: a
  session that has not been hardware-verified must complete a key assertion
  before it authorizes an application or makes an admin change. Signing in
  through the upstream identity provider establishes a browser session
  directly. The `auth_time` claim on device-flow and FIDO2 assertion grants
  records when the hardware key ceremony ran, and every time comparison in a
  request uses a single arrival timestamp.
- **Deleting a key, user, or application revokes credentials first**: the
  server revokes issued credentials before it deletes the hardware key that
  approved them, revokes an application&#39;s client secrets before it sweeps the
  sessions that used them, and revokes user-issued access tokens along with
  the rest. Retries re-read the authenticator list and evict the session
  cache, so a retried deletion cannot leave a credential behind. Deleting a
  key preserves the device authorizations it already consumed, and a
  deactivated user cannot delete keys or link a GitHub account.
- **Audit events are written before the next step can fail**: every committed
  change records its audit event before the server attempts the next fallible
  operation, so an enrollment or credential release that fails midway still
  leaves a record. Audit writes go through one API and are awaited.
- **Email domains are validated where accounts are created**: enrollment
  rejects addresses with an empty domain or whitespace in the domain, SCIM
  rejects a malformed local part in `userName`, and domains asserted by an
  identity provider parse into a validated domain type. An organization can no
  longer claim a subdomain that collapses onto another organization&#39;s apex
  domain.
- **Client authentication fixes**: an
  [RFC 7592](https://www.rfc-editor.org/rfc/rfc7592) update preserves the
  application&#39;s mTLS client identity, device-flow token redemption enforces
  the mTLS client authentication the application is configured for, IPv6
  addresses in a certificate SAN are normalized before comparison, and
  [FAPI](https://openid.net/specs/fapi-2_0-security-profile.html) clients
  cannot mint client secrets. Failed authentication on the client-credentials
  and token-exchange grants returns a `WWW-Authenticate` header.
- **Unusable JWKS keys no longer fail the request**: when a client&#39;s JWKS
  contains a key the server cannot build or a key of the wrong family, the
  server skips it and keeps searching the remaining keys for one that
  verifies the assertion or ID token.
- **Hardware key registration binds to the user who started it**: completing a
  registration requires the same user the registration state was issued to,
  and the server rejects an attestation statement whose `x5c` certificate
  chain contains anything other than byte strings.
- **Device flow survives concurrent polling**: an authorization that collides
  with the CLI&#39;s poll retries instead of failing, and each retry re-stamps its
  timestamp.
- **Protocol and CLI fixes**: SAML exclusive canonicalization emits an empty
  default-namespace declaration only for unprefixed elements, a failed
  `request_uri` fetch echoes the request object&#39;s `state` back to the
  application, and SSH certificate serials are canonicalized before the
  revocation lookup, so a serial submitted in any form matches its
  certificate.
- **Faster OAuth grants**: the user record carries its organization&#39;s primary
  domain, removing a database lookup from every grant, and the session cache
  hands out a shared session instead of a copy.

## [v2026.9.1](https://github.com/vouch-sh/vouch/releases/tag/v2026.9.1) - September 1, 2026

- **Hardware key registration requires attestation**: **Breaking:**
  registering a hardware key now requires a verified attestation chain (the
  manufacturer&#39;s proof that the key is a genuine device). Sign-ins that used
  only the upstream identity provider no longer count as hardware-verified,
  and the re-authentication check before key deletion rejects future
  timestamps.
- **AWS setup verifies each role**: `vouch setup aws` tests that each
  discovered or existing profile can assume its role. When a profile fails,
  the CLI prints the IAM policy change that fixes it and the role&#39;s Identity
  Center status.
- **Authorization code replay revokes only the affected tokens**: when an
  authorization code is used twice, Vouch revokes the tokens issued from
  that code. The user&#39;s other sessions stay active.
- **SAML assertions must reference their request**: the server rejects
  assertions without `InResponseTo` and reads values only from the signed
  portion of the response.
- **Registration rejects misconfigured applications**: the server rejects
  an OAuth client with an invalid configuration (a signing key incompatible
  with its algorithm, a private key in a JWKS, an invalid redirect URI) at
  create or update time, before the configuration causes sign-in failures.
  The same rules apply on every registration path.
- **Client management API fixes**: failed authentication on the
  [RFC 7592](https://www.rfc-editor.org/rfc/rfc7592) client-management
  endpoints returns 401 and revokes the presented token,
  an update replaces the application&#39;s metadata as a whole, and a stale
  request carrying an old registration token cannot undo a rotation.
- **The agent shuts down on SIGTERM and Ctrl&#43;C**: the agent finishes
  in-flight requests before exiting, and it installs the signal handlers
  before its listeners bind, so it does not miss an early signal.
- **Deactivation revokes credentials first**: the server revokes a user&#39;s
  credentials before it saves the deactivation, so a failure partway through
  cannot leave a deactivated user with working credentials. Deleting a
  hardware key voids the pending device sign-in approvals it made.
- **SSH CA keys must be Ed25519**: the server rejects other key types when
  it loads the key, not later during signing.
- **OAuth and request-signing error fixes**: the authorize endpoint returns
  errors in the response mode the application requested, the server reports
  its own faults as server errors instead of client errors, session-age
  checks compare at full precision instead of whole seconds, and
  [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) request-signature
  validation implements the remaining canonicalization algorithms and
  rejects duplicate covered components.

## [v2026.8.4](https://github.com/vouch-sh/vouch/releases/tag/v2026.8.4) - August 16, 2026

- **SLSA Build Level 3 provenance**: release artifacts are now built and
  attested inside a
  [dedicated reusable workflow](https://docs.github.com/actions/security-guides/using-artifact-attestations-and-reusable-workflows-to-achieve-slsa-v1-build-level-3),
  so the attestation signing identity is out of reach of the build steps
  themselves, the bar [SLSA](https://slsa.dev/) sets for Build Level 3. Every archive,
  container image, Helm chart, and SBOM can be verified with the builder
  pinned; the attestation records the reusable workflow as the signer,
  distinct from the release workflow that called it:

  ```console
  $ gh attestation verify vouch-v2026.8.4-aarch64-apple-darwin.tar.gz \
      --owner vouch-sh \
      --signer-workflow vouch-sh/vouch/.github/workflows/reusable-build.yml
  Loaded 1 attestation from GitHub API
  ✓ Verification succeeded!

  The following 1 attestation matched the policy criteria

  - Attestation #1
    - Build repo:..... vouch-sh/vouch
    - Build workflow:. .github/workflows/release.yml@refs/tags/v2026.8.4
    - Signer repo:.... vouch-sh/vouch
    - Signer workflow: .github/workflows/reusable-build.yml@refs/tags/v2026.8.4
  ```

  The GitHub Actions OIDC identity behind release signing also moved to
  [immutable subject claims](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/),
  so deleting and re-registering a repository name can no longer mint tokens
  that match the release trust policies.
- **AWS role discovery from Identity Center entitlements**: `vouch setup aws`
  discovery now runs a second pass over
  [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
  [account access manager](https://aws.amazon.com/about-aws/whats-new/2026/08/aws-iam-aam/)
  entitlements, so roles assigned to you or your groups through Identity
  Center are found without hand-configuration. The management role needs new
  read-only IAM permissions for the pass, which runs in the commercial
  partition only. **Breaking:** AWS authorization failures now exit with
  code 5 instead of 4 (signature and clock-skew errors keep exit code 4).
- **Sender-constrained tokens enforced on every grant**: issuing any token
  now requires a sender-constraint witness
  ([DPoP](https://www.rfc-editor.org/rfc/rfc9449) proof or
  [mTLS](https://www.rfc-editor.org/rfc/rfc8705) certificate),
  DPoP-bound device-flow and client-credentials tokens are correctly labeled
  `token_type: DPoP`, and resource endpoints and `/oauth/register` answer
  stale-nonce requests with a `DPoP-Nonce` challenge instead of an opaque
  failure. Device-flow infrastructure failures now return the retryable
  `server_error` instead of `invalid_grant`.
- **SCIM PATCH overhaul**:
  [PATCH](https://www.rfc-editor.org/rfc/rfc7644#section-3.5.2) operations
  apply through a shared attribute
  table covering every mutable attribute, group-membership changes apply
  atomically, Entra-style remove-by-value is supported, and concurrent adds
  of the same member resolve to a single membership. Status codes are
  corrected across the board: infrastructure faults during group creation are
  500s, a NUL byte in a member ID is a 400, and deleting an already-removed
  user is a 404.
- **SSH agent socket peer verification**: the agent now verifies the
  connecting process&#39;s UID on its SSH agent socket and validates its runtime
  directory before listening, so another local user cannot use the socket.
- **Account lifecycle fixes**: when an org-scoped application&#39;s creator is
  deleted, the application transfers to an active organization admin instead
  of being orphaned. Deactivated users are blocked from CLI hardware-key
  registration, and sessions arriving exactly at a `max_age` boundary are
  accepted.
- **SAML interop fix**: assertions carrying multiple SubjectConfirmation
  elements now sign in when any bearer confirmation validates, instead of
  being rejected outright.
- **security.txt**: the server publishes an
  [RFC 9116](https://www.rfc-editor.org/rfc/rfc9116) `security.txt`, giving
  security researchers a discoverable reporting channel.

## [v2026.8.3](https://github.com/vouch-sh/vouch/releases/tag/v2026.8.3) - August 10, 2026

- **New policy engine with history-aware policies**: device-posture policies
  now run on [Dogwood](https://github.com/dogwood-policy/dogwood), a
  [Cedar](https://www.cedarpolicy.com/)-based engine with a temporal
  sublanguage, replacing CEL. Policies can
  reason about recent activity: five new built-ins cover token issuance rate
  limiting, failed-login bursts, and step-up, IP-consistency, and
  logout-invalidation checks on token exchange. Policies are type-checked
  against a schema at save time, so a typo&#39;d field is an error instead of a
  silent miss, and every denial emits a new `policy_denied` audit event.
  Custom policies written in CEL fail closed until re-authored; the policies
  page flags them and the docs include a rewriting guide.
- **Guided policy rule builder**: the admin policies page replaces its
  free-text editor with a guided builder: pick a decision point, then compose
  conditions from dropdowns of known device fields, operators, and activity
  windows, with a live preview and continuous validation. Raw policy text
  remains as an escape hatch. Two new built-in policies: MDM enrollment
  required, and a token-exchange rate limit.
- **Hardware key possession enforced for enrollment**: `vouch enroll` for a
  user with a registered key previously released a credential-capable token
  without a WebAuthn ceremony. Enrollment now requires a key assertion,
  tokens record whether their approval verified hardware, and
  credential endpoints refuse tokens without that proof.
- **NUL bytes rejected in client-supplied identifiers**: identifiers
  containing a NUL byte (e.g. a SCIM `externalId`) caused backend-dependent
  500s on PostgreSQL and Aurora DSQL. Every backend now answers with a 400
  naming the offending field.
- **FIDO2 token issuance audited**: the FIDO2 assertion grant now records an
  `oauth_token_issued` audit event with the user, client IP, and grant type,
  matching the other OAuth grants.
- **SSH certificate serials printed exactly**: `vouch credential ssh` rounded
  serials above 2^53 in its output, and a rounded serial submitted for
  revocation silently matches no certificate. Serials now print
  digit-for-digit.

## [v2026.8.2](https://github.com/vouch-sh/vouch/releases/tag/v2026.8.2) - August 9, 2026

- **SSO sign-in fixed on PostgreSQL and Aurora DSQL**: the lookup that matches
  an upstream identity to a Vouch account built its database index value in a
  form those backends reject, so OIDC sign-ins (and SAML sign-ins with a
  persistent NameID) failed there. The index value is now a SHA-256 hash,
  which also keeps the raw upstream subject out of the index table. No
  migration is required.

## [v2026.8.1](https://github.com/vouch-sh/vouch/releases/tag/v2026.8.1) - August 8, 2026

- **Organization audit events API with OCSF export**: a new org-scoped API
  lets organizations query their own audit events and export them in
  [OCSF](https://schema.ocsf.io/) format for SIEM ingestion. Browser sign-in
  events now record the user&#39;s email, and the admin audit page loads faster
  with batched user lookups.
- **Deactivation enforced everywhere**: deactivated users are now blocked
  consistently across SSO sign-in, device-flow approval, hardware key
  registration, and OAuth token introspection, with every session path
  running through a single active-user check.
- **SCIM provisioning hardening**: user provisioning now rejects addresses
  outside the organization&#39;s verified domains, checked inside the creation
  transaction to close a race. Concurrent create requests for the same user
  resolve to a single account, and filter matching follows
  [RFC 7644](https://www.rfc-editor.org/rfc/rfc7644) case rules.
- **No more duplicate users from email casing**: email addresses are
  canonicalized to lowercase across sign-in, SCIM, and audit correlation, so
  the same person arriving with different casing no longer creates duplicate
  accounts. JIT account linking is now keyed on the upstream issuer and
  subject pair.
- **AWS partition mismatches caught early**: the CLI validates that the
  configured region&#39;s partition matches the role ARN before calling STS (in
  credential minting, SSM setup, and CodeCommit setup), failing with a clear
  error instead of an opaque STS one.
- **Git credential helper fixes**: remote URLs with an explicit `:443` port
  now match the CodeCommit and GitHub helpers, the original host is preserved
  for git&#39;s credential cache, and `vouch exec` and the CodeCommit helper
  propagate the child process&#39;s exit code on Windows.

## [v2026.7.4](https://github.com/vouch-sh/vouch/releases/tag/v2026.7.4) - July 29, 2026

- **Explicit AWS profile selection**: with multiple Vouch-managed AWS
  profiles, the CLI no longer silently uses whichever appears first in
  `~/.aws/config`. It resolves `--profile`, then `AWS_PROFILE`, then the sole
  managed profile, and otherwise errors with a list of every profile and its
  role ARN. `--profile` now means an AWS profile on every command; the
  CodeArtifact bundle flag is renamed `--domain-profile`.
- **CodeCommit git helper fixed**: `vouch setup codecommit --configure`
  previously wrote a credential helper value git could not execute. The helper
  (and the GitHub one) is now built shell-safe, `codecommit::&lt;region&gt;://`
  URLs parse correctly, and the region resolves from the same profile that
  mints the credentials.
- **In-process server bootstrap**: on EC2 the server now reads instance facts
  from IMDS and fetches its configuration from SSM Parameter Store itself at
  startup, replacing the AMI&#39;s external bootstrap script and bundled AWS CLI.
  Bootstrapped values apply below CLI flags and environment variables.
- **CLI fully translation-ready**: the remaining English-only CLI strings
  (command help text and error messages) moved into the Fluent catalogs.
- **CodeArtifact role ARN validation**: an unparseable role ARN in
  CodeArtifact setup now fails with an error naming the ARN, instead of
  silently writing commercial-partition registry URLs into pip, cargo, and
  npm configuration.

## [v2026.7.3](https://github.com/vouch-sh/vouch/releases/tag/v2026.7.3) - July 26, 2026

- **Credential and key lifecycle audit events**: AWS credential issuance, SSH
  certificate issuance, [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693)
  token exchange, hardware key registration and
  removal, device-flow approvals, and application secret changes now emit
  queryable audit events, each with its own retention class.
- **Website sign-ins recorded correctly**: signing in through the website now
  emits a `login_success` event carrying the client IP, instead of being
  misreported as a CLI device-flow approval.
- **Vouch attributable in CloudTrail**: the server tags its AWS SDK requests
  with `lib/vouch-server/&lt;version&gt;` in the user agent, so the S3 and KMS calls
  it makes are identifiable in CloudTrail.

## [v2026.7.2](https://github.com/vouch-sh/vouch/releases/tag/v2026.7.2) - July 14, 2026

- **OIDC role pinning for AWS**: each token now carries the target role ARN in
  the `https://aws.amazon.com/roles` claim, and AWS STS enforces it: a leaked
  token cannot be exchanged for any role other than the one it was minted for.
  Trust policies can make pinning mandatory with the
  [`sts:RoleAuthorizedByIdp`](https://awsteele.com/blog/2026/07/13/oidc-tokens-can-restrict-which-aws-roles-they-assume.html)
  condition key.
- **Per-organization OIDC issuers**: each organization gets its own subdomain
  issuer for AWS federation, with per-org signing keys and key rotation.
- **Post-quantum TLS**: the CLI and agent now prefer hybrid post-quantum key
  exchange when connecting to the Vouch server.
- **IAM Identity Center context in AssumeRole**: the CLI forwards IdC identity
  context into role sessions for
  [trusted identity propagation](https://docs.aws.amazon.com/singlesignon/latest/userguide/trustedidentitypropagation.html).
- **Access-token audience enforcement**: the server now enforces the token
  audience at every resource endpoint.

## [v2026.7.1](https://github.com/vouch-sh/vouch/releases/tag/v2026.7.1) - July 3, 2026

- **AWS setup redesigned around organization anchors**: `vouch credential aws`
  and `vouch setup aws` were rebuilt around a single org-level anchor, so
  multi-account access is configured once and roles resolve consistently
  everywhere.
- **Interactive AWS setup wizard**: `vouch setup aws` now walks you through
  account and role configuration step by step instead of requiring flags.
- **Device code pre-fill**: `vouch login` pre-fills the device code in the
  browser, removing the copy-and-type step from the login flow.

## [v2026.6.3](https://github.com/vouch-sh/vouch/releases/tag/v2026.6.3) - June 24, 2026

- **Internationalization**: the server UI, CLI, and agent are now fully
  translation-ready, with all user-facing strings extracted into per-binary
  [Fluent](https://projectfluent.org/) catalogs and locale negotiated
  automatically. Timestamps in the applications and admin UI render in the
  viewer&#39;s locale.
- **XDG Base Directory compliance**: the CLI and agent now follow the
  [XDG spec](https://specifications.freedesktop.org/basedir-spec/latest/) on
  every platform for config, state, data, and cache files. A legacy
  flat `~/.vouch/` directory is migrated automatically on first run.
- **Mandatory HTTP message signatures**: all `/v1/*` API requests now require
  [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) HTTP message signatures
  with Content-Digest enforcement, and the server advertises its policy via
  `Accept-Signature`.
- **OpenID Connect RP-Initiated Logout 1.0**: applications can now end a
  Vouch session through the standard
  [RP-initiated logout flow](https://openid.net/specs/openid-connect-rpinitiated-1_0.html).
- **STIG-aligned kernel hardening**: the server AMI ships with
  [STIG](https://public.cyber.mil/stigs/)-aligned kernel settings and a
  published STIG mapping document.

## [v2026.6.2](https://github.com/vouch-sh/vouch/releases/tag/v2026.6.2) - June 13, 2026

- **Security evaluation remediation**: a prioritized internal security
  evaluation was completed and all P1 and P2 findings were remediated,
  including an SSRF egress guard for client-controlled JWKS and `request_uri`
  fetches.
- **Server-rendered security-keys page**: key management moved to full
  server-side rendering with form POSTs, reducing the JavaScript surface.
- **FIPS crypto-policy in the attestable AMI**: the hardened server image now
  enables the FIPS cryptographic policy.

## [v2026.5.4](https://github.com/vouch-sh/vouch/releases/tag/v2026.5.4) - May 30, 2026

- **Workload identity federation for AI APIs**: exchange a Vouch session for
  short-lived Anthropic (Claude) and OpenAI API credentials, so AI tooling
  runs without long-lived API keys on disk.

## [v2026.5.2](https://github.com/vouch-sh/vouch/releases/tag/v2026.5.2) - May 17, 2026

- **Multiple upstream IdPs**: a single Vouch server can now federate with
  more than one upstream identity provider.
- **Multi-domain organizations**: organizations can span multiple email
  domains.
- **Cross-account KMS keys**: the server can use KMS signing keys that live
  in a different AWS account.

## [v2026.5.1](https://github.com/vouch-sh/vouch/releases/tag/v2026.5.1) - May 11, 2026

- **DNS-over-HTTPS**: the CLI and agent resolve the Vouch server over
  encrypted DNS, hardening credential flows on untrusted networks.

## [v2026.4.8](https://github.com/vouch-sh/vouch/releases/tag/v2026.4.8) - May 1, 2026

- **Windows support matured**: Vouch is published to winget
  (`winget install SmokeTurner.Vouch`) and Windows binaries are code-signed.
- **Clock-skew detection**: the CLI warns when local clock drift would cause
  token validation failures.

## [v2026.4.6](https://github.com/vouch-sh/vouch/releases/tag/v2026.4.6) - April 27, 2026

- **SSH certificate caching**: issued SSH certificates are cached for their
  lifetime instead of being re-requested on every connection.
- **AI agent attribution**: when an AI agent invokes AWS credential helpers,
  the `AI_AGENT` environment variable is forwarded into session tags for
  CloudTrail attribution, with a session policy limiting role chaining.
- **IAM role paths**: roles with paths (e.g. `/engineering/deploy`) are fully
  supported.

## [v2026.4.5](https://github.com/vouch-sh/vouch/releases/tag/v2026.4.5) - April 21, 2026

- **AWS multi-account SSO**: automatic discovery of accounts and roles across
  an AWS organization, with role chaining.
- **`vouch aws console`**: open the AWS web console from the terminal using
  your short-lived credentials.
- **RFC 9728 Protected Resource Metadata**: the server publishes
  [OAuth 2.0 protected resource metadata](https://www.rfc-editor.org/rfc/rfc9728)
  for standards-based client discovery.
