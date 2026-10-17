<div align="center">
  <img src="./assets/wordmark.svg" alt="Spyglass" width="360">

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
  <a href="./LICENSE">
    <img src="https://img.shields.io/github/license/spyglass-so/spyglass-python" alt="License">
  </a>
  <a href="https://github.com/spyglass-so/spyglass-python/stargazers">
    <img src="https://img.shields.io/github/stars/spyglass-so/spyglass-python?style=social" alt="Stars">
  </a>
  <a href="https://pypi.org/project/spyglass-sdk/">
    <img src="https://img.shields.io/pypi/v/spyglass-sdk" alt="PyPI">
  </a>
  <a href="https://github.com/spyglass-so/spyglass-typescript">
    <img src="https://img.shields.io/badge/TypeScript_SDK-%40spyglass%2Fsdk-3178C6?logo=typescript&logoColor=white" alt="TypeScript SDK">
  </a>
</div>

---

Official Python client for the [Spyglass API](https://docs.spyglass.so) with synchronous and asynchronous (`asyncio`) support. Looking for TypeScript? See [`spyglass-so/spyglass-typescript`](https://github.com/spyglass-so/spyglass-typescript).

## Installation

```bash
pip install spyglass-sdk
# or
uv add spyglass-sdk
```

## Quickstart

Create an API key in [Settings → API](https://app.spyglass.so). Keys start with `sg_live_` and are shown once.

```python
import os
from spyglass import Spyglass

client = Spyglass(token=os.environ["SPYGLASS_API_KEY"])

# 1. Resolve a brand
result = client.search.search_brands_v2(names=["Nike"], limit=1)

if result.brands:
    brand = result.brands[0]

    # 2. Pull the creatives that brand is running
    media_result = client.search.get_media_v2(
        brand_id=brand.id,
        query="running shoes comfort",
        limit=20,
    )

    for ad in media_result.media:
        print(ad.id, ad.hook, ad.landing_page_url)
```

## Async Usage

Use `AsyncSpyglass` for non-blocking calls with `asyncio`:

```python
import asyncio
import os
from spyglass import AsyncSpyglass

client = AsyncSpyglass(token=os.environ["SPYGLASS_API_KEY"])


async def main() -> None:
    result = await client.search.search_brands_v2(names=["Nike"], limit=3)
    for brand in result.brands:
        print(brand.id, brand.name)


asyncio.run(main())
```

## What You Can Query

| Method | Endpoint | What it answers |
| --- | --- | --- |
| `client.search.search_brands_v2(...)` | `POST /brands` | Which brand is this? Who else is in this space? |
| `client.search.creators(...)` | `POST /creators` | Which creator is this? Who matches this audience profile? |
| `client.search.get_account(...)` | `GET /account/{id}` | What do we know about a brand or creator ID you already have? |
| `client.search.get_media_v2(...)` | `POST /media` | What creatives is this account running, or what's out there for this theme? |
| `client.search.taxonomies(...)` | `GET /taxonomies` | Which product-category paths match this phrase? |
| `client.search.get_brand_media_insights(...)` | `GET /account/{id}/insights` | What hooks, USPs, personas, and formats does it lean on? |
| `client.search.get_brand_landing_pages(...)` | `GET /account/{id}/landing-pages` | Where does its ad traffic go? |
| `client.search.get_comments(...)` | `POST /comments` | What are people saying on this Instagram or TikTok post? |
| `client.indexing.index_brand(...)` | `POST /account/index` | Index a brand we don't have yet by website or social URL. |
| `client.indexing.index_brand_status(...)` | `GET /account/{id}/index-status` | Has that brand finished indexing? |
| `client.account.get_credit_usage()` | `GET /team/credit-usage` | How many monthly credits does this team have left? |

Full parameters and response schemas are documented in [`reference.md`](./reference.md) and at [docs.spyglass.so](https://docs.spyglass.so).

## Indexing a New Brand

If Spyglass doesn't have a brand yet, queue it by website or social profile URL and poll until `status` is `"ready"`:

```python
import time

indexed = client.indexing.index_brand(website_url="https://www.nike.com")

status = indexed.status
while status == "indexing":
    time.sleep(3)
    check = client.indexing.index_brand_status(id=indexed.brand_id)
    status = check.status
```

## Error Handling

Non-2xx responses raise an `ApiError` with status code and parsed error body:

```python
from spyglass.core.api_error import ApiError

try:
    client.search.search_brands_v2(names=["Nike"], limit=5)
except ApiError as e:
    print(e.status_code, e.body)
```

## Advanced Options

### Retries & Timeouts

Requests default to a 60-second timeout and up to 2 automatic retries with exponential backoff on `408`, `409`, `429`, and `5xx` responses. Override at the client or per-request level:

```python
client = Spyglass(token="...", timeout=30.0, max_retries=3)

response = client.search.search_brands_v2(
    names=["Nike"],
    limit=5,
    request_options={"timeout_in_seconds": 15, "max_retries": 1},
)
```

### Raw Response & Headers

Use `.with_raw_response` to inspect HTTP headers and status codes:

```python
raw = client.search.with_raw_response.search_brands_v2(names=["Nike"], limit=1)
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
- **Issues:** [GitHub Issues](https://github.com/spyglass-so/spyglass-python/issues)
- **Email:** [support@spyglass.so](mailto:support@spyglass.so)

## License

[MIT](./LICENSE)
