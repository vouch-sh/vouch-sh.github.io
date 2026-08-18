# Audit Log Export

> Pull audit events into your SIEM with the audit events API — cursor-based polling, NDJSON streaming, and OCSF 1.9.0 projection.

Source: https://vouch.sh/docs/audit-export/
Last updated: 2026-08-18

---
Every authentication, credential issuance, and administrative action in Vouch is recorded as an audit event. The [admin dashboard](/docs/admin/#audit-log) provides a browsable, filterable view; this page covers **programmatic access** — SIEM ingestion, backfills, and ad hoc scripting — via the audit events API.

Email addresses in audit events are masked to domain only, with an HMAC column alongside so you can correlate a user&#39;s activity without the log holding their address. Events are enriched with the country code, ASN, and network organization resolved from the client IP.

---

## The audit events API

```
GET /api/v1/org/audit-events
```

Returns audit events scoped to your organization (the primary domain plus any [verified additional domain](/docs/domains/)) in ID order.

### Authentication

Two methods are accepted. Browser cookie auth is rejected outright — the endpoint is designed for unattended pollers as much as interactive use.

- **Org API token with the `audit:read` scope** — the same token type used for SCIM provisioning, generalized to carry additional scopes. Mint one on `/admin/scim-tokens` by checking **&#34;Also grant read-only audit log access&#34;**, or via `POST /api/v1/org/scim-tokens` with `&#34;audit_read&#34;: true`. A token minted without that option is rejected with `403`.
- **Org-admin user session** — a FIDO2-authenticated org admin&#39;s access token (`Authorization: Bearer` or `DPoP`), the same credential used for the other `/api/v1/org/*` endpoints.

```bash
curl -H &#34;Authorization: Bearer $VOUCH_AUDIT_TOKEN&#34; \
  &#34;https://us.vouch.sh/api/v1/org/audit-events&#34;
```

### Filters

| Parameter | Description |
|---|---|
| `event_type` | Comma-separated list of event types (e.g. `login_success,login_failed`). Unknown or empty values return `400` rather than silently matching nothing. |
| `user_id` | Exact match. |
| `email` | Exact match (HMAC lookup, case-insensitive). |
| `since` / `until` | RFC 3339 timestamps; only events strictly after `since` and strictly before `until`. |
| `after` | Forward cursor: the `id` of the last event from a previous page. Returns events oldest-first — the shape a poller wants. Takes precedence over `before`. |
| `before` | Backward cursor. Returns events newest-first, matching the admin dashboard view. |
| `limit` | Page size, default 500, maximum 1000. |
| `format` | `ocsf` to project events into OCSF (see below); omit for native JSON. |

With neither cursor set, the first call defaults to an ascending walk from the start of retained history — a poller with no saved cursor can call the endpoint with no parameters and start following `next_cursor` forward.

### Response

```json
{
  &#34;events&#34;: [
    {
      &#34;id&#34;: &#34;01920000-...&#34;,
      &#34;event_type&#34;: &#34;login_success&#34;,
      &#34;user_id&#34;: &#34;01910000-...&#34;,
      &#34;email_domain&#34;: &#34;example.com&#34;,
      &#34;email_hmac&#34;: &#34;9f86d0...&#34;,
      &#34;created_at&#34;: &#34;2026-01-01T00:00:03.512Z&#34;,
      &#34;data&#34;: { &#34;authenticator_id&#34;: &#34;...&#34; }
    }
  ],
  &#34;next_cursor&#34;: &#34;01920000-...&#34;
}
```

`email_hmac` is the documented correlation key for tying events to a specific user without storing their address in the log, and is already org-scoped.

---

## Cursor semantics and the delivery guarantee

`next_cursor` is present whenever there may be more matching events; pass it back as `after` (or `before`, if walking backward) to continue. Event IDs are UUIDv7 (time-ordered), but authentication events are written from detached background tasks, so commit order can trail ID order under load — a naive poller that just tracks &#34;the highest ID seen&#34; can miss events that commit late.

The API&#39;s guarantee is therefore about time, not ID order: **an event is never returned with `created_at` newer than `now − 30s`**. A poller that requests `after=&lt;last cursor&gt;` no more often than every 30 seconds, and persists the returned `next_cursor` after each successful page, will not miss events that commit within that window. Always follow `next_cursor` until a page comes back without one — pages can be size-capped, so one poll does not necessarily drain everything new.

Treat this as a best-effort guarantee under normal operating conditions: `created_at` is stamped when the event is minted, not when it commits, so a write delayed beyond 30 seconds by write-path contention can still land later than a poller expects. If your environment is prone to write-path contention, use a polling interval longer than 30 seconds.

Polling this endpoint does not itself write an audit event — that would create a feedback loop of one event per poll.

---

## NDJSON streaming

Send `Accept: application/x-ndjson` to receive one JSON object per line instead of the envelope — useful for appending to a file or piping into `jq`. Responses are capped at 5 MiB; if a page would exceed that, the response stops at the last complete line and a `Link: &lt;...&gt;; rel=&#34;next&#34;` header carries the cursor for the rest. Follow it the same way you&#39;d follow `next_cursor`.

```bash
curl -H &#34;Authorization: Bearer $VOUCH_AUDIT_TOKEN&#34; \
     -H &#34;Accept: application/x-ndjson&#34; \
     &#34;https://us.vouch.sh/api/v1/org/audit-events&#34; | jq -c .
```

---

## OCSF projection

`?format=ocsf` projects each event into [OCSF](https://ocsf.io) 1.9.0, mapping Vouch&#39;s ~40 event types onto four Identity &amp; Access Management classes:

| OCSF class | Vouch events |
|---|---|
| **Authentication (3002)** | Logins, logout, device authorization, policy denials |
| **Account Change (3001)** | Enrollment, key registration/removal, admin promote/demote/activate/deactivate, identity binding |
| **Authorize Session (3003)** | SSH, AWS, and GitHub credential issuance, token exchange, OAuth token issue/revoke |
| **Entity Management (3004)** | SCIM operations, OAuth client lifecycle, posture policies, SCIM tokens, domain and subdomain lifecycle |

Native JSON stays the canonical, lossless representation — the projection is for SIEM ingestion, and every field Vouch records is still present in `data`. Events with no predefined OCSF activity carry a source-specific `activity_name` and preserve the original `event_type` in `unmapped.event_type` for cross-product correlation.

### SIEM poller configuration

- **Microsoft Sentinel** (Codeless Connector Framework `RestApiPoller`) — poll on an interval, carry `next_cursor` forward as the `after` query parameter between polls, and treat the 30-second lag window as the platform&#39;s ingestion delay tolerance.
- **Splunk / Elastic (generic HTTP poll)** — configure a REST/HTTP input against `GET /api/v1/org/audit-events?format=ocsf` with the bearer token, checkpoint on the response&#39;s `next_cursor`, and poll no more frequently than every 30 seconds.

---

## Retention

Events fall into three retention classes. The class is a property of the event type and is not configurable per event:

| Class | Default retention | Contains |
|---|---|---|
| Authentication | 90 days | Logins, enrollment, logout, key and device-auth lifecycle, SCIM operations |
| OAuth and credentials | 90 days | Credential issuance, token issue/revoke, client registration — the high-volume events |
| **Kept forever** | Never deleted | Every administrative action, OAuth client and secret lifecycle, and all domain, subdomain, and issuer-key events |

The third class is the one to know about: administrative and organization-lifecycle records are never purged, because they answer &#34;who granted this person admin, and when.&#34; On self-hosted deployments the two retention windows are configurable (`VOUCH_AUTH_EVENTS_RETENTION_DAYS`, `VOUCH_OAUTH_EVENTS_RETENTION_DAYS`) — see the [server documentation](https://docs.vouch.sh) for operator details.
