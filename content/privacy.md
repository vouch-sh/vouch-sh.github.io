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
- **Display name** -- Your full name as provided by your organization's directory.
- **Organization membership** -- Which organization and groups you belong to, as determined by your IdP.

We do not ask you to create a separate username or password. Your identity is established entirely through your organization's existing identity provider.

### Authentication Data

When you register a security key and authenticate with Vouch, we collect:

- **FIDO2/WebAuthn credential identifiers** -- A public key and credential ID generated during enrollment. These are used to verify your identity during subsequent logins.
- **Attestation metadata** -- Information about the type and manufacturer of your security key (e.g., YubiKey 5 series), used for security policy enforcement.

**Vouch never stores your private keys.** The private key component of your FIDO2 credential never leaves your hardware security key. We store only the public key, which cannot be used to impersonate you.

### Usage Logs

We collect operational logs to maintain security and troubleshoot issues:

- **Authentication events** -- Timestamps of successful and failed login attempts.
- **IP addresses** -- The network address from which authentication requests originate.
- **Credential usage** -- Records of when short-lived credentials (SSH certificates, AWS STS tokens, OIDC tokens) are issued, including the type of credential and its expiration time.
- **Administrative actions** -- Logs of administrative operations such as user enrollment, key registration, and SCIM provisioning events.

### Device Information

When you use a hardware security key with Vouch, we collect:

- **Security key metadata** -- The make, model, and firmware version of your FIDO2 security key, as reported through the WebAuthn attestation process.
- **Key capabilities** -- Supported authentication features (e.g., user verification, resident keys).

This information is used to enforce security policies (for example, requiring keys that support user verification) and to help administrators inventory the security keys in use across the organization.

---

## How We Use Your Information

We use the information we collect for the following purposes:

- **Authenticate your identity and issue credentials** -- This is the core function of Vouch. Your identity information and FIDO2 credentials are used to verify you are who you claim to be and to issue short-lived SSH certificates, AWS STS credentials, OIDC tokens, and other credentials.
- **Maintain security and detect unauthorized access** -- Authentication logs, IP addresses, and usage data are used to detect anomalous behavior, investigate potential security incidents, and enforce organizational security policies.
- **Comply with legal obligations** -- We may retain and disclose information as required by applicable laws, regulations, or legal processes.
- **Improve the Service** -- Aggregated, anonymized usage data may be used to understand how the Service is used and to identify areas for improvement. We do not use individual user data for this purpose without anonymization.

---

## Data Retention

- **Authentication logs** are retained according to your organization's configured retention policies. By default, authentication events are retained for 90 days. Your organization's administrator may configure a shorter or longer retention period.
- **Credential data** (FIDO2 public keys and credential identifiers) is retained for as long as the credential is registered. When you remove a security key or when an administrator de-provisions your account, the associated credential data is deleted.
- **Session tokens** are short-lived, with a maximum lifetime of **8 hours**. They auto-expire and are not retained after expiration. There are no long-lived session tokens or refresh tokens.
- **Account information** is retained for as long as your account is active within your organization. When your account is de-provisioned (either through SCIM or manual administrative action), your account data is marked as inactive. Your organization's administrator can request permanent deletion.

---

## Data Security

We employ multiple layers of security to protect your information:

- **TLS encryption** -- All communication between the Vouch CLI, your browser, and the Vouch server is encrypted using TLS 1.2 or higher. Data is encrypted both in transit and at rest.
- **FIDO2/WebAuthn phishing-resistant authentication** -- Vouch exclusively uses the FIDO2/WebAuthn standard for user authentication. This protocol is resistant to phishing, credential stuffing, and man-in-the-middle attacks because the cryptographic assertion is bound to the origin (domain) of the Vouch server.
- **Private keys never leave your security device** -- The private key associated with your FIDO2 credential is generated on and never exported from your hardware security key. Even if the Vouch server were compromised, your private key would remain safe.
- **Short-lived credentials minimize breach exposure** -- All credentials issued by Vouch (SSH certificates, AWS STS tokens, OIDC tokens) have a maximum lifetime of 8 hours. This dramatically limits the window of exposure if a credential is intercepted or a system is compromised. There are no long-lived secrets to steal.
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

Vouch uses only essential session cookies required for authentication. These cookies are necessary for the Service to function and cannot be disabled. We do not use analytics cookies, advertising trackers, or any third-party tracking technologies. No user behavior is tracked across websites.

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
