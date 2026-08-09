# mAItion

![mAItion](https://github.com/WikiTeq/mAItion/blob/main/mAItion.png?raw=true)

mAItion is an all-in-one ready-to-use AI-powered tool that combines your existing knowledge with LLMs,
allowing you to chat, search and interact with your data through a slick chat interface. With mAItion
you can aggregate all your knowledge from many sources using Connectors into a central place and
interact with your knowledge with ease!

📚 **Documentation:** [docs.maition.com](https://docs.maition.com/)

## ✨ Features

* External Ollama at `http://100.85.191.78:11434`, with the installed `qwen2.5:14b` model as the default for chat and inference
* Support for local and remote models for embeddings and inference, including Ollama, OpenRouter, and OpenAI-compatible APIs
* Asynchronous data ingestion with deduplication and per-source configurable schedules
* Data ingestion from a local directory by default, with optional S3 bucket ingestion and Everything-to-Markdown conversion via [MarkItDown](https://github.com/microsoft/markitdown)
* Data ingestion from MediaWiki with Wiki-to-Markdown conversion via [html2text](https://github.com/Alir3z4/html2text)
* SerpAPI search query results ingestion from Google Search results with customizable queries
* Flexible configuration supporting an arbitrary number of connectors
* Built with extensibility in mind, allowing for custom connectors addition with ease
* MCP servers support (stdio, streamable http)
* Web-search support (through external services and via on-premise services)
* In-place chat with uploaded documents and images (for multi-modal LLMs)
* Code execution and Code interpreter
* Text-to-Speech and Speech-to-Text capabilities
* Image generator (requires model supporting image generation)
* Flexible automation capabilities through Functions and Pipelines
* Multi-user setup with fine-grained groups and permissions
* Support for multiple customized configurations of LLM models

## ✨ Use-cases

* A single place to chat with your company knowledge that's scattered across many external systems
* A central space for looking up and refining your existing knowledge across many knowledge bases
* A tool to find secret knowledge that cannot be found any other way across your scattered data
* An entry-point into your on-premise hosted LLM models supporting evaluations and per-model settings

## 🌐 Connectors included

* Directory (default; a host directory mounted read-only into the ingestion worker)
* S3 (optional; any AWS-compatible Object Storage including AWS, Contabo, B2, Cloudflare R2, OVH, etc.)
* MediaWiki (all versions supported, both private and public wiki)
* SerpAPI
* Web
* Jira
* Pipedrive
* Slack
* IMAP

## 🌐 Extra connectors

Over 100 extra connectors are available at request, including the most popular ones:

* Gmail
* Google Drive
* Jira
* GitHub
* Gitlab
* Notion
* Microsoft Teams
* Microsoft Office 365
* Dropbox
* Trello
* Web scraper
* YouTube
* FTP
* SCP
* SSH
* and many more..

## Quick start

### Requirements

* Docker and Docker Compose
* An Ollama server reachable from the Docker host and containers at `http://100.85.191.78:11434`
* `qwen2.5:14b` already installed on that Ollama server
* An internet connection on first start to download the container images and local HuggingFace embedding model

An OpenRouter/OpenAI API key and an S3 bucket are no longer required for the default setup.

### Setup

Copy the example configuration files:

```bash
cp .env.rag.example .env.rag
cp .env.openwebui.example .env
cp config.yaml.example config.yaml
```

The copied files are ready to use without cloud credentials. Their defaults are:

* `./data/input` on the host is mounted read-only at `/data/input` in the ingestion worker.
* The Directory connector recursively ingests `/data/input` once per hour.
* `sentence-transformers/all-mpnet-base-v2` runs locally for embeddings.
* OpenWebUI and the RAG backend connect to the external Ollama server at `http://100.85.191.78:11434`.
* `qwen2.5:14b`, selected from the server's installed models, is used for both the RAG rephrase endpoint and OpenWebUI chat.

Place documents in `./data/input`, then build and start the stack:

```bash
docker compose up -d --build --remove-orphans
```

`--remove-orphans` removes any old `ollama` and `ollama-init` containers left by
an earlier local-Ollama deployment. Its old `ollama_data` volume is harmless and
is not removed automatically.

Check the service states with:

```bash
docker compose ps -a
```

The long-running services should become healthy. If the API or OpenWebUI cannot reach
Ollama, verify the external server is listening on a network-accessible interface and
allows connections from the Docker host.

Once all the services are booted and report healthy status visit http://localhost:3000 and
login using Admin credentials. The credentials are defined in `X_WEBUI_ADMIN_EMAIL` and `X_WEBUI_ADMIN_PASS`
of the `.env` file. The default ones are:

* username: `admin@example123.com`
* password: `q1w2e3r4!`

On a new installation, OpenWebUI automatically configures the external Ollama connection at
`http://100.85.191.78:11434` and selects `qwen2.5:14b` as the global default chat model.
OpenWebUI binds to `127.0.0.1` by default. Change the bootstrap credentials and
`WEBUI_SECRET_KEY` before setting `WEBUI_BIND_HOST=0.0.0.0` for trusted-network access.

Two components handle RAG service communication:

- **Filter function** (`functions/function.py`) — intercepts every user message and injects ROAT context automatically. Enabled globally via Admin Panel → Functions.
- **Knowledge Base Search tool** (`tools/roat_retrieval.py`) — a Workspace Tool that lets the LLM decide when to query ROAT. Requires a model with native function calling support. Both are automatically provisioned on first boot.
- **Web Search Tool** (`tools/web_search.py`) — an optional Workspace Tool that lets the LLM search the web via the Tavily API. Enable with `TOOL_WEB_SEARCH_ENABLED=True` in `.env`, then either set `TOOL_WEB_SEARCH_API_KEY` to auto-configure the key on install, or set the `tavily_api_key` valve manually from Workspace → Tools.

## Connectors configuration

The service supports multiple data sources, including multiple data sources of the same type, each with its own
ingestion schedule. Connectors are enabled in `config.yaml`; connector runtime values and secrets are normally
defined in `.env.rag`. The host path mounted for the Directory connector is defined in `.env`.

### Directory Connector (default)

The Directory connector ingests files from a local host directory. By default, Compose mounts `./data/input`
read-only into the ingestion worker at `/data/input`, and the connector scans that container path recursively.

```yaml
# config.yaml

sources:
  - type: "directory"
    name: "local_docs"
    config:
      path: "${DIRECTORY1_PATH}"
      recursive: true
      exclude_hidden: true
      exclude_empty: false
      schedules: "${DIRECTORY1_SCHEDULES}"
```

```dotenv
# .env (host-to-container mount)
DIRECTORY_HOST_PATH=./data/input

# .env.rag (path visible inside the ingestion worker)
DIRECTORY1_PATH=/data/input
DIRECTORY1_SCHEDULES=3600
```

Change `DIRECTORY_HOST_PATH` to ingest another host directory. Keep `DIRECTORY1_PATH=/data/input` unless you
also change the container mount target in `compose.yaml`.

### S3 Connector (optional)

The S3 connector ingests documents from S3 buckets and converts them to Markdown format.
The connector has the following configuration options:

```yaml
# config.yaml

sources:
  - type: "s3" # must be s3
    name: "account1" # arbitrary name for the connector, will be stored in metadata
    config:
      endpoint: "${S3_ACCOUNT1_ENDPOINT}" # s3 endpoint
      access_key: "${S3_ACCOUNT1_ACCESS_KEY}" # s3 access key
      secret_key: "${S3_ACCOUNT1_SECRET_KEY}" # s3 secret key
      region: "${S3_ACCOUNT1_REGION}" # s3 region
      use_ssl: "${S3_ACCOUNT1_USE_SSL}" # use ssl for s3 connection, can be True or False
      buckets: "${S3_ACCOUNT1_BUCKETS}" # single entry or comma-separated list i.e. bucket1,bucket2
      schedules: "${S3_ACCOUNT1_SCHEDULES}" # single entry or comma-separated list i.e. 3600,60

  - type: "s3"
    name: "account2"
    config:
      ...

  - type: "s3"
    name: "account3"
    config:
      ...
```

```dotenv
# .env.rag

S3_ACCOUNT1_ENDPOINT=https://s3.amazonaws.com
S3_ACCOUNT1_ACCESS_KEY=xxx
S3_ACCOUNT1_SECRET_KEY=xxx
S3_ACCOUNT1_REGION=us-east-1
S3_ACCOUNT1_USE_SSL=True
S3_ACCOUNT1_BUCKETS=bucket1,bucket2
S3_ACCOUNT1_SCHEDULES=3600,60
```

### MediaWiki Connector

The MediaWiki connector ingests documents from MediaWiki sites and converts them to Markdown format.
The connector has the following configuration options:

```yaml
# config.yaml

sources:
  - type: "mediawiki"
    name: "wiki1"
    config:
      host: "${MEDIAWIKI1_HOST}"
      path: "/w/"          # optional, default /w/
      scheme: "https"      # optional, default https
      page_limit: 500      # optional, max pages per namespace (default: unlimited)
      namespaces: "0,1"    # optional, comma-separated namespace IDs (default: content namespaces)
      filter_redirects: true  # optional, exclude redirect pages (default: true)
      username: "${MEDIAWIKI1_USERNAME}"  # optional, for private wikis
      password: "${MEDIAWIKI1_PASSWORD}"  # optional, for private wikis
      schedules: "${MEDIAWIKI1_SCHEDULES}"

  - type: "mediawiki"
    name: "wiki2"
    config:
      host: "${MEDIAWIKI2_HOST}"
      schedules: "${MEDIAWIKI2_SCHEDULES}"
```

```dotenv
# .env.rag

MEDIAWIKI1_HOST=wiki.example.org
MEDIAWIKI1_SCHEDULES=3600
# Only needed for private wikis requiring login:
#MEDIAWIKI1_USERNAME=your-bot-username
#MEDIAWIKI1_PASSWORD=your-bot-password
```

### SerpAPI Connector

The SerpAPI connector ingests documents from Google Search results and converts them to Markdown format.
The connector has the following configuration options:

```yaml
# config.yaml

sources:
  - type: "serpapi"
    name: "serp_ingestion1"
    config:
      api_key: "${SERPAPI1_KEY}"
      queries: "${SERPAPI1_QUERIES}"
      schedules: "${SERPAPI1_SCHEDULES}"

  - type: "serpapi"
    name: "serp_ingestion2"
    config:

  - type: "serpapi"
    name: "serp_ingestion3"
    config:
```

```dotenv
# .env.rag

SERPAPI1_KEY=xxxx
SERPAPI1_QUERIES=aaa
SERPAPI1_SCHEDULES=3600
```

### Web Connector

The Web connector ingests content from web pages. It supports two mutually exclusive modes:
- **URLs mode**: scrapes a fixed list of URLs
- **Sitemap mode**: discovers URLs from a `sitemap.xml` with optional `include_prefix` filter

```yaml
# config.yaml

sources:
  # URLs mode
  - type: "web"
    name: "web1"
    config:
      urls: "${WEB1_URLS}"
      html_to_text: true
      schedules: "${WEB1_SCHEDULES}"

  # Sitemap mode
  - type: "web"
    name: "web2"
    config:
      sitemap_url: "${WEB2_SITEMAP_URL}"
      include_prefix: "${WEB2_INCLUDE_PREFIX}"
      html_to_text: true
      schedules: "${WEB2_SCHEDULES}"
```

```dotenv
# .env.rag

WEB1_URLS=https://example.com/page1,https://example.com/page2
WEB1_SCHEDULES=60
WEB2_SITEMAP_URL=https://example.com/sitemap.xml
WEB2_INCLUDE_PREFIX=/blog/
WEB2_SCHEDULES=60
```

### Jira Connector

The Jira connector ingests issues from Jira Cloud or Jira Server/Data Center.
It supports two authentication modes: `basic` (email + API token) and `token` (Personal Access Token / PAT).

```yaml
# config.yaml

sources:
  - type: "jira"
    name: "jira1"
    config:
      server_url: "${JIRA1_SERVER_URL}"
      auth_type: "basic"        # "basic" (email + API token) or "token" (PAT)
      email: "${JIRA1_EMAIL}"   # required when auth_type is "basic"
      api_token: "${JIRA1_API_TOKEN}"
      jql: "${JIRA1_JQL}"       # JQL query to select issues
      max_results: 50
      load_comments: false
      max_comments: 10
      schedules: "${JIRA1_SCHEDULES}"
```

```dotenv
# .env.rag

JIRA1_SERVER_URL=https://your-org.atlassian.net
JIRA1_EMAIL=your-email@example.com
JIRA1_API_TOKEN=your-api-token
JIRA1_JQL=project = MYPROJECT ORDER BY updated DESC
JIRA1_SCHEDULES=3600
```

### Pipedrive Connector

The Pipedrive connector ingests CRM records from Pipedrive via the REST API v1. Supports activities, deals, notes, organizations, persons, products, projects, leads, tasks, and mails.

```yaml
# config.yaml

sources:
  - type: "pipedrive"
    name: "pipedrive1"
    config:
      api_token: "${PIPEDRIVE1_API_TOKEN}"
      load_types:           # optional, default: all entity types
        - deals
        - notes
        - persons
        - mails
      max_items: 500        # optional, per-entity limit (default: unlimited)
      max_retries: 3        # optional, retry attempts on failure (default: 3)
      filter_mail_folders:  # optional, default: [inbox]
        - inbox
        - sent
      schedules: "${PIPEDRIVE1_SCHEDULES}"

  - type: "pipedrive"
    name: "pipedrive2"
    config:
      api_token: "${PIPEDRIVE2_API_TOKEN}"
      schedules: "${PIPEDRIVE2_SCHEDULES}"
```

```dotenv
# .env.rag

PIPEDRIVE1_API_TOKEN=your-pipedrive-api-token
PIPEDRIVE1_SCHEDULES=3600
PIPEDRIVE2_API_TOKEN=your-second-pipedrive-api-token
PIPEDRIVE2_SCHEDULES=3600
```

### Slack Connector

The Slack connector ingests messages from Slack channels. Each message and each thread reply becomes a separate document in the vector store.

Channels can be specified directly by ID or resolved dynamically via name patterns or regex.

Authentication requires a Slack bot token with the following scopes: `channels:history`, `groups:history`, `channels:read`, `groups:read`, `users:read`. Invite the bot to each target channel (`/invite @YourBot`).

```yaml
# config.yaml

sources:
  - type: "slack"
    name: "slack1"
    config:
      token: "${SLACK1_TOKEN}"
      channel_ids: "${SLACK1_CHANNEL_IDS}"          # comma-separated channel IDs (mutually exclusive with channel_patterns)
      # channel_patterns: "${SLACK1_CHANNEL_PATTERNS}" # channel name patterns or regex (mutually exclusive with channel_ids)
      # channel_types: "public_channel,private_channel"  # optional, used with channel_patterns
      # earliest_date: "2024-01-01"                 # optional: fetch messages from this date
      # latest_date: "2025-01-01"                   # optional: fetch messages up to this date
      schedules: "${SLACK1_SCHEDULES}"
```

```dotenv
# .env.rag

SLACK1_TOKEN=xoxb-your-bot-token
SLACK1_CHANNEL_IDS=C1234567890,C0987654321
SLACK1_CHANNEL_PATTERNS=general,^dev.*
SLACK1_SCHEDULES=3600
```

### IMAP Connector

The IMAP connector ingests emails from any IMAP server over implicit TLS or STARTTLS (Gmail, Outlook,
Exchange, self-hosted, etc.). Each email becomes a document with subject as title, parsed body as
content, and sender/recipient metadata (including Cc/Bcc). Mailboxes are auto-discovered when not
specified.

Server certificates are always verified (hostname + trust chain) — self-signed certificates will be
rejected unless the server's CA is trusted by the environment running the connector.

```yaml
# config.yaml

sources:
  - type: "imap"
    name: "imap1"
    config:
      host: "${IMAP1_HOST}"
      port: 993                     # optional, default 993 (IMAPS), or 143 when use_starttls is set
      username: "${IMAP1_USERNAME}"
      password: "${IMAP1_PASSWORD}" # app-specific password for Gmail
      mailboxes: "${IMAP1_MAILBOXES}" # optional, comma-separated; remove this line (not just the env var) to ingest all mailboxes
      since: "2024-01-01"            # optional, only ingest messages on or after this date (YYYY-MM-DD)
      use_starttls: false            # optional, default false; connect plaintext then upgrade via STARTTLS instead of implicit TLS
      schedules: "${IMAP1_SCHEDULES}"
```

```dotenv
# .env.rag

IMAP1_HOST=imap.example.com
IMAP1_USERNAME=user@example.com
IMAP1_PASSWORD=your-app-password
IMAP1_MAILBOXES=INBOX,Sent
IMAP1_SCHEDULES=3600
```

## Single Sign-On (SSO)

mAItion inherits full SSO support from OpenWebUI. SSO is disabled by default and configured
entirely via environment variables in `.env` (copied from `.env.openwebui.example`).

### Google OAuth2 example

1. Create OAuth2 credentials in [Google Cloud Console](https://console.cloud.google.com/apis/credentials).
   Set the authorized redirect URI to `http(s)://your-domain/oauth/google/callback`.
2. Add to `.env`:

```dotenv
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-client-secret
ENABLE_OAUTH_SIGNUP=True
```

3. Restart the OpenWebUI service: `docker compose up -d openwebui`
4. Visit http://localhost:3000 — a **Continue with Google** button will appear on the login page.

For all supported providers (Microsoft/Azure AD, GitHub, generic OIDC, trusted header SSO),
full configuration reference, and common gotchas, see the
[mAItion SSO docs](https://docs.maition.com/configuration/sso) and the
[OpenWebUI SSO troubleshooting guide](https://docs.openwebui.com/troubleshooting/sso/).

## Embeddings and Inference

### Embeddings support

The RAG backend supports:

* `local` — HuggingFace embeddings running in the RAG containers (default)
* `ollama` — an embedding model served by the configured external Ollama server
* `openrouter`
* `openai` or another OpenAI-compatible endpoint

### Inference support

The RAG rephrase endpoint supports:

* `ollama` — inference through the external Ollama server's OpenAI-compatible `/v1` API (default)
* `openrouter`
* `openai` or another OpenAI-compatible endpoint

The locally built derived ROAT image adds `provider: ollama` support to the pinned
Rag-Of-All-Trades backend. It has no registry image name, so Compose always builds it from
`Dockerfile.roat`. OpenWebUI and the RAG backend receive the same native root URL in
`OLLAMA_BASE_URL`; ROAT appends `/v1` internally for its OpenAI-compatible client.

### Default: local HuggingFace embeddings and Ollama inference

```yaml
# config.yaml

embedding:
  provider: local
  model_config: sentence-transformers/all-mpnet-base-v2
  embedding_dim: 768

inference:
  provider: ollama
  model_config: "${OLLAMA_DEFAULT_MODEL}"
```

```dotenv
# .env (shared Ollama connection and model)
ENABLE_OLLAMA_API=True
OLLAMA_BASE_URL=http://100.85.191.78:11434
OLLAMA_DEFAULT_MODEL=qwen2.5:14b
```

Keep `OLLAMA_BASE_URL` at the Ollama root URL without `/v1`. Compose supplies this
single value to OpenWebUI and ROAT, and ROAT appends `/v1` internally. The default
model is not downloaded by this stack; `qwen2.5:14b` must already exist on the external
server. To use another model, install it on that server first, set
`OLLAMA_DEFAULT_MODEL` in `.env`, and then run `docker compose up -d`.

### Embeddings-only HuggingFace local model

You can configure the service to use **local embeddings** only, in this mode
you can use any embedding model supported by HuggingFace. Inference is disabled in
this mode, so you won't be able to use the **rephrase** endpoint.

```yaml
# config.yaml

embedding:
  provider: local
  # you can use any embedding model supported by HuggingFace
  model_config: sentence-transformers/all-MiniLM-L6-v2
  embedding_dim: 384

inference:
  provider: null
  model_config: null
```

### Ollama embeddings (optional)

Ollama can also provide embeddings. Install a compatible embedding model on the
external server, then select it and use the dimension published for that model. For
example, run this on the Ollama host:

```bash
ollama pull nomic-embed-text
```

```yaml
# config.yaml

embedding:
  provider: ollama
  model_config: nomic-embed-text
  embedding_dim: 768

inference:
  provider: ollama
  model_config: "${OLLAMA_DEFAULT_MODEL}"
```

Changing embedding models or dimensions for an existing vector table requires re-indexing the source data.

### OpenRouter/OpenAI-compatible providers (optional)

You can use a remote OpenAI-compatible endpoint for embeddings, inference, or both. For example, to use
OpenAI for both:

```yaml
# config.yaml

embedding:
  provider: openai
  model_config: text-embedding-3-small
  embedding_dim: 1536

inference:
  provider: openai
  model_config: gpt-4o
```

```dotenv
# .env.rag
OPENROUTER_API_KEY=your-api-key
OPENROUTER_API_BASE=https://api.openai.com/v1
```

The backend retains the `OPENROUTER_API_KEY` and `OPENROUTER_API_BASE` variable names for both `openrouter`
and `openai` provider labels. For OpenRouter, set the base URL to `https://openrouter.ai/api/v1`, select
`provider: openrouter`, and use the provider's model IDs.

RAG inference and OpenWebUI chat are configured independently. To switch OpenWebUI to the same optional
OpenAI-compatible provider, set `ENABLE_OLLAMA_API=False`, `ENABLE_OPENAI_API=True`, `OPENAI_API_BASE_URL`,
`OPENAI_API_KEY`, and `OPENAI_DEFAULT_MODEL` in `.env` before the first start.

## Reference of the `config.yaml`

The `config.yaml` file contains the main configuration of the service.

> Environment variables (`${...}`) in the config file are evaluated at runtime.

```yaml
sources: # holds the list of sources to ingest from (Connectors)

  - type: directory # directory, s3, mediawiki, serpapi, web, jira, pipedrive, slack, or imap
    name: local_docs # arbitrary name for the connector, stored in metadata
    enabled: true # optional, set to false to skip this source entirely (default: true)
    config:
      path: "${DIRECTORY1_PATH}"
      recursive: true
      exclude_hidden: true
      exclude_empty: false
      schedules: "${DIRECTORY1_SCHEDULES}"

# configures models and dimensions for embeddings
embedding:
  provider: local # `local` (default), `ollama`, `openrouter`, or `openai`
  model_config: sentence-transformers/all-mpnet-base-v2 # model to use
  embedding_dim: 768 # dimensions must match the selected embedding model

# configures the LLM provider and model
inference:
  provider: ollama # `ollama` (default), `openrouter`, `openai`, or null
  model_config: "${OLLAMA_DEFAULT_MODEL}" # defaults to qwen2.5:14b

# vector store configuration
vector_store:
  table_name: embeddings
  hybrid_search: true # whether to use hybrid search or not
  chunk_size: 512 # chunk size for vector indexing
  chunk_overlap: 50 # overlap between chunks
  # hnsw indexes settings
  hnsw:
    hnsw_m: 16 # number of neighbors
    hnsw_ef_construction: 64 # ef construction parameter for HNSW
    hnsw_ef_search: 40 # ef search parameter for HNSW
    hnsw_dist_method: vector_cosine_ops # distance metric for HNSW
```

## Tech Stack

* A derived [Rag-Of-All-Trades](https://github.com/wikiteq/rag-of-all-trades) image with Ollama `/v1` provider compatibility as the RAG backend
* An external [Ollama](https://ollama.com/) server for inference
* [OpenWebUI](https://github.com/open-webui/open-webui) v0.6.5 as a front-end

## Troubleshooting

### OpenWebUI does not start

Inspect the service states and relevant logs:

```bash
docker compose ps -a
docker compose logs --tail=200 api openwebui
```

Also review `config.yaml`, `.env.rag`, and `.env` for typos or mismatched model names.

### Ollama is unreachable or the model is missing

The Compose stack does not run Ollama or download models. From the Docker host,
verify that the external API is reachable and that `qwen2.5:14b` is installed:

```bash
curl http://100.85.191.78:11434/api/tags
```

If the model is absent, ask the external Ollama administrator to install it or
another model and update `OLLAMA_DEFAULT_MODEL`. Also ensure Ollama listens on an
interface reachable from the Docker host and that any firewall permits TCP port
`11434`.

### A changed Ollama model is not selected

Install the model on the external Ollama server, set `OLLAMA_DEFAULT_MODEL` in
`.env`, then run `docker compose up -d`. Compose uses that value for the OpenWebUI
default and RAG inference; it does not pull the model.
OpenWebUI provider settings are provisioned on its first boot and persisted, so an existing installation may
also need its Ollama connection updated to `http://100.85.191.78:11434` and its
default model updated in the Admin Panel.

### Directory documents are not ingested

Confirm that the host files are under the path selected by `DIRECTORY_HOST_PATH` in `.env` and are visible at
`/data/input` inside the worker:

```bash
docker compose exec celery_worker ls -la /data/input
docker compose logs --tail=200 celery_worker celery_beat
```

The default schedule is 3600 seconds, so ingestion is not necessarily immediate. `DIRECTORY1_PATH` should stay
set to `/data/input` unless the Compose mount target is also changed.

### HuggingFace connection timeout

```
requests.exceptions.ReadTimeout: (ReadTimeoutError("HTTPSConnectionPool(host='huggingface.co', port=443): Read timed out. (read timeout=10)"), '(Request ID: da122313-e11f-4d54-b4f3-187abfea0ca3)')
```

The default local embedding model is downloaded from HuggingFace during the first boot. If HuggingFace times
out, verify network access and retry the affected services without deleting the database volume:

```bash
docker compose restart api celery_worker
docker compose logs --tail=200 api celery_worker
```

## Star History

<a href="https://www.star-history.com/?repos=WikiTeq%2FmAItion&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=WikiTeq/mAItion&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=WikiTeq/mAItion&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=WikiTeq/mAItion&type=date&legend=top-left" />
 </picture>
</a>
