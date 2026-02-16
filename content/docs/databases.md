---
title: "Connect to RDS and Aurora without Database Passwords"
linkTitle: "Databases"
description: "Replace static database passwords with 15-minute IAM auth tokens generated from hardware-backed credentials."
weight: 13
subtitle: "Connect to RDS, Aurora, and Redshift using IAM database authentication"
params:
  docsGroup: infra
---

Static database passwords are shared across developers, stored in configuration files, and rarely rotated. When someone leaves the team, do you rotate every database password they had access to? Most teams don't.

[IAM database authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html) replaces static passwords with 15-minute tokens generated from IAM credentials. With Vouch, those IAM credentials are themselves hardware-backed and short-lived -- every database connection traces back to a verified human identity.

## How it works

1. **`vouch login`** -- The developer authenticates with their YubiKey.
2. **`credential_process`** -- The AWS CLI calls Vouch to obtain temporary STS credentials.
3. **`generate-db-auth-token`** -- The AWS CLI uses the STS credentials to generate a short-lived database authentication token.
4. **Database client** -- The token is passed as the password to `psql`, `mysql`, or another database client.

```
vouch login → credential_process → STS → generate-db-auth-token → database client
```

---

## RDS / Aurora PostgreSQL

Generate a 15-minute authentication token and pass it as the password to `psql`:

```bash
# Generate an IAM auth token (valid for 15 minutes)
TOKEN=$(aws rds generate-db-auth-token \
  --hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --username mydbuser \
  --profile vouch)

# Connect with psql
PGPASSWORD="$TOKEN" psql \
  -h mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  -p 5432 \
  -U mydbuser \
  -d mydb \
  "sslmode=require"
```

**Database setup:** The database user must be configured for IAM authentication. For PostgreSQL, grant the `rds_iam` role:

```sql
GRANT rds_iam TO mydbuser;
```

---

## RDS / Aurora MySQL

MySQL requires the `--enable-cleartext-plugin` flag because the IAM token is sent as a cleartext password over TLS:

```bash
# Generate an IAM auth token
TOKEN=$(aws rds generate-db-auth-token \
  --hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --port 3306 \
  --username mydbuser \
  --profile vouch)

# Connect with mysql (note: --enable-cleartext-plugin is required)
mysql -h mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  -P 3306 \
  -u mydbuser \
  --password="$TOKEN" \
  --ssl-mode=REQUIRED \
  --enable-cleartext-plugin
```

**Database setup:** Create the user with the `AWSAuthenticationPlugin`:

```sql
CREATE USER 'mydbuser'@'%' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
```

---

## Amazon Redshift

Redshift uses `get-cluster-credentials` to generate temporary database credentials with a configurable lifetime (15--60 minutes):

```bash
# Get temporary Redshift credentials
CREDS=$(aws redshift get-cluster-credentials \
  --cluster-identifier my-cluster \
  --db-user mydbuser \
  --db-name mydb \
  --duration-seconds 3600 \
  --profile vouch)

# Extract and connect
DB_USER=$(echo "$CREDS" | jq -r '.DbUser')
DB_PASS=$(echo "$CREDS" | jq -r '.DbPassword')

PGPASSWORD="$DB_PASS" psql \
  -h my-cluster.abc123.us-east-1.redshift.amazonaws.com \
  -p 5439 \
  -U "$DB_USER" \
  -d mydb
```

---

## Required IAM permissions

Your Vouch IAM role needs permission to generate database auth tokens:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:us-east-1:123456789012:dbuser:cluster-ABC123/mydbuser"
    }
  ]
}
```

For Redshift, use `redshift:GetClusterCredentials` instead.

---

## Troubleshooting

### "PAM authentication failed for user"

Ensure IAM database authentication is enabled on the RDS instance and the database user has the `rds_iam` role (PostgreSQL) or was created with `AWSAuthenticationPlugin` (MySQL).

### Token expired

RDS/Aurora auth tokens are valid for 15 minutes. Generate a fresh token before connecting. The token is only used to establish the connection -- active sessions are not affected by expiry.

### SSL required error

IAM database authentication requires SSL/TLS. Use `sslmode=require` for PostgreSQL or `--ssl-mode=REQUIRED` for MySQL.
