# Authentication

Infini merchant APIs use HMAC-SHA256 request signing.

## Required Headers

All authenticated merchant requests include:

| Header | Required | Notes |
| --- | --- | --- |
| `Date` | Yes | RFC 1123 GMT string. Client time must be within +/- 300 seconds of server time. |
| `Authorization` | Yes | `Signature keyId="...",algorithm="hmac-sha256",headers="@request-target date",signature="..."` |
| `Digest` | Body requests only | `SHA-256=<base64 sha256 body>` |
| `Content-Type` | JSON body requests | `application/json` |

The body digest is not part of the signing string.

## Signing String

Use the exact method and request path, including query string if present:

```text
{keyId}
{METHOD} {path}
date: {GMT_time}
```

The string has a trailing newline after the `date` line.

Example:

```text
merchant-001
GET /v1/acquiring/order?order_id=ord_123
date: Tue, 21 Jan 2025 12:00:00 GMT
```

Compute:

```text
signature = Base64(HMAC-SHA256(secret_key, signing_string))
digest = Base64(SHA256(raw_body_bytes))
```

## Python Client

```python
from __future__ import annotations

import base64
import hashlib
import hmac
import json
from datetime import datetime, timezone
from email.utils import format_datetime
from typing import Any

import requests


class InfiniClient:
    def __init__(self, key_id: str, secret_key: str, base_url: str) -> None:
        self.key_id = key_id
        self.secret_key = secret_key.encode("utf-8")
        self.base_url = base_url.rstrip("/")

    def _date(self) -> str:
        return format_datetime(datetime.now(timezone.utc), usegmt=True)

    def _headers(self, method: str, path: str, body: bytes | None) -> dict[str, str]:
        gmt = self._date()
        signing_string = f"{self.key_id}\n{method.upper()} {path}\ndate: {gmt}\n"
        signature = base64.b64encode(
            hmac.new(self.secret_key, signing_string.encode("utf-8"), hashlib.sha256).digest()
        ).decode("ascii")
        headers = {
            "Date": gmt,
            "Authorization": (
                f'Signature keyId="{self.key_id}",algorithm="hmac-sha256",'
                f'headers="@request-target date",signature="{signature}"'
            ),
        }
        if body is not None:
            digest = base64.b64encode(hashlib.sha256(body).digest()).decode("ascii")
            headers["Digest"] = f"SHA-256={digest}"
            headers["Content-Type"] = "application/json"
        return headers

    def request(self, method: str, path: str, payload: dict[str, Any] | None = None) -> dict[str, Any]:
        body = None
        if payload is not None:
            body = json.dumps(payload, separators=(",", ":"), ensure_ascii=False).encode("utf-8")
        response = requests.request(
            method.upper(),
            f"{self.base_url}{path}",
            data=body,
            headers=self._headers(method, path, body),
            timeout=30,
        )
        response.raise_for_status()
        data = response.json()
        if data.get("code") not in (0, None):
            raise RuntimeError(f"Infini API error {data.get('code')}: {data.get('message')}")
        return data
```

## Node.js Client

```javascript
const crypto = require("crypto");

class InfiniClient {
  constructor({ keyId, secretKey, baseUrl }) {
    this.keyId = keyId;
    this.secretKey = secretKey;
    this.baseUrl = baseUrl.replace(/\/$/, "");
  }

  _headers(method, path, body) {
    const date = new Date().toUTCString();
    const signingString = `${this.keyId}\n${method.toUpperCase()} ${path}\ndate: ${date}\n`;
    const signature = crypto
      .createHmac("sha256", this.secretKey)
      .update(signingString, "utf8")
      .digest("base64");

    const headers = {
      Date: date,
      Authorization:
        `Signature keyId="${this.keyId}",algorithm="hmac-sha256",` +
        `headers="@request-target date",signature="${signature}"`,
    };

    if (body !== null) {
      headers["Digest"] =
        "SHA-256=" + crypto.createHash("sha256").update(body, "utf8").digest("base64");
      headers["Content-Type"] = "application/json";
    }
    return headers;
  }

  async request(method, path, payload = null) {
    const body = payload === null ? null : JSON.stringify(payload);
    const response = await fetch(`${this.baseUrl}${path}`, {
      method: method.toUpperCase(),
      headers: this._headers(method, path, body),
      body,
    });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    const data = await response.json();
    if (data.code !== 0 && data.code !== undefined) {
      throw new Error(`Infini API error ${data.code}: ${data.message || ""}`);
    }
    return data;
  }
}
```

## Common Signature Bugs

- Signing `/v1/acquiring/order` but sending `/v1/acquiring/order?order_id=...`.
- Omitting the trailing newline in the signing string.
- Including `Digest` in the signing string.
- Calculating `Digest` over pretty JSON but sending compact JSON, or the reverse.
- Using local timezone text instead of GMT.
- Calling from frontend code and exposing the secret.
