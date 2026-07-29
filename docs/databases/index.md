# Connect to RDS and Aurora without Database Passwords

> Replace static database passwords with 15-minute IAM auth tokens generated from hardware-backed credentials.

Source: https://vouch.sh/docs/databases/
Last updated: 2026-07-04

---
[IAM database authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html) replaces static database passwords with short-lived tokens generated from IAM credentials -- with Vouch, hardware-backed ones.

{{&lt; tldr &gt;}}
- **Prerequisites:** [Getting Started](/docs/getting-started/) → [AWS integration](/docs/aws/) → this page.
- **Admin, once:** grant [`rds-db:connect`](#required-iam-permissions) on the Vouch IAM role and enable IAM auth on the database user.
- **Each developer:** `vouch exec --type rds --rds-hostname &lt;host&gt; --rds-username &lt;user&gt; -- psql`, then `psql` just works — the 15-minute token is injected automatically.
{{&lt; /tldr &gt;}}

## RDS / Aurora PostgreSQL

{{&lt; role developer &gt;}}


#### Vouch CLI

`vouch exec` generates the token and injects PostgreSQL environment variables (`PGPASSWORD`, `PGHOST`, `PGPORT`, `PGUSER`, `PGSSLMODE=require`) automatically:

```bash
vouch exec --type rds \
  --rds-hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --rds-username mydbuser \
  -- psql -d mydb
```

Or set them in your current shell:

```bash
eval &#34;$(vouch env --type rds \
  --rds-hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --rds-username mydbuser)&#34;
psql -d mydb
```

To generate just the token (for scripts or non-PostgreSQL clients):

```bash
TOKEN=$(vouch credential rds \
  --hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --username mydbuser)
```

#### AWS CLI

Or use the AWS CLI with Vouch&#39;s `credential_process` integration:

```bash
# Generate an IAM auth token (valid for 15 minutes)
TOKEN=$(aws rds generate-db-auth-token \
  --hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --username mydbuser \
  --profile vouch)

# Connect with psql
PGPASSWORD=&#34;$TOKEN&#34; psql \
  -h mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  -p 5432 \
  -U mydbuser \
  -d mydb \
  &#34;sslmode=require&#34;
```



{{&lt; role admin &gt;}}

**Database setup:** grant the PostgreSQL user the `rds_iam` role:

```sql
GRANT rds_iam TO mydbuser;
```

---

## RDS / Aurora MySQL

{{&lt; role developer &gt;}}

MySQL requires `--enable-cleartext-plugin` because the IAM token is sent as a cleartext password over TLS.


#### Vouch CLI

Generate the token and pass it to the MySQL client:

```bash
TOKEN=$(vouch credential rds \
  --hostname mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  --username mydbuser \
  --port 3306)

mysql -h mydb.cluster-abc123.us-east-1.rds.amazonaws.com \
  -P 3306 \
  -u mydbuser \
  --password=&#34;$TOKEN&#34; \
  --ssl-mode=REQUIRED \
  --enable-cleartext-plugin
```

&gt; **Note:** `vouch exec` and `vouch env --type rds` inject PostgreSQL-style variables; MySQL users should use `vouch credential rds` and pass the token manually.

#### AWS CLI

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
  --password=&#34;$TOKEN&#34; \
  --ssl-mode=REQUIRED \
  --enable-cleartext-plugin
```



{{&lt; role admin &gt;}}

**Database setup:** Create the user with the `AWSAuthenticationPlugin`:

```sql
CREATE USER &#39;mydbuser&#39;@&#39;%&#39; IDENTIFIED WITH AWSAuthenticationPlugin AS &#39;RDS&#39;;
```

---

## Amazon Redshift

{{&lt; role developer &gt;}}

Redshift issues temporary credentials with a configurable lifetime (15--60 minutes); Vouch supports provisioned clusters and Redshift Serverless workgroups.

### Using Vouch CLI (provisioned cluster)

`vouch exec` generates credentials and injects PostgreSQL environment variables (`PGPASSWORD`, `PGUSER`, `PGSSLMODE=require`) automatically:

```bash
vouch exec --type redshift \
  --redshift-cluster-id my-cluster \
  --redshift-db-name mydb \
  -- psql -h my-cluster.abc123.us-east-1.redshift.amazonaws.com -p 5439
```

Or set them in your current shell:

```bash
eval &#34;$(vouch env --type redshift \
  --redshift-cluster-id my-cluster \
  --redshift-db-name mydb)&#34;
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
DB_USER=$(echo &#34;$CREDS&#34; | jq -r &#39;.DbUser&#39;)
DB_PASS=$(echo &#34;$CREDS&#34; | jq -r &#39;.DbPassword&#39;)

PGPASSWORD=&#34;$DB_PASS&#34; psql \
  -h my-cluster.abc123.us-east-1.redshift.amazonaws.com \
  -p 5439 \
  -U &#34;$DB_USER&#34; \
  -d mydb
```

---

## Required IAM permissions

{{&lt; role admin &gt;}}

Your Vouch IAM role needs permission to generate database auth tokens and credentials.

**RDS / Aurora:**

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Action&#34;: &#34;rds-db:connect&#34;,
      &#34;Resource&#34;: &#34;arn:aws:rds-db:us-east-1:123456789012:dbuser:cluster-ABC123/mydbuser&#34;
    }
  ]
}
```

**Redshift (provisioned clusters):**

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Action&#34;: &#34;redshift:GetClusterCredentialsWithIAM&#34;,
      &#34;Resource&#34;: &#34;arn:aws:redshift:us-east-1:123456789012:dbname:my-cluster/*&#34;
    }
  ]
}
```

**Redshift Serverless:**

```json
{
  &#34;Version&#34;: &#34;2012-10-17&#34;,
  &#34;Statement&#34;: [
    {
      &#34;Effect&#34;: &#34;Allow&#34;,
      &#34;Action&#34;: &#34;redshift-serverless:GetCredentials&#34;,
      &#34;Resource&#34;: &#34;arn:aws:redshift-serverless:us-east-1:123456789012:workgroup/*&#34;
    }
  ]
}
```

---

## How it works

1. **`vouch login`** -- The developer authenticates with their YubiKey.
2. **`vouch credential rds`** or **`vouch credential redshift`** -- Vouch generates a short-lived auth token or temporary credentials.
3. **Database client** -- The token is passed as the password to `psql`, `mysql`, or another client; `vouch exec` handles this automatically.

```
vouch login → vouch credential rds|redshift → database client
vouch login → vouch exec --type rds|redshift -- psql
```

---

## Troubleshooting

### &#34;PAM authentication failed for user&#34;

Ensure IAM database authentication is enabled on the RDS instance and the database user has the `rds_iam` role (PostgreSQL) or was created with `AWSAuthenticationPlugin` (MySQL).

### Token expired

RDS/Aurora auth tokens are valid for 15 minutes. Generate a fresh token before connecting. The token is only used to establish the connection -- active sessions are not affected by expiry.

### SSL required error

IAM database authentication requires SSL/TLS. Use `sslmode=require` for PostgreSQL or `--ssl-mode=REQUIRED` for MySQL.
