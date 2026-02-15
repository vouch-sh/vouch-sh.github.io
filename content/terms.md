---
title: "Terms of Service"
description: "Terms of service for the Vouch authentication service."
---

**Last updated: January 2026**

## Acceptance of Terms

By accessing or using the Vouch authentication service ("Vouch" or the "Service"), you agree to be bound by these Terms of Service ("Terms"). If you do not agree to these Terms, you may not use the Service.

Your use of Vouch is also governed by the [Privacy Policy](/privacy/), which describes how we collect, use, and protect your information.

These Terms constitute a legal agreement between you ("User" or "you") and your organization that operates the Vouch deployment. The Service is provided by Smoke Turner, LLC and is deployed and managed by your organization.

---

## Description of Service

Vouch is a hardware-backed authentication and credential brokering service that provides:

- **FIDO2/WebAuthn authentication** -- Vouch uses the FIDO2/WebAuthn standard to verify your identity through a hardware security key (such as a YubiKey). Authentication requires both a PIN and physical touch of the key, providing strong two-factor security.
- **Short-lived SSH certificates** -- After successful authentication, Vouch issues SSH certificates signed by your organization's certificate authority. These certificates are valid for a maximum of 8 hours and replace the need for static SSH keys.
- **Cloud credential brokering** -- Vouch issues short-lived AWS STS credentials, OIDC tokens, and other cloud credentials that expire automatically. This eliminates the need for long-lived API keys or access tokens.
- **Identity provider integration** -- Vouch integrates with your organization's identity provider (IdP) to establish your identity. It supports SAML, OIDC, and SCIM protocols for authentication and user provisioning.

The Service is designed to replace long-lived secrets (SSH keys, API tokens, cloud credentials) with short-lived, hardware-backed credentials that are issued on demand and expire automatically.

---

## User Responsibilities

As a user of the Vouch service, you agree to the following responsibilities:

- **Keep your security key secure and report loss immediately.** Your hardware security key is the primary factor used to authenticate your identity. You are responsible for the physical security of your key. If your key is lost, stolen, or compromised, report it to your organization's IT administrator immediately so that the associated credentials can be revoked.
- **Do not share credentials.** Credentials issued by Vouch -- including SSH certificates, session tokens, AWS STS credentials, and OIDC tokens -- are personal to you and must not be shared with others. Each user must authenticate independently using their own security key.
- **Use the Service only for authorized purposes.** You may use Vouch only to access systems and resources that you are authorized to access as part of your role within your organization. Do not use Vouch to access systems you are not permitted to use.
- **Comply with your organization's security policies.** Your organization may have additional security policies that govern the use of Vouch, including policies about acceptable use, security key management, and credential handling. You must comply with all applicable organizational policies.
- **Do not circumvent security controls.** You must not attempt to bypass, disable, or circumvent any security mechanism provided by Vouch, including but not limited to: the FIDO2 authentication requirement, session expiration, credential scoping, rate limiting, or IP-based access controls.

---

## Security Key Requirements

To use Vouch, you must obtain and maintain a **FIDO2-compatible hardware security key** that supports **user verification** (UV). Most YubiKey 5 series keys and comparable FIDO2 security keys meet this requirement. Specifically:

- **You must obtain a compatible security key.** Your organization may provide one, or you may need to purchase your own. Consult your organization's IT team for guidance on approved key models.
- **You must maintain your security key in working order.** Keep the key's firmware up to date (if applicable) and ensure it remains functional. If your key is damaged or malfunctioning, contact your IT administrator for a replacement.
- **You must set up a PIN on your security key.** FIDO2 user verification requires a PIN. If your key does not already have a PIN configured, the Vouch enrollment process will prompt you to set one. Do not share your PIN with anyone.
- **You are strongly encouraged to register backup keys.** If your primary security key is lost or damaged, a registered backup key allows you to continue authenticating without requiring administrator intervention. Vouch supports registering multiple keys per user. Contact your IT administrator for your organization's policy on backup keys.

---

## Service Availability

Your organization strives to maintain high availability of the Vouch service. However, availability is not guaranteed. The Service may be unavailable:

- During scheduled maintenance windows, which your organization will communicate in advance when possible.
- During unscheduled outages caused by infrastructure failures, network issues, or other unforeseen events.
- During security incidents that require the Service to be temporarily taken offline.

Because Vouch issues short-lived credentials, an outage of the Vouch server does not immediately revoke access. Credentials that were issued before the outage remain valid until their natural expiration (up to 8 hours). However, you will not be able to obtain new credentials until the Service is restored.

Your organization is responsible for the operational management and availability of its Vouch deployment. Smoke Turner, LLC does not guarantee uptime for individual organization deployments.

---

## Limitation of Liability

TO THE MAXIMUM EXTENT PERMITTED BY APPLICABLE LAW, NEITHER SMOKE TURNER, LLC NOR YOUR ORGANIZATION SHALL BE LIABLE FOR ANY INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL, OR PUNITIVE DAMAGES ARISING OUT OF OR RELATED TO YOUR USE OF OR INABILITY TO USE THE SERVICE. THIS INCLUDES, BUT IS NOT LIMITED TO:

- Loss of data, revenue, or profits.
- Business interruption.
- Loss of access to systems or resources.
- Damages resulting from unauthorized access to your accounts or credentials.
- Damages arising from security key loss, theft, or malfunction.

This limitation applies regardless of the legal theory on which the claim is based, whether in contract, tort (including negligence), strict liability, or otherwise, even if the liable party has been advised of the possibility of such damages.

Some jurisdictions do not allow the exclusion or limitation of incidental or consequential damages. In those jurisdictions, the above limitation applies to the fullest extent permitted by law.

---

## Termination

Your access to the Vouch service may be terminated:

- **By your organization** at any time, for any reason, with or without notice. Your organization's administrators have full authority to de-provision your account.
- **Upon termination of your employment or affiliation** with the organization. When your relationship with the organization ends, your Vouch account will be deactivated, typically through automated SCIM de-provisioning.
- **For violations of these Terms or your organization's policies.** If you violate these Terms, your organization's acceptable use policies, or any applicable security policies, your access may be terminated immediately without prior notice.

Upon termination, regardless of the reason:

- All active sessions will be revoked.
- All previously issued credentials (SSH certificates, OIDC tokens, AWS STS credentials) will be invalidated.
- Your FIDO2 credential registrations will be removed.
- You will no longer be able to authenticate or obtain new credentials.
- Your account data will be retained or deleted according to the organization's data retention policies and the [Privacy Policy](/privacy/).

---

## Changes to Terms

These Terms of Service may be updated from time to time. When significant changes are made, your organization will notify you through appropriate channels (e.g., email, internal communication tools, or a banner within the Service).

Your continued use of the Vouch service after changes are published constitutes your acceptance of the revised Terms. If you do not agree with the updated Terms, you should discontinue use of the Service and contact your organization's IT administrator.

---

## Contact

For questions about these Terms of Service, your use of Vouch, or any related concerns, contact your **organization's IT administrator or security team**. They manage your organization's Vouch deployment and can address questions about account management, security policies, and access control.

For questions about the Vouch software itself, contact Smoke Turner, LLC.
