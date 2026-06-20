# Local LLM Full Stack

A Docker Compose stack for running a full local AI workspace with Open WebUI, Postgres, SearXNG, and llama.cpp behind one configurable gateway.

The project is infrastructure-focused: it provides the Docker services, environment configuration, SearXNG settings, and llama.cpp model registry needed to run the stack locally.

## Services

| Service | Image | Purpose | Local port |
| --- | --- | --- | --- |
| `postgres` | `postgres:14` | Persistent database for Open WebUI | Internal only |
| `open-webui` | `ghcr.io/open-webui/open-webui` | Web UI for chat, model selection, settings, and RAG | `OPEN_WEBUI_PORT`, default `3000` |
| `searxng` | `searxng/searxng:latest` | Local web search engine used by Open WebUI RAG | `SEARXNG_PORT`, default `8080` |
| `llama_cpp` | `ghcr.io/ggml-org/llama.cpp:${IMAGE_TAG}` | llama.cpp API server for local models | `LLAMA_PORT`, default `8000` |

## Requirements

- Docker and Docker Compose
- NVIDIA GPU and NVIDIA Container Toolkit for CUDA-enabled llama.cpp images
- A GPU with enough VRAM for the selected quantized model
- Model `.gguf` files and corresponding `.gguf` metadata/projection files

## Quick start

1. Copy the sample environment file:

   ```bash
   cp .env.skel .env
   ```

2. Edit `.env` and add required secrets:

   ```env
   POSTGRES_PASSWORD="change-this-postgres-password"
   SEARXNG_SECRET="change-this-searxng-secret"
   ```

   Optionally set an API key for the llama.cpp server:

   ```env
   LLAMA_API_KEY="change-this-llama-api-key"
   ```

3. Place model files under `llama_cpp/models/` using the paths referenced by `llama_cpp/models.ini`.

4. Start the stack:

   ```bash
   docker compose up -d
   ```

5. Open Open WebUI:

   ```text
   http://localhost:3000
   ```

6. Open SearXNG directly if needed:

   ```text
   http://localhost:8080
   ```

The llama.cpp API is available at:

```text
http://localhost:8000
```

## Environment variables

`.env.skel` contains the default non-secret values. Secrets must be added to `.env`.

| Variable | Default | Description |
| --- | --- | --- |
| `IMAGE_TAG` | `server-cuda13` | llama.cpp image tag |
| `LLAMA_PORT` | `8000` | Host port for the llama.cpp API |
| `OPEN_WEBUI_PORT` | `3000` | Host port for Open WebUI |
| `SEARXNG_PORT` | `8080` | Host port for SearXNG |
| `POSTGRES_PASSWORD` | `postgres` | Database password used by Open WebUI and Postgres |
| `SEARXNG_SECRET` | Required | SearXNG secret key used for cookies and signed requests |
| `LLAMA_API_KEY` | Empty | Optional API key for the llama.cpp server |

`.env` is ignored by Git and should not be committed.

## llama.cpp model setup

Model definitions are registered in:

```text
llama_cpp/models.ini
```

The Docker Compose file mounts:

```text
./llama_cpp/models:/models
./llama_cpp/models.ini:/app/models.ini
```

The active model blocks currently configured are:

- `Nex-Mini-2-35B-A3B-Reasoning`
- `Qwen-Max`
- `Gemma-4-12B-Reasoning`

The llama.cpp service starts with:

```bash
--models-preset /app/models.ini
--models-autoload
--models-max 1
-ngl 99
```

That means it auto-loads models from `models.ini` but limits the runtime to one model at a time. The `ngl = 99` setting attempts to offload layers to the GPU; reduce this value if the model fails to load because of insufficient VRAM.

To add a model:

1. Add the model file to `llama_cpp/models/`.
2. Add a new section to `llama_cpp/models.ini`.
3. Point `m` to the model file path inside the container.
4. Point `mmproj` to the metadata/projection file if the model requires one.
5. Restart the `llama_cpp` service.

Example:

```ini
[My-Model]
m = /models/my-model/My-Model.gguf
mmproj = /models/my-model/My-Model-mmproj.gguf
ctx-size = 32768
temp = 0.7
top-p = 0.9
top-k = 40
ngl = 99
```

## Open WebUI configuration

Open WebUI is configured with authentication enabled:

```env
WEBUI_AUTH=true
```

It connects to Postgres using:

```env
DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/postgres
```

Web search is enabled through SearXNG:

```env
RAG_WEB_SEARCH_ENGINE=searxng
SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
```

Inside Open WebUI, add the llama.cpp server as an LLM provider and configure:

```text
Base URL: http://llama_cpp:8000
API key: your selected value from .env, if LLAMA_API_KEY is set
```

Use `http://llama_cpp:8000` inside Docker Compose, not `http://localhost:8000`.

## SearXNG configuration

SearXNG settings are mounted from:

```text
./searxng:/etc/searxng
```

The main configuration file is:

```text
searxng/settings.yml
```

The Compose file passes these environment variables into the SearXNG container:

```env
SEARXNG_PORT
SEARXNG_BIND_ADDRESS
SEARXNG_SECRET
```

`SEARXNG_SECRET` must be unique and should be changed before exposing SearXNG on a network.

## Useful commands

List running services:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f open-webui searxng llama_cpp postgres
```

Restart llama.cpp after changing model files or `models.ini`:

```bash
docker compose restart llama_cpp
```

Stop the stack:

```bash
docker compose down
```

Stop the stack and remove containers, networks, and named volumes:

```bash
docker compose down -v
```

Use `docker compose down -v` only if you are sure you want to delete persisted Postgres and Open WebUI data.

## Troubleshooting

### llama.cpp fails to start

Check for GPU runtime errors:

```bash
docker compose logs -f llama_cpp
```

If the CUDA image fails, verify that the NVIDIA Container Toolkit is installed and working on the host.

If a model fails to load, check:

- The paths in `llama_cpp/models.ini`
- File permissions under `llama_cpp/models`
- GPU VRAM availability
- The `ngl` value
- Whether the model requires a matching `mmproj` or metadata file

### Open WebUI cannot connect to Postgres

Verify that `POSTGRES_PASSWORD` in `.env` matches the value used in `docker-compose.yml`.

### Open WebUI search returns no results

Verify SearXNG is reachable:

```text
http://localhost:8080
```

Then check the Open WebUI logs:

```bash
docker compose logs -f open-webui
```

### SearXNG requires a secret

Set `SEARXNG_SECRET` in `.env` before starting the stack. The current Compose configuration uses a required secret placeholder:

```env
SEARXNG_SECRET=${SEARXNG_SECRET:?Set SEARXNG_SECRET in your shell or .env file}
```

## Security notes

This stack is intended for local use. Before exposing any ports publicly:

- Use strong values for `POSTGRES_PASSWORD` and `SEARXNG_SECRET`
- Do not expose Postgres to the host
- Use HTTPS or a trusted local network for Open WebUI
- Review SearXNG settings before making it public
- Back up Postgres and Open WebUI volumes regularly
