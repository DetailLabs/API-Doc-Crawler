# API Doc Crawler

Crawl any API documentation site and generate a Postman Collection v2.1.

Point it at a docs URL — it discovers all endpoints, scrapes structured data from each page, and outputs a ready-to-import Postman collection with auth headers, path variables, request bodies, and Markdown documentation per request.

Built with [Playwright](https://playwright.dev/python/) for headless browser automation. Handles JavaScript-rendered docs, password-gated sites, OpenAPI/Swagger specs, and sidebar-navigated platforms like ReadMe, GitBook, and Docusaurus.

Available as a **web app** (paste a URL, get a collection) or as **CLI scripts** for full control over each pipeline step.

---

## Web App

The fastest way to use API Doc Crawler. Paste a docs URL in the browser and download a Postman collection — no command line needed.

### Option 1: GitHub Codespaces (no install required)

GitHub Codespaces runs the app in a cloud container directly from this repo. Free tier includes 120 core-hours/month (~60 hours on a 2-core machine).

1. Click **Code** → **Codespaces** → **Create codespace on main**
2. Wait for setup to complete (~2 minutes — installs dependencies and Chromium)
3. Run in the terminal:
   ```bash
   python app.py
   ```
4. A browser tab opens automatically to the app (or click the forwarded port 8000 link)

> Stop your Codespace when done to conserve free hours.

### Option 2: Run locally

Requires Python 3.10+.

```bash
git clone https://github.com/DetailLabs/API-Doc-Crawler.git
cd API-Doc-Crawler
python3 -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
playwright install --with-deps chromium
python app.py
```

Open http://localhost:8000

### Option 3: Docker

```bash
git clone https://github.com/DetailLabs/API-Doc-Crawler.git
cd API-Doc-Crawler
docker build -t api-doc-crawler .
docker run -p 8000:8000 api-doc-crawler
```

Open http://localhost:8000

### Web App Features

- **Live progress** — real-time status updates as endpoints are discovered and scraped
- **Endpoint preview** — table showing method, path, and category for all discovered endpoints
- **One-click download** — download the Postman collection JSON directly from the browser
- **Advanced options** — optional password for gated docs, custom collection name, max endpoint limit

---

## CLI Usage

For full control over each pipeline step, use the scripts directly.

## Pipeline Overview

The project is split into three standalone scripts. Each reads the previous step's output, so you can re-run any step independently without re-crawling.

```
01_download.py          02_categorize.py          03_postman.py
     │                        │                        │
     ▼                        ▼                        ▼
output/endpoints/        output/endpoints.json    output/postman_collection.json
(one JSON per endpoint)  (clean, deduplicated)    (import into Postman)
```

---

## Install

```bash
pip install -r requirements.txt
playwright install --with-deps chromium
```

Requires Python 3.10+. The `--with-deps` flag installs system libraries needed by Chromium on Linux.

---

## Usage

### Public API docs (no password)

```bash
python3 scripts/01_download.py https://petstore.swagger.io -o output
python3 scripts/02_categorize.py -o output
python3 scripts/03_postman.py -o output --name "Petstore API"
```

### Password-protected docs

```bash
python3 scripts/01_download.py https://developers.example.com/reference -p "yourpassword" -o output
python3 scripts/02_categorize.py -o output
python3 scripts/03_postman.py -o output --name "Example API"
```

### Debugging (visible browser)

```bash
python3 scripts/01_download.py https://docs.example.com -p "pass" -o output --no-headless
```

### Import into Postman

1. Open Postman → click **Import**
2. Drag `output/postman_collection.json` into the window
3. Go to collection variables → set `base_url` and `api_key`

---

## Output Structure

```
output/
├── endpoints/                    # Step 1: one JSON per endpoint
│   ├── GET_getvaults.json
│   ├── POST_createtransfer.json
│   ├── DELETE_canceltransfer.json
│   └── ...
├── endpoints.json                # Step 2: clean, categorized, deduplicated
└── postman_collection.json       # Step 3: ready for Postman import
```

---

## Script Details

### `01_download.py` — Endpoint Discovery & Scraping

This script handles authentication, endpoint discovery, and per-page content extraction.

**Authentication** (`authenticate`)

Detects password gates by checking the URL for `/password` or `/login` paths and scanning for `<input type="password">` elements. Fills the password field using multiple CSS selectors (covering ReadMe, GitBook, and generic form layouts), clicks submit, and verifies the gate is cleared.

**Endpoint Discovery** (`discover_endpoints`)

Runs two strategies in priority order:

1. **OpenAPI/Swagger spec** (`try_openapi`, `parse_openapi`) — Checks common spec URLs (`/openapi.json`, `/swagger.json`, `/v2/swagger.json`) and scans page links for spec references. If found, parses the spec directly — extracting method, path, parameters, request body fields (resolving `$ref` schemas via `resolve_schema`), response examples, and tags. When the spec has 3+ complete endpoints, sidebar scraping is skipped entirely since the spec is the authoritative source.

2. **Sidebar navigation** (`discover_sidebar`) — Executes JavaScript in the browser to find links in `nav`, `aside`, `[role="navigation"]`, and `[class*="sidebar"]` elements. Detects HTTP method badges (GET/POST/etc.) next to each link. Filters out non-doc domains, asset files, and duplicates.

After discovery, results are deduplicated (OpenAPI endpoints by `METHOD:path`, sidebar endpoints by URL) and documentation pages like "Getting Started", "Errors", "Rate Limits" are filtered out via a blocklist.

**Page Extraction** (`extract_page`, `EXTRACT_JS`)

For each endpoint that needs scraping (sidebar-discovered), the script navigates to the page and executes JavaScript to extract:

- **Title** — from the `<h1>` element
- **HTTP method** — from method badge elements (`[class*="method"]`, `[class*="badge"]`) or by regex-matching `GET /v2/path` patterns in the page text
- **API path** — from URL/path/endpoint elements or regex extraction
- **Description** — clean `<p>` paragraphs only, skipping elements inside `pre`, `code`, `nav`, `table`, `footer`, and sidebar containers. Deduplicates by tracking seen text. Filters out noise lines: "Updated X ago", "Did this page help", "Too Many Requests", "Internal Server Error", "Log in to see", "Make a request to see", HTTP status labels, and request count lines
- **Permissions** — regex match for "Permissions required: ..." patterns
- **Parameters** — queries `[class*="Param"]` and `[class*="param"]` elements, but only leaf nodes (skips parent containers that hold child params, preventing the duplicate-extraction bug on ReadMe sites). Deduplicates by `name::type` key
- **Response examples** — from `[class*="response"]` containers containing `<pre>` blocks

Each endpoint is saved as an individual JSON file: `output/endpoints/GET_getvaults.json`, `output/endpoints/POST_createtransfer.json`, etc.

```
Options:
  -p, --password TEXT    Password for gated documentation sites
  -o, --output DIR       Output directory (default: output)
  -d, --delay FLOAT      Delay between page requests in seconds (default: 1.5)
  --max INT              Maximum endpoints to crawl (default: 500)
  --no-headless          Run browser in visible mode for debugging
```

---

### `02_categorize.py` — Categorization, Deduplication & Cleanup

Reads all individual endpoint JSON files from `output/endpoints/` and runs four processing passes:

**Pass 1: Clean descriptions** (`clean_descriptions`)

Strips trailing HTTP method names from titles and descriptions (e.g., "Accept a settlement\nPOST" → "Accept a settlement"). Removes duplicate lines from `description_body` by tracking seen text.

**Pass 2: Backfill methods** (`backfill_methods`)

For endpoints missing an HTTP method, infers it from three sources in priority order:
1. Page text — regex matches `GET /v2/path` patterns
2. Slug prefix — maps prefixes like `get` → GET, `create` → POST, `delete` → DELETE, `update` → PUT, `cancel` → DELETE (40+ prefix mappings)
3. Title — same prefix matching against the endpoint title

**Pass 3: Categorize** (`categorize`, `resource_to_category`)

Two-tier categorization:
1. **API path grouping** — splits the path by `/`, strips version prefixes (`v1`, `v2`), and uses the first resource segment as the category. e.g., `/v2/wallets/{id}/addresses` → "Wallets"
2. **Slug-based fallback** — for endpoints without an API path, strips CRUD prefixes from the slug to find the resource name, then maps it through a compound-word dictionary (40+ entries) that handles concatenated terms: `webhookendpoint` → "Webhooks", `kycapplication` → "KYC Onboarding", `cmpackage` → "Collateral Management", `snsettlement` → "Atlas Settlement Network", etc.

OpenAPI-sourced endpoints keep their original tag-based categories.

**Pass 4: Drop non-endpoints & deduplicate** (`deduplicate`)

Removes entries with no API path (category index pages like "Wallets", "Trading" that are navigation pages, not actual endpoints). Then deduplicates by `METHOD:api_path` — when the same endpoint was discovered via both sidebar and content extraction, keeps the entry with more parameters and longer description.

Output: `output/endpoints.json` — a single sorted JSON array.

---

### `03_postman.py` — Postman Collection Generator

Reads `output/endpoints.json` and generates a Postman Collection v2.1 JSON file.

**Auth detection** (`detect_auth_header`)

Scans the first 30 endpoints' page text for auth header patterns. Uses priority-ordered matching — first match wins, only one header is added:
1. `Api-Access-Key` (ReadMe-style APIs)
2. `X-API-Key` (generic)
3. `Authorization: Bearer` (OAuth/JWT)

Returns a single header dict that gets added to every request. No secondary headers (signatures, org keys) are included.

**Collection variables** (`build_variables`)

Sets `base_url` from the OpenAPI spec's `host` + `basePath` fields (Swagger 2.x) or `servers[0].url` (OpenAPI 3.x), falling back to the docs site domain. Adds auth variable (e.g., `api_key`) extracted from the detected header's `{{variable}}` reference.

**Request building** (`build_request`, `classify_params`)

For each endpoint:
- **URL** — constructs `{{base_url}}/v2/path` with `:paramName` path variables. Path segments are split for Postman's URL parser
- **Path variables** — each gets a `(Required) description` pulled from the parameter data
- **Request body** — for POST/PUT/PATCH, builds a JSON body with typed placeholders: strings → `""`, integers → `0`, booleans → `false`, arrays → `[]`, objects → `{}`
- **Parameter classification** (`classify_params`) — routes each parameter to path, query, or body based on its `in` field from the extraction. Falls back to heuristics: if the param name appears in the path template it's a path param; for write methods remaining params go to body; for read methods they go to query. Query params are not included in the output (kept empty per configuration)
- **Headers** — only the single detected auth header

**Markdown descriptions** (`build_description`)

Each request's Docs tab gets a Markdown description matching this format:
```
# Endpoint Title

**Permissions required: Read vault activity**

Description prose from the documentation page, cleaned and deduplicated.
```
Description body is capped at ~600 characters of clean prose.

**Folder structure**

Endpoints are grouped into Postman folders by their category assignment from step 2.

```
Options:
  -o, --output DIR       Output directory (default: output)
  -n, --name TEXT        Collection name (default: inferred from domain)
```

---

## Data Extracted Per Endpoint

Each endpoint JSON file contains:

| Field | Description |
|-------|-------------|
| `method` | HTTP method (GET, POST, PUT, PATCH, DELETE) |
| `api_path` | API path with parameter placeholders, e.g. `/v2/wallets/{walletId}` |
| `title` | Endpoint name from the documentation |
| `description` | Short description (≤200 chars) |
| `description_body` | Full prose from the page — deduplicated, noise-filtered |
| `permissions` | Permission requirement string, e.g. "Permissions required: Transfer funds" |
| `parameters` | Array of `{name, type, required, description, in}` objects |
| `category` | Auto-assigned category (Wallets, Trading, Webhooks, etc.) |
| `response_example` | Sample response JSON if available on the page |
| `api_path` | The API path, e.g. `/v2/transfers/{transferId}` |
| `source` | Discovery method: `"openapi"` or `"sidebar"` |
| `url` | Original documentation page URL |
| `text` | Full raw text content of the page |

---

## Postman Collection Features

- **Folders** — endpoints grouped by category
- **Auth header** — auto-detected, one per request (e.g., `Api-Access-Key: {{api_key}}`)
- **Path variables** — `:paramName` format with `(Required)` descriptions
- **Request bodies** — JSON with typed field placeholders for POST/PUT/PATCH
- **Markdown docs** — title, permissions, and description prose on every request
- **Collection variables** — `base_url` and `api_key` ready to configure
- **Clean params** — no query params pre-populated (empty Params tab)

---

## Supported Platforms

| Platform | Discovery Method |
|----------|-----------------|
| Swagger UI / OpenAPI | Parses spec JSON directly — no page scraping needed |
| ReadMe | Sidebar navigation + per-page extraction |
| GitBook | Sidebar navigation + per-page extraction |
| Docusaurus | Menu link crawling + per-page extraction |
| Redocly | Sidebar navigation + per-page extraction |
| Custom sites | Sidebar/nav element crawling + per-page extraction |

---

## Tips

- **Re-run steps independently** — step 2 and 3 don't need the browser; just re-run them to tweak categorization or collection output without re-crawling
- **Inspect before generating** — check `output/endpoints/` files after step 1 to verify extraction quality, and `output/endpoints.json` after step 2 to verify categories
- **Debug auth issues** — use `--no-headless` to watch the browser navigate and authenticate
- **Rate limiting** — increase `--delay` (default 1.5s) if the target site throttles requests
- **Large APIs** — use `--max` to cap endpoint count for initial testing

---

## License

MIT
