---
title: "Privacy Policy"
description: "Privacy policy for the Vouch authentication service."
---

**Last updated: February 2026**

## Introduction

Smoke Turner, LLC ("we", "us", or "our") operates the Vouch authentication service ("Vouch" or the "Service"). This Privacy Policy describes how we collect, use, and protect your personal information when you use Vouch through your organization's deployment.

Vouch is designed with a privacy-first approach. We collect only the minimum information necessary to authenticate your identity and issue short-lived credentials. We do not sell your data, do not use it for advertising, and do not share it with third parties except as described in this policy.

---

## Information We Collect

### Account Information

When you enroll with Vouch, we receive basic identity information from your organization's identity provider (IdP):

- **Email address** -- Used as your primary identifier within Vouch.
- **Display name** (if available) -- Your name as provided by your organization's directory through SCIM provisioning. Not all accounts will have a display name.
- **Organization membership** -- Which organization and groups you belong to, as determined by your IdP.

We do not ask you to create a separate username or password. Your identity is established entirely through your organization's existing identity provider.

### GitHub Integration Data

If your organization uses the Vouch GitHub integration, we store:

- **GitHub user ID and username** -- Used to associate your Vouch identity with your GitHub account for repository access.
- **GitHub OAuth refresh token** -- Used to obtain short-lived GitHub access tokens on your behalf. Deleted when your account is de-provisioned or the GitHub integration is disconnected.

### Authentication Data

When you register a security key and authenticate with Vouch, we collect:

- **FIDO2/WebAuthn credential identifiers** -- A public key and credential ID generated during enrollment. These are used to verify your identity during subsequent logins.
- **Attestation metadata** -- The AAGUID (Authenticator Attestation Globally Unique Identifier) of your security key, which identifies its make and model (e.g., "YubiKey 5 series"). We do not store firmware versions, serial numbers, or device-unique identifiers. This is used to enforce security policies and help administrators inventory security key types in use.

**Vouch never stores your private keys.** The private key component of your FIDO2 credential never leaves your hardware security key. We store only the public key, which cannot be used to impersonate you.

### Usage Logs

We collect operational logs to maintain security and troubleshoot issues:

- **Authentication events** -- Timestamps and outcomes of login attempts, enrollments, and logouts.
- **IP addresses** -- The network address from which requests originate.
- **Client environment** -- The user-agent string and Vouch CLI version reported during authentication, used for security monitoring.
- **Credential issuance records** -- Records of when short-lived credentials (SSH certificates, AWS STS tokens, OIDC tokens, GitHub access tokens) are issued, including the credential type and expiration time.
- **Administrative actions** -- Logs of administrative operations such as user enrollment, key registration, and SCIM provisioning events.

---

## How We Use Your Information

We use the information we collect for the following purposes:

- **Authenticate your identity and issue credentials** -- This is the core function of Vouch. Your identity information and FIDO2 credentials are used to verify you are who you claim to be and to issue short-lived SSH certificates, AWS STS credentials, OIDC tokens, and other credentials.
- **Facilitate GitHub repository access** -- If your organization uses the GitHub integration, your GitHub identity and OAuth tokens are used to obtain short-lived access tokens scoped to authorized repositories.
- **Maintain security and detect unauthorized access** -- Authentication logs, IP addresses, and usage data are used to detect anomalous behavior, investigate potential security incidents, and enforce organizational security policies.
- **Comply with legal obligations** -- We may retain and disclose information as required by applicable laws, regulations, or legal processes.
- **Improve the Service** -- Aggregated, anonymized usage data may be used to understand how the Service is used and to identify areas for improvement. We do not use individual user data for this purpose without anonymization.

---

## Data Retention

- **Authentication event logs** are retained for **90 days** by default. Your organization's administrator may configure a different retention period.
- **OAuth and credential issuance logs** are retained for **90 days** by default.
- **SCIM provisioning logs** are retained for **90 days** by default.
- **Credential data** (FIDO2 public keys and credential identifiers) is retained for as long as the credential is registered. When you remove a security key or your account is de-provisioned, the associated credential data is deleted.
- **Session data** is short-lived, with a default lifetime of **8 hours** (configurable by your organization's administrator). Sessions auto-expire and are cleaned up automatically.
- **GitHub OAuth tokens** are stored for the duration of your active account and deleted upon de-provisioning.
- **Account information** is retained for as long as your account is active. When de-provisioned, your data is marked inactive. Your organization's administrator can request permanent deletion.

---

## Data Security

We employ multiple layers of security to protect your information:

- **TLS encryption** -- All communication between the Vouch CLI, your browser, and the Vouch server is encrypted using TLS 1.2 or higher. Data is encrypted both in transit and at rest.
- **FIDO2/WebAuthn phishing-resistant authentication** -- Vouch exclusively uses the FIDO2/WebAuthn standard for user authentication. This protocol is resistant to phishing, credential stuffing, and man-in-the-middle attacks because the cryptographic assertion is bound to the origin (domain) of the Vouch server.
- **Private keys never leave your security device** -- The private key associated with your FIDO2 credential is generated on and never exported from your hardware security key. Even if the Vouch server were compromised, your private key would remain safe.
- **Short-lived credentials minimize breach exposure** -- All credentials issued by Vouch (SSH certificates, AWS STS tokens, OIDC tokens) have a short lifetime (8 hours by default). This dramatically limits the window of exposure if a credential is intercepted or a system is compromised. There are no long-lived secrets to steal.
- **SCIM tokens are stored as hashes** -- SCIM provisioning tokens are stored only as cryptographic hashes. The plaintext value is shown once at creation and is never stored or recoverable.

---

## Your Rights

Depending on your jurisdiction, you may have the following rights regarding your personal information:

- **Access** -- You have the right to request a copy of the personal information we hold about you.
- **Correction** -- You have the right to request correction of inaccurate personal information. Note that most account information is sourced from your organization's identity provider; corrections should typically be made there.
- **Deletion** -- You have the right to request deletion of your personal information. Account deletion requests should be directed to your organization's administrator, who can de-provision your account through SCIM or the Vouch administrative interface.
- **Data portability** -- You have the right to request your data in a structured, commonly used, machine-readable format.

To exercise any of these rights, contact your organization's IT administrator or security team. They can process your request directly or escalate it as needed.

---

## Contact

For questions about this Privacy Policy or about how your data is handled within Vouch, contact your **organization's IT administrator or security team**. They manage your organization's Vouch deployment and can address questions about data collection, retention, and deletion.

For questions about the Vouch software itself, contact Smoke Turner, LLC at [privacy@vouch.sh](mailto:privacy@vouch.sh).

---

## Cookies and Tracking

Vouch uses a single session cookie (`vouch_session`) required for authentication. This cookie is set with `HttpOnly`, `Secure`, and `SameSite=Lax` attributes — it cannot be read by JavaScript, is only sent over HTTPS, and is not sent in cross-site requests. We do not use analytics cookies, advertising trackers, or any third-party tracking technologies.

---

## International Data Transfers

The Vouch service is hosted in the United States. If you access the Service from outside the United States, your information will be transferred to, stored, and processed in the United States. By using the Service, you consent to the transfer of your information to the United States.

---

## Children's Privacy

Vouch is not directed at children under the age of 13. We do not knowingly collect personal information from children under 13. If we become aware that a child under 13 has provided personal information, we will take steps to delete that information. If you believe a child under 13 has provided personal information to us, please contact us at [privacy@vouch.sh](mailto:privacy@vouch.sh).

---

## Data Controller and Processor

Under applicable data protection laws, your organization (the entity that operates the Vouch deployment) is the **data controller** responsible for determining the purposes and means of processing your personal information. Smoke Turner, LLC acts as a **data processor**, processing personal information on behalf of your organization according to their instructions and the terms of the applicable service agreement.

For questions about how your personal data is processed, contact your organization's data protection officer or IT administrator. For questions about Smoke Turner, LLC's data processing practices, contact [privacy@vouch.sh](mailto:privacy@vouch.sh).
