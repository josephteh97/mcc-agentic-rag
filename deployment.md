# WeKnora — Local RAG Deployment Guide

A reproducible setup for running Tencent **[WeKnora](https://github.com/Tencent/WeKnora)**
(an LLM-powered RAG / knowledge platform) locally on **Docker Desktop**, using
**100 % free local models** served by **Ollama** — no API keys, no cloud costs.

> Written from a working deployment on **Windows 10 + Docker Desktop (WSL2) + NVIDIA GPU**.
> Commands are PowerShell-first; macOS/Linux notes are included where they differ.

---

## 1. What you get

- WeKnora **core stack** — 5 containers:

  | Container | Role | Exposed |
  |---|---|---|
  | `WeKnora-frontend` | Web UI (nginx) | http://localhost (port 80) |
  | `WeKnora-app` | Backend API (Go) | http://localhost:8080 |
  | `WeKnora-docreader` | Document parsing (PDF / Office / OCR) | internal (gRPC) |
  | `WeKnora-postgres` | Database **+ vector store** (ParadeDB / pgvector) | internal |
  | `WeKnora-redis` | Cache / task queue | internal |

- Models (local, free, via Ollama):
  - **Chat:** `qwen2.5:3b`
  - **Embeddings:** `bge-m3` (1024-dim)

Only the default/core profile runs. Neo4j, MinIO, Langfuse, extra vector DBs, etc. are
optional `--profile` add-ons and are **not** needed for a working RAG.

**Data flow:** Browser → frontend → app → { PostgreSQL+pgvector, Redis, docreader } ; and
app → **Ollama on the host** (via `host.docker.internal`) for chat + embeddings.

---

## 2. Prerequisites

| Requirement | Notes |
|---|---|
| **Docker Desktop** | Compose v2+. Allocate the Docker VM **≥ 6–8 GB RAM**. |
| **Git** | To clone WeKnora. |
| **Ollama** | Runs on the **host**, not in a container. |
| **Host RAM** | **16 GB minimum** (Docker VM + Ollama share it). |
| **Disk** | **~8–10 GB** free (images ~5–6 GB + models ~3 GB). |
| **GPU** | Optional. NVIDIA GPU accelerates Ollama; 4 GB VRAM fits the models below. CPU-only works but is slower. |
| **Free ports** | `80` (UI) and `8080` (API). |

### Install prerequisites — Windows (PowerShell)

```powershell
winget install -e --id Docker.DockerDesktop
winget install -e --id Git.Git
winget install -e --id Ollama.Ollama
```

Then **start Docker Desktop once** and let it finish initializing. Open a fresh terminal so
`PATH` picks up the new tools.

### macOS / Linux

- **macOS:** Docker Desktop + `brew install git ollama`.
- **Linux:** Docker Engine + Compose plugin + Ollama. ⚠️ On *native* Docker (not Docker
  Desktop), `host.docker.internal` does not reach the host by default — see
  [§10 Networking](#10-networking-notes).

---

## 3. Pull the free models into Ollama

Ollama runs as a background service after install. Pull both models:

```powershell
ollama pull qwen2.5:3b
ollama pull bge-m3
```

Verify:

```powershell
ollama list
```

You should see `qwen2.5:3b` and `bge-m3:latest`.

**Model choice & sizing**
- `qwen2.5:3b` (~1.9 GB) — fast, multilingual chat; fits a 4 GB GPU.
- `bge-m3` (~1.2 GB) — strong multilingual embedding model for retrieval.
- Want higher quality and have the RAM/VRAM? Use `qwen2.5:7b` or `qwen3.5:4b` for chat and
  just enter that name in [§7](#7-first-run-configuration-in-the-browser).

---

## 4. Clone WeKnora

```powershell
cd C:\path\to\your\projects
git clone https://github.com/Tencent/WeKnora.git
cd WeKnora
```

All remaining commands run from inside the `WeKnora` folder.

---

## 5. Configure `.env`

Copy the template:

```powershell
Copy-Item .env.example .env
```

Generate the security keys and write them into `.env` so API keys and logins **survive
restarts** (without these, encrypted fields and sessions reset each restart):

```powershell
function New-HexKey([int]$bytes){ $b=New-Object 'byte[]' $bytes; [System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); ($b | ForEach-Object { $_.ToString('x2') }) -join '' }
$c = [IO.File]::ReadAllText('.env')
$c = $c -replace '(?m)^JWT_SECRET=.*$',         ('JWT_SECRET='        + (New-HexKey 32))  # 64 hex chars
$c = $c -replace '(?m)^SYSTEM_AES_KEY=.*$',     ('SYSTEM_AES_KEY='    + (New-HexKey 16))  # MUST be 32 chars
$c = $c -replace '(?m)^SYSTEM_SIGNING_KEY=.*$', ('SYSTEM_SIGNING_KEY='+ (New-HexKey 32))
$c = $c -replace '(?m)^# DEFAULT_LOCALE=.*$',   'DEFAULT_LOCALE=en-US'                     # English UI (optional)
[IO.File]::WriteAllText('.env', $c, (New-Object Text.UTF8Encoding($false)))
```

macOS / Linux equivalent:

```bash
cp .env.example .env
sed -i.bak \
  -e "s|^JWT_SECRET=.*|JWT_SECRET=$(openssl rand -hex 32)|" \
  -e "s|^SYSTEM_AES_KEY=.*|SYSTEM_AES_KEY=$(openssl rand -hex 16)|" \
  -e "s|^SYSTEM_SIGNING_KEY=.*|SYSTEM_SIGNING_KEY=$(openssl rand -hex 32)|" \
  -e "s|^# DEFAULT_LOCALE=.*|DEFAULT_LOCALE=en-US|" .env && rm -f .env.bak
```

> **`SYSTEM_AES_KEY` must be exactly 32 characters** (`openssl rand -hex 16` → 32 hex chars).
> Keep `.env` out of git (see `.gitignore`).

The core-stack defaults in `.env.example` already work as-is and need no changes:
`DB_DRIVER=postgres`, `RETRIEVE_DRIVER=postgres` (pgvector), `STORAGE_TYPE=local`,
`OLLAMA_OPTIONAL=true`, `OLLAMA_BASE_URL=http://host.docker.internal:11434`.

---

## 6. Start the stack

```powershell
docker compose pull
docker compose up -d
```

Wait ~1–2 minutes for health checks, then verify:

```powershell
docker compose ps
```

All services should be `Up` / `healthy`. Open **http://localhost**.

---

## 7. First-run configuration (in the browser)

1. **http://localhost** → **Create Account** (fill Username / Email / Password → Register →
   log in). This first account is your admin. **Save the credentials securely.**
2. Top-left **avatar → Model Management** (or **Settings → Models → Model Management**).
3. **Add Model → Chat**
   - Source: **Ollama**
   - Model Name: `qwen2.5:3b`  *(click **Refresh List** if the dropdown is empty)*
   - Context Window: `32768`
   - **Save and close**
4. **Add Model → Embedding**
   - Source: **Ollama**
   - Model Name: `bge-m3:latest`
   - Click **Detect Dimension** → it should report **1024**
   - **Save and close**
5. **Set the workspace defaults** — avatar → **Workspace Settings → Defaults**: set
   *Default chat model* = `qwen2.5:3b` and *Default embedding model* = `bge-m3`.
6. **Assign each built-in agent's model** — **Agents** → open **Quick Answer** (the RAG Q&A
   agent) → **Model Config** → set *Model* = `qwen2.5:3b` → **Save**. Registering a model in
   Model Management is **not** enough: an agent with no model gives
   *"Agent … is not ready. Please configure: Chat model"* the moment you chat. (The ReRank
   field shows as required but can be left empty — reranking is optional for a basic RAG.)
7. *(Optional)* **Model Test** (Model Management) → send a prompt to confirm each model responds.

---

## 8. Verify / smoke test

Container → host Ollama reachability:

```powershell
docker exec WeKnora-app curl -s http://host.docker.internal:11434/api/version
```

Returns a version JSON (e.g. `{"version":"0.40.14"}`) if the containers can reach Ollama.

Then in the UI: **Create Knowledge Base → upload a PDF or Markdown file → wait for parsing →
ask a question in chat.** You should get an answer with **source citations**.

---

## 9. Image understanding (vision / VLM)

*Optional.* This lets WeKnora **read images** — scanned documents, screenshots, diagrams,
photos — and make their content searchable. Each image becomes two retrievable chunks: an
`image_ocr` chunk (text read verbatim) **and** an `image_caption` chunk (a natural-language
description), both embedded with your embedding model.

1. **Pull a vision model** (`qwen3.5:4b` reads text in images well; `moondream` is a lighter,
   simpler alternative):
   ```powershell
   ollama pull qwen3.5:4b
   ```
2. **Register it** — Model Management → **Add Model** → Type **Vision** → Source **Ollama** →
   `qwen3.5:4b` → **Save and close**.
3. **Set it as default** — **Settings → Defaults → Default vision model** = `qwen3.5:4b`.
4. **Turn it on for a knowledge base** — either per upload (`enable_multimodel=true`, below) or
   per KB by setting `vlm_config` = `{"enabled": true, "model_id": "<vision-model-id>"}` when you
   create / `PATCH` the KB. Registering the model alone is **not** enough.

**Uploading images.** Standalone `PNG` / `JPG` files go in through the **REST API** (the UI
drop-zone lists only pdf/doc/txt/md, but the API accepts images). Create a key in **Settings →
API Integration → Create API Key** (capabilities: *ingest*, *retrieve*, *chat*), then:

```bash
curl -X POST "http://localhost/api/v1/knowledge-bases/<KB_ID>/knowledge/file" -H "X-API-Key: <API_KEY>" -F "file=@my-image.png" -F "enable_multimodel=true"
```

Poll `GET /api/v1/knowledge/<KNOWLEDGE_ID>` until `parse_status` is `completed`, then ask in chat
(scope it to the KB with **@**) or query the API:

```bash
curl -X POST "http://localhost/api/v1/knowledge-search" -H "X-API-Key: <API_KEY>" -H "Content-Type: application/json" -d '{"query":"your question","knowledge_base_ids":["<KB_ID>"]}'
```

**Verified end-to-end:** an uploaded image whose facts were entirely invented was answered
correctly from chat — *"Mass 12.6 kg · Catalog NW-MET-0047-K · Gallery 3, Plinth B"* — with a
source citation, proving the text was read from the image rather than recalled.

**Notes.** A VLM needs more VRAM: `qwen3.5:4b` (~3.4 GB) partly offloads on a 4 GB GPU, so the
first image is slower while the model loads. The optional per-document *summary* step may report
`failed` without affecting OCR, captioning, retrieval, or answers.

---

## 10. Networking notes

- **Docker Desktop (Windows / macOS):** containers reach the host Ollama via
  `host.docker.internal:11434` **out of the box** — Docker Desktop proxies it to host
  loopback, so **no Ollama change is required**. (Already set via `OLLAMA_BASE_URL` in `.env`.)
- **Native Docker on Linux:** `host.docker.internal` is not automatic. Either:
  - add `extra_hosts: ["host.docker.internal:host-gateway"]` **and** make Ollama listen on
    all interfaces — `OLLAMA_HOST=0.0.0.0:11434`, then restart Ollama; **or**
  - run Ollama as a container on the same Compose network and point `OLLAMA_BASE_URL` at it.
  - ⚠️ `OLLAMA_HOST=0.0.0.0` exposes Ollama to the LAN — only on a trusted network / behind a firewall.

---

## 11. Day-to-day operations

```powershell
docker compose up -d          # start
docker compose down           # stop (keeps all data)
docker compose ps             # status
docker compose logs -f app    # follow backend logs
```

Data persists in Docker volumes (`weknora_postgres-data`, `weknora_data-files`) across
restarts.

**Upgrade** — set a release tag in `.env` (`WEKNORA_VERSION=v0.x.y`), then:

```powershell
docker compose pull
docker compose up -d
```

**Full reset (⚠️ wipes all knowledge bases and data):**

```powershell
docker compose down -v
```

---

## 12. Troubleshooting

| Symptom | Fix |
|---|---|
| Port 80 already in use | Set `FRONTEND_PORT=8081` in `.env`, `docker compose up -d`, use http://localhost:8081. |
| Models missing from the Ollama dropdown | Confirm `ollama list` shows them and Ollama is running; click **Refresh List** in the form. |
| Container can't reach Ollama | See [§10](#10-networking-notes). On Linux set `OLLAMA_HOST=0.0.0.0:11434` and restart Ollama. |
| `app` stays unhealthy | `docker compose logs app` — usually the DB wasn't ready yet; wait or `docker compose restart app`. |
| First chat response is slow | The model loads into VRAM/RAM on first call; later calls are faster. Use a smaller model or add a GPU. |
| Logged out / API keys gone after restart | `SYSTEM_AES_KEY` / `JWT_SECRET` weren't set in `.env` ([§5](#5-configure-env)). |

---

## 13. Security notes

- This is a **local, single-user practice** deployment — keep it on `localhost` / a trusted
  network. WeKnora's own docs advise against exposing it to the public internet.
- `.env` (keys) and your admin login are **secrets** — never commit them (see `.gitignore`).

---

## 14. Reference — known-good configuration

Verified working on:

- **OS:** Windows 10 Pro
- **CPU:** Intel Core i7-1165G7
- **RAM:** 32 GB (Docker VM ~15.5 GB)
- **GPU:** NVIDIA T500 (4 GB VRAM)
- **Docker Desktop:** 4.65 (Compose v2) · **Ollama:** 0.40.14
- **WeKnora:** `WEKNORA_VERSION=latest`
- **Models:** `qwen2.5:3b` (chat) · `bge-m3` (embedding, dim 1024) · `qwen3.5:4b` (vision)
