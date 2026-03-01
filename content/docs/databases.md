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
2. **`vouch credential rds`** or **`vouch credential redshift`** -- Vouch generates a short-lived database authentication token or temporary credentials directly.
3. **Database client** -- The token is passed as the password to `psql`, `mysql`, or another database client.

```
vouch login → vouch credential rds → database client
vouch login → vouch credential redshift → database client
```

Or use `vouch exec` to skip manual token handling entirely:

```
vouch login → vouch exec --type rds -- psql
vouch login → vouch exec --type redshift -- psql
```

---

## RDS / Aurora PostgreSQL

### Using Vouch CLI

The simplest approach is `vouch exec`, which generates the token and injects PostgreSQL environment variables (`PGPASSWORD`, `PGHOST`, `PGPORT`, `PGUSER`, `PGSSLMODE=require`) automatically:

```bash
vouch exec --type rds \
  --rds-hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --rds-username mydbuser \
  -- psql -d mydb
```

To set the variables in your current shell instead:

```bash
eval "$(vouch env --type rds \
  --rds-hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --rds-username mydbuser)"
psql -d mydb
```

To generate just the token (e.g., for scripts or non-PostgreSQL clients):

```bash
TOKEN=$(vouch credential rds \
  --hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --username mydbuser)
```

### Using AWS CLI

You can also use the AWS CLI with Vouch's `credential_process` integration:

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

MySQL requires the `--enable-cleartext-plugin` flag because the IAM token is sent as a cleartext password over TLS.

### Using Vouch CLI

Generate the token with `vouch credential rds` and pass it to the MySQL client:

```bash
TOKEN=$(vouch credential rds \
  --hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --username mydbuser \
  --port 3306)

mysql -h mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  -P 3306 \
  -u mydbuser \
  --password="$TOKEN" \
  --ssl-mode=REQUIRED \
  --enable-cleartext-plugin
```

> **Note:** `vouch exec --type rds` and `vouch env --type rds` inject PostgreSQL-style environment variables (`PGPASSWORD`, `PGHOST`, etc.), so MySQL users should use `vouch credential rds` to get the token and pass it manually.

### Using AWS CLI

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

Redshift generates temporary database credentials with a configurable lifetime (15--60 minutes). Vouch supports both provisioned clusters and Redshift Serverless workgroups.

### Using Vouch CLI (provisioned cluster)

The simplest approach is `vouch exec`, which generates credentials and injects PostgreSQL environment variables (`PGPASSWORD`, `PGUSER`, `PGSSLMODE=require`) automatically:

```bash
vouch exec --type redshift \
  --redshift-cluster-id my-cluster \
  --redshift-db-name mydb \
  -- psql -h my-cluster.abc123.us-east-1.redshift.amazonaws.com -p 5439
```

To set the variables in your current shell:

```bash
eval "$(vouch env --type redshift \
  --redshift-cluster-id my-cluster \
  --redshift-db-name mydb)"
psql -h my-cluster.abc123.us-east-1.redshift.amazonaws.com -p 5439
```

To generate just the credentials:

```bash
vouch credential redshift --cluster-id my-cluster --db-name mydb
```

The `--duration` flag controls credential lifetime for provisioned clusters (900--3600 seconds, default: 900):

```bash
vouch credential redshift --cluster-id my-cluster --duration 3600
```

### Using Vouch CLI (Redshift Serverless)

```bash
vouch exec --type redshift \
  --redshift-workgroup my-workgroup \
  --redshift-db-name mydb \
  -- psql -h my-workgroup.123456789012.us-east-1.redshift-serverless.amazonaws.com -p 5439
```

Or generate credentials directly:

```bash
vouch credential redshift --workgroup my-workgroup --db-name mydb
```

### Using AWS CLI

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

Your Vouch IAM role needs permission to generate database auth tokens and credentials.

**RDS / Aurora:**

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

**Redshift (provisioned clusters):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "redshift:GetClusterCredentialsWithIAM",
      "Resource": "arn:aws:redshift:us-east-1:123456789012:dbname:my-cluster/*"
    }
  ]
}
```

**Redshift Serverless:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "redshift-serverless:GetCredentials",
      "Resource": "arn:aws:redshift-serverless:us-east-1:123456789012:workgroup/*"
    }
  ]
}
```

---

## Troubleshooting

### "PAM authentication failed for user"

Ensure IAM database authentication is enabled on the RDS instance and the database user has the `rds_iam` role (PostgreSQL) or was created with `AWSAuthenticationPlugin` (MySQL).

### Token expired

RDS/Aurora auth tokens are valid for 15 minutes. Generate a fresh token before connecting. The token is only used to establish the connection -- active sessions are not affected by expiry.

### SSL required error

IAM database authentication requires SSL/TLS. Use `sslmode=require` for PostgreSQL or `--ssl-mode=REQUIRED` for MySQL.
