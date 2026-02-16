---
title: "Device Authorization (CLI)"
description: "Integrate Vouch authentication into native desktop applications and CLI tools using the Device Authorization Grant."
weight: 13
params:
  category: "native"
  language: "CLI"
---

> See the [Applications overview]({{< ref "/docs/applications" >}}) for prerequisites, configuration endpoints, and available scopes.

For native desktop applications and CLI tools that cannot open a browser redirect, use the **Device Authorization Grant** ([RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628)). This flow displays a URL and code that the user enters in a browser on any device.

## Device Authorization Flow

1. Your application requests a device code from the Vouch server.
2. The server returns a `device_code`, a `user_code`, and a `verification_uri`.
3. Your application displays the `user_code` and `verification_uri` to the user.
4. The user opens `verification_uri` in a browser, enters the code, and authenticates with their YubiKey.
5. Your application polls the token endpoint until the user completes authentication.
6. Once approved, the token endpoint returns an access token and ID token.

## Python

```python
import os
import time
import requests

VOUCH_URL = "{{< instance-url >}}"
CLIENT_ID = os.environ["VOUCH_CLIENT_ID"]

# Step 1: Request device code
device_response = requests.post(
    f"{VOUCH_URL}/oauth/device/code",
    data={
        "client_id": CLIENT_ID,
        "scope": "openid email",
    },
)
device_data = device_response.json()

device_code = device_data["device_code"]
user_code = device_data["user_code"]
verification_uri = device_data["verification_uri"]
interval = device_data.get("interval", 5)
expires_in = device_data.get("expires_in", 600)

# Step 2: Display instructions to the user
print(f"\nTo sign in, open this URL in your browser:\n")
print(f"  {verification_uri}\n")
print(f"And enter the code: {user_code}\n")
print("Waiting for authentication...")

# Step 3: Poll for the token
deadline = time.time() + expires_in
while time.time() < deadline:
    time.sleep(interval)

    token_response = requests.post(
        f"{VOUCH_URL}/oauth/token",
        data={
            "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
            "device_code": device_code,
            "client_id": CLIENT_ID,
        },
    )

    if token_response.status_code == 200:
        tokens = token_response.json()
        print(f"\nAuthenticated successfully!")
        print(f"Access token: {tokens['access_token'][:20]}...")

        # Fetch user info
        userinfo = requests.get(
            f"{VOUCH_URL}/oauth/userinfo",
            headers={"Authorization": f"Bearer {tokens['access_token']}"},
        ).json()
        print(f"Hello, {userinfo.get('name', 'user')}!")
        break

    error_data = token_response.json()
    error = error_data.get("error")

    if error == "authorization_pending":
        continue
    elif error == "slow_down":
        interval += 5
        continue
    elif error == "expired_token":
        print("The device code has expired. Please try again.")
        break
    elif error == "access_denied":
        print("Authentication was denied.")
        break
    else:
        print(f"Unexpected error: {error}")
        break
else:
    print("Timed out waiting for authentication.")
```

## Node.js

```javascript
const https = require("https");
const querystring = require("querystring");

const VOUCH_URL = "{{< instance-url >}}";
const CLIENT_ID = process.env.VOUCH_CLIENT_ID;

function post(path, data) {
  return new Promise((resolve, reject) => {
    const url = new URL(path, VOUCH_URL);
    const body = querystring.stringify(data);
    const req = https.request(
      url,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
          "Content-Length": Buffer.byteLength(body),
        },
      },
      (res) => {
        let data = "";
        res.on("data", (chunk) => (data += chunk));
        res.on("end", () =>
          resolve({ status: res.statusCode, body: JSON.parse(data) })
        );
      }
    );
    req.on("error", reject);
    req.write(body);
    req.end();
  });
}

function get(path, token) {
  return new Promise((resolve, reject) => {
    const url = new URL(path, VOUCH_URL);
    const req = https.request(
      url,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${token}` },
      },
      (res) => {
        let data = "";
        res.on("data", (chunk) => (data += chunk));
        res.on("end", () => resolve(JSON.parse(data)));
      }
    );
    req.on("error", reject);
    req.end();
  });
}

async function main() {
  // Step 1: Request device code
  const deviceResponse = await post("/oauth/device/code", {
    client_id: CLIENT_ID,
    scope: "openid email",
  });

  const { device_code, user_code, verification_uri, interval = 5, expires_in = 600 } =
    deviceResponse.body;

  // Step 2: Display instructions
  console.log(`\nTo sign in, open this URL in your browser:\n`);
  console.log(`  ${verification_uri}\n`);
  console.log(`And enter the code: ${user_code}\n`);
  console.log("Waiting for authentication...");

  // Step 3: Poll for the token
  let pollInterval = interval;
  const deadline = Date.now() + expires_in * 1000;

  while (Date.now() < deadline) {
    await new Promise((r) => setTimeout(r, pollInterval * 1000));

    const tokenResponse = await post("/oauth/token", {
      grant_type: "urn:ietf:params:oauth:grant-type:device_code",
      device_code,
      client_id: CLIENT_ID,
    });

    if (tokenResponse.status === 200) {
      console.log("\nAuthenticated successfully!");

      const userinfo = await get("/oauth/userinfo", tokenResponse.body.access_token);
      console.log(`Hello, ${userinfo.name || "user"}!`);
      return;
    }

    const error = tokenResponse.body.error;

    if (error === "authorization_pending") continue;
    if (error === "slow_down") {
      pollInterval += 5;
      continue;
    }
    if (error === "expired_token") {
      console.log("The device code has expired. Please try again.");
      return;
    }
    if (error === "access_denied") {
      console.log("Authentication was denied.");
      return;
    }

    console.log(`Unexpected error: ${error}`);
    return;
  }

  console.log("Timed out waiting for authentication.");
}

main().catch(console.error);
```

## Rust

Add the required dependencies to your `Cargo.toml`:

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

Implement the device authorization flow:

```rust
use serde::Deserialize;
use std::env;
use std::time::{Duration, Instant};

#[derive(Deserialize)]
struct DeviceCodeResponse {
    device_code: String,
    user_code: String,
    verification_uri: String,
    #[serde(default = "default_interval")]
    interval: u64,
    #[serde(default = "default_expires_in")]
    expires_in: u64,
}

fn default_interval() -> u64 { 5 }
fn default_expires_in() -> u64 { 600 }

#[derive(Deserialize)]
struct TokenResponse {
    access_token: String,
}

#[derive(Deserialize)]
struct ErrorResponse {
    error: String,
}

#[derive(Deserialize)]
struct UserInfo {
    name: Option<String>,
    email: Option<String>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let vouch_url = "{{< instance-url >}}";
    let client_id = env::var("VOUCH_CLIENT_ID")
        .expect("VOUCH_CLIENT_ID environment variable not set");

    let client = reqwest::Client::new();

    // Step 1: Request device code
    let device_response: DeviceCodeResponse = client
        .post(format!("{vouch_url}/oauth/device/code"))
        .form(&[
            ("client_id", client_id.as_str()),
            ("scope", "openid email"),
        ])
        .send()
        .await?
        .json()
        .await?;

    // Step 2: Display instructions
    println!("\nTo sign in, open this URL in your browser:\n");
    println!("  {}\n", device_response.verification_uri);
    println!("And enter the code: {}\n", device_response.user_code);
    println!("Waiting for authentication...");

    // Step 3: Poll for the token
    let mut interval = Duration::from_secs(device_response.interval);
    let deadline = Instant::now() + Duration::from_secs(device_response.expires_in);

    loop {
        tokio::time::sleep(interval).await;

        if Instant::now() > deadline {
            println!("Timed out waiting for authentication.");
            break;
        }

        let response = client
            .post(format!("{vouch_url}/oauth/token"))
            .form(&[
                ("grant_type", "urn:ietf:params:oauth:grant-type:device_code"),
                ("device_code", &device_response.device_code),
                ("client_id", &client_id),
            ])
            .send()
            .await?;

        if response.status().is_success() {
            let tokens: TokenResponse = response.json().await?;
            println!("\nAuthenticated successfully!");

            let userinfo: UserInfo = client
                .get(format!("{vouch_url}/oauth/userinfo"))
                .bearer_auth(&tokens.access_token)
                .send()
                .await?
                .json()
                .await?;

            println!(
                "Hello, {}!",
                userinfo.name.unwrap_or_else(|| "user".to_string())
            );
            break;
        }

        let error: ErrorResponse = response.json().await?;
        match error.error.as_str() {
            "authorization_pending" => continue,
            "slow_down" => {
                interval += Duration::from_secs(5);
                continue;
            }
            "expired_token" => {
                println!("The device code has expired. Please try again.");
                break;
            }
            "access_denied" => {
                println!("Authentication was denied.");
                break;
            }
            other => {
                println!("Unexpected error: {other}");
                break;
            }
        }
    }

    Ok(())
}
```

---

## Device Authorization Response

When your application requests a device code, the server returns the following fields:

| Field | Type | Description |
|---|---|---|
| `device_code` | string | The device verification code. Used by your application when polling the token endpoint. Do not display this to the user. |
| `user_code` | string | The end-user verification code. Display this to the user so they can enter it in their browser. |
| `verification_uri` | string | The URL the user should visit to enter the code and authenticate. |
| `verification_uri_complete` | string | Optional. A URL that includes the `user_code`, so the user can navigate directly without manually entering the code. |
| `expires_in` | number | The lifetime of the `device_code` and `user_code` in seconds. |
| `interval` | number | The minimum number of seconds your application should wait between polling requests. |

---

## Polling Errors

When polling the token endpoint during the device authorization flow, the server may return the following error codes:

| Error | Description | Action |
|---|---|---|
| `authorization_pending` | The user has not yet completed authentication. | Continue polling at the specified interval. |
| `slow_down` | Your application is polling too frequently. | Increase the polling interval by 5 seconds and continue. |
| `expired_token` | The `device_code` has expired. | Stop polling. Restart the flow by requesting a new device code. |
| `access_denied` | The user denied the authorization request. | Stop polling. Inform the user that authentication was denied. |
