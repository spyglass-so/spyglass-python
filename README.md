<div align="center">
  <img src="https://raw.githubusercontent.com/spyglass-so/spyglass-python/main/assets/wordmark.svg" alt="Spyglass" width="360">

  <h3>The API layer for the social internet</h3>
  <p>Official Python SDK for the Spyglass API — search brands, creators, and the ads they run.</p>

  <a href="https://app.spyglass.so">
    <img src="https://img.shields.io/badge/Get_Started-111111?style=for-the-badge" alt="Get Started">
  </a>
  <a href="https://docs.spyglass.so">
    <img src="https://img.shields.io/badge/Documentation-4A4A4A?style=for-the-badge" alt="Documentation">
  </a>
  <a href="https://spyglass.so">
    <img src="https://img.shields.io/badge/spyglass.so-0A0A0A?style=for-the-badge" alt="spyglass.so">
  </a>
</div>

<br>

<div align="center">
  <a href="https://github.com/spyglass-so/spyglass-python/blob/main/LICENSE">
    <img src="https://img.shields.io/github/license/spyglass-so/spyglass-python" alt="License">
  </a>
  <a href="https://github.com/spyglass-so/spyglass-python/stargazers">
    <img src="https://img.shields.io/github/stars/spyglass-so/spyglass-python?style=social" alt="Stars">
  </a>
  <a href="https://pypi.org/project/spyglass-sdk/">
    <img src="https://img.shields.io/pypi/v/spyglass-sdk" alt="PyPI">
  </a>
  <a href="https://github.com/spyglass-so/spyglass-typescript">
    <img src="https://img.shields.io/badge/TypeScript_SDK-%40spyglasshq%2Fsdk-3178C6?logo=typescript&logoColor=white" alt="TypeScript SDK">
  </a>
</div>

---

**`spyglass-sdk`** is the official Python client for the [Spyglass API](https://docs.spyglass.so) with both synchronous (`Spyglass`) and asynchronous (`AsyncSpyglass`) clients. Resolve brands and creators, search across millions of Meta, Instagram, and TikTok creatives with natural-language briefs or structured filters, and extract hooks, personas, landing pages, and public comments.

> Looking for TypeScript / Node.js? See [`spyglass-so/spyglass-typescript`](https://github.com/spyglass-so/spyglass-typescript) (`npm install @spyglasshq/sdk`).

## Installation

```bash
pip install spyglass-sdk
# or
uv add spyglass-sdk
```

## Quickstart

Create an API key in **[Settings → API](https://app.spyglass.so)**. Keys start with `sg_live_` and are shown once.

```python
import os
from spyglass import Spyglass

client = Spyglass(token=os.environ["SPYGLASS_API_KEY"])

# 1. Resolve a brand
result = client.search_brands(names=["Nike"], limit=1)

if result.brands:
    brand = result.brands[0]

    # 2. Search the creatives that brand is running
    media_result = client.search_media(
        brand_id=brand.id,
        query="marathon running shoes comfort",
        limit=10,
    )

    for ad in media_result.media:
        print(ad.id, ad.hook, ad.core_angle, ad.url)
```

## Async Usage

Use `AsyncSpyglass` for non-blocking calls with `asyncio`:

```python
import asyncio
import os
from spyglass import AsyncSpyglass

client = AsyncSpyglass(token=os.environ["SPYGLASS_API_KEY"])


async def main() -> None:
    result = await client.search_brands(names=["Nike"], limit=3)
    for brand in result.brands:
        print(brand.id, brand.name)


asyncio.run(main())
```

## API Reference Overview

| SDK Method | HTTP Endpoint | Cost | Rate Limit |
| --- | --- | --- | --- |
| `client.search_brands(...)` | `POST /brands` | $0.0020 / brand returned | 120 / min |
| `client.search_creators(...)` | `POST /creators` | $0.0020 / creator returned | 120 / min |
| `client.search_media(...)` | `POST /media` | $0.0010 / creative returned | 120 / min |
| `client.search_taxonomies(query=...)` | `GET /taxonomies` | Free | 120 / min |
| `client.get_comments(...)` | `POST /comments` | $0.0100 / live page fetched | 20 / min |
| `client.account.get(id=...)` | `GET /account/{id}` | $0.0020 / account returned | 120 / min |
| `client.account.insights(...)` | `GET /account/{id}/insights` | $0.0010 / cluster returned | 120 / min |
| `client.account.landing_pages(...)` | `GET /account/{id}/landing-pages` | $0.0050 / landing page returned | 120 / min |
| `client.account.index(...)` | `POST /account/index` | Free | 50 / min |
| `client.account.index_status(id=...)` | `GET /account/{id}/index-status` | Free | 300 / min |
| `client.get_credit_usage()` | `GET /team/credit-usage` | Free | 120 / min |

Full parameters and response schemas are documented in [`reference.md`](https://github.com/spyglass-so/spyglass-python/blob/main/reference.md) and at [docs.spyglass.so](https://docs.spyglass.so).

## Indexing a New Account

If Spyglass doesn't have a brand or creator yet, queue it by website or social profile URL (`POST /account/index` is free) and poll until `status` is `"ready"`:

```python
import time

indexed = client.account.index(website_url="https://www.on.com")

status = indexed.status
while status == "indexing":
    time.sleep(3)
    check = client.account.index_status(id=indexed.brand_id)
    status = check.status

if status in ("ready", "already_exists"):
    account = client.account.get(id=indexed.brand_id)
    print("Indexed account:", account.name)
```

## Error Handling

Non-2xx responses raise an `ApiError` with status code and parsed error body:

```python
from spyglass.core.api_error import ApiError

try:
    client.search_brands(names=["Nike"], limit=5)
except ApiError as e:
    print(e.status_code, e.body)
```

## Advanced Options

### Retries & Timeouts

Requests default to a 60-second timeout and up to 2 automatic retries with exponential backoff on `408`, `409`, `429`, and `5xx` responses. Override at the client or per-request level:

```python
client = Spyglass(token="...", timeout=30.0, max_retries=3)

response = client.search_brands(
    names=["Nike"],
    limit=5,
    request_options={"timeout_in_seconds": 15, "max_retries": 1},
)
```

### Raw Response & Headers

Use `.with_raw_response` to inspect HTTP headers and status codes:

```python
raw = client.with_raw_response.search_brands(names=["Nike"], limit=1)
print(raw.status_code)
print(raw.headers)
print(raw.data.brands)
```

### Custom HTTPX Client

Pass a custom `httpx.Client` or `httpx.AsyncClient` for custom proxies or transport settings:

```python
import httpx
from spyglass import Spyglass

client = Spyglass(
    token="...",
    httpx_client=httpx.Client(proxy="http://proxy.example.com"),
)
```

## Billing & Conventions

- **Credits:** Most calls spend monthly credits per row returned (`count` in the response). Always bound requests with `limit`. See [Billing docs](https://docs.spyglass.so/documentation/get-started/billing).
- **Pagination:** Page through results with `offset` and `limit`. `count` is the number of rows in the current response.
- **Server-to-server:** Call the API from your backend, data pipeline, or agent runtime.

## Support

- **Documentation:** [docs.spyglass.so](https://docs.spyglass.so)
- **Issues:** [github.com/spyglass-so/spyglass-python/issues](https://github.com/spyglass-so/spyglass-python/issues)
- **Email:** [support@spyglass.so](mailto:support@spyglass.so)

## License

[MIT](https://github.com/spyglass-so/spyglass-python/blob/main/LICENSE)
