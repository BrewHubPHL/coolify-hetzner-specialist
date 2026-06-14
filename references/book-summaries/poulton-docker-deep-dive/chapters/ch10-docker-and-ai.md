# Chapter 10: Docker and AI

## Core Idea

Docker Model Runner (DMR) is the preferred way to run local LLMs with Docker: it executes models as a host process (outside containers) for direct GPU/NPU access, exposes OpenAI-compatible APIs, and integrates with Docker CLI, Hub, and Compose—while containerized model servers remain a slower fallback.

## Frameworks Introduced

- **Docker Model Runner (DMR)**: Host-side inference process wrapping pluggable runtimes (default `llama.cpp`), dynamically loading models and serving OpenAI-compatible endpoints.
  - When to use: Local AI on Mac/Windows with privacy, cost control, latency sensitivity, or custom prompts; existing Docker users consolidating tooling.
  - How: Enable in Docker Desktop → Features in development; pull models with `docker model pull`; apps call `http://model-runner.docker.internal/engines/v1` from containers or `localhost:12434` from the host.

- **Models as OCI artifacts**: AI models stored in registries under the `ai/` namespace with GGUF layers and model-specific mediaTypes—same pull/push workflow as images.
  - When to use: Distributing models through existing OCI registries (Docker Hub, private registries) without registry sprawl.
  - How: `docker model pull ai/gemma3:4B-Q4_K_M`; blobs land in `~/.docker/models/blobs/sha256/`.

- **Compose provider extension for DMR**: Declare DMR as a Compose service with `provider: type: model` so Compose orchestrates model availability alongside app tiers.
  - When to use: Multi-tier chatbot apps where frontend/backend containers depend on a local model.
  - How: Requires Docker Compose v2.35+; `depends_on` chains frontend → backend → dmr service.

## Key Concepts

- **DMR vs containers for AI**: Containers cannot easily access most AI accelerators (NPUs, TPUs, non-NVIDIA GPUs); DMR runs on host hardware with Docker toolchain integration.
- **OpenAI-compatible endpoints**: `/engines/llama.cpp/v1/chat/completions`, `/completions`, `/embeddings`, `/models`—third-party apps (Open WebUI) plug in via `OPENAI_API_BASE_URL`.
- **Quantization**: Reduces model size (e.g. Q4_K_M) with modest accuracy trade-off; models still often multi-GB.
- **model-runner.docker.internal**: Special hostname containers use to reach DMR without host port mapping.
- **Host-side TCP (port 12434)**: Optional setting for localhost and remote access to DMR APIs.
- **Verified Publisher models**: Docker Hub `ai/` catalog under Docker Verified Publisher Program.

## Mental Models

- **DMR is "Docker for models, not models in Docker"**: Integration with Docker ecosystem without container GPU limitations.
- **Think registry-unified**: Models beside images, SBOMs, Helm charts, Wasm modules—one OCI workflow.
- **Prefer DMR over containerized Ollama**: Unless you have CUDA NVIDIA GPUs + NVIDIA Container Toolkit and accept complexity; otherwise DMR is faster and simpler.
- **Provider extension = dependency injection for LLMs**: Compose ensures the model is pulled and ready before backend starts.

## Anti-patterns

- **Running models in containers without NVIDIA Container Toolkit**: Models fall back to slow CPU inference.
- **Assuming CLI `docker model run` maintains conversation history**: REPL treats each prompt independently; use Desktop UI or app integration for context.
- **Trusting publisher benchmarks alone**: Always test models against your specific use case.
- **Remote DMR with Compose provider extension**: Provider extension does not yet support remote DMR instances for `depends_on`.

## Code Examples

```bash
docker model status
docker model pull ai/gemma3:4B-Q4_K_M
docker model ls
docker model inspect ai/gemma3:4B-Q4_K_M
docker manifest inspect ai/gemma3:4B-Q4_K_M | jq
docker model run ai/gemma3:4B-Q4_K_M
docker model rm <model>
```

- **What it demonstrates**: DMR lifecycle—status, pull, list, inspect manifest, interactive test, delete.

```bash
curl -s localhost:12434/engines/v1/models | jq
curl -s http://localhost:12434/engines/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "ai/gemma3:4B-Q4_K_M",
    "messages": [
      {"role": "system", "content": "Keep your responses to one sentence only."},
      {"role": "user", "content": "How long is a day on Mars?"}
    ],
    "temperature": 0.7,
    "max_tokens": 500
  }' | jq -r '.choices[0].message.content'
```

- **What it demonstrates**: OpenAI-compatible chat completion against local DMR.

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    env_file:
      - .env
    depends_on:
      - backend
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - dmr
  dmr:
    provider:
      type: model
      options:
        model: ${LLM_MODEL_NAME}
```

```env
MODEL_HOST=http://model-runner.docker.internal/engines/v1
LLM_MODEL_NAME=ai/gemma3:4B-Q4_K_M
```

- **What it demonstrates**: Three-tier Remix/FastAPI chatbot with DMR as a Compose-managed model provider.

```yaml
volumes:
  open-webui:
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    environment:
      - DEFAULT_MODELS=${MODEL_NAME}
      - WEBUI_AUTH=False
      - OPENAI_API_KEY=${OPENAI_KEY}
      - OPENAI_API_BASE_URL=${MODEL_HOST}
    volumes:
      - open-webui:/app/backend/data
    ports:
      - "3001:8080"
    restart: always
    depends_on:
      - dmr
  dmr:
    provider:
      type: model
      options:
        model: ${MODEL_NAME}
```

- **What it demonstrates**: Open WebUI wired to DMR via OpenAI-compatible env vars.

## Reference Tables

| DMR native endpoints | Method |
|---------------------|--------|
| `/models` | GET |
| `/models/create` | POST |
| `/models/{namespace}/{name}` | GET, DELETE |

| OpenAI-compatible endpoints (llama.cpp runtime) | Method |
|------------------------------------------------|--------|
| `/engines/llama.cpp/v1/models` | GET |
| `/engines/llama.cpp/v1/chat/completions` | POST |
| `/engines/llama.cpp/v1/completions` | POST |
| `/engines/llama.cpp/v1/embeddings` | POST |

| Access path | URL |
|-------------|-----|
| From containers | `http://model-runner.docker.internal/engines/v1` |
| From host (TCP enabled) | `http://localhost:12434/engines/v1` |
| Remote host | `http://<host>:12434/engines/v1` |

| Command | Purpose |
|---------|---------|
| `docker model status` | Running state and runtime info |
| `docker model pull` | Download model from OCI registry |
| `docker model push` | Push model to registry |
| `docker model ls` | List local models |
| `docker model inspect` | Tag, format, architecture, quantization |
| `docker model rm` | Delete local model |

| DMR vs alternatives | DMR advantage |
|--------------------|---------------|
| vs Ollama / LM Studio | Seamless Docker CLI, Hub, Compose, MCP integration |
| vs containerized Ollama | Broader accelerator access without NVIDIA toolkit |

## Worked Example

End-to-end DMR chatbot from `ddd-book/dmr/`:

1. Enable DMR + host-side TCP (port 12434) in Docker Desktop → Features in development.
2. `docker model status` — confirm llama.cpp runtime (e.g. Metal on Apple Silicon).
3. `docker model pull ai/gemma3:4B-Q4_K_M` — verify with `docker model ls`.
4. Set `.env`: `MODEL_HOST=http://model-runner.docker.internal/engines/v1`, `LLM_MODEL_NAME=ai/gemma3:4B-Q4_K_M`.
5. `cd ddd-book/dmr && docker compose up --build --detach` — starts dmr → backend → frontend.
6. Open `http://localhost:3000` — conversational chat with streaming via DMR.
7. Cleanup: `docker compose down --rmi all --volumes` in `dmr/` and `openwebui/`; `docker model rm` for models; disable DMR in Desktop settings.

## Key Takeaways

1. DMR runs models on host hardware, not in containers—key for GPU/NPU performance on Mac and Windows.
2. Models are OCI artifacts in the `ai/` Hub namespace; local store at `~/.docker/models/`.
3. OpenAI-compatible APIs enable any compatible third-party UI (Open WebUI) with minimal config.
4. Compose `provider: type: model` (v2.35+) declares model dependencies in multi-service apps.
5. Containerized model servers (Ollama) are deprecated path—only viable with NVIDIA Container Toolkit on CUDA GPUs.
6. DMR is Docker Desktop today; Docker CE integration planned for Linux and CI/CD.

## Connects To

- **Ch 9 (Compose)**: Same Compose workflow; DMR adds the `provider` extension for AI.
- **Ch 11 (Wasm)**: Models listed alongside Wasm modules as OCI artifact types in unified registries.
- **Coolify / self-hosting**: Local LLM patterns mirror privacy-first deployment on your own infrastructure.
- **Open WebUI / MCP Toolkit**: Third-party and Docker MCP integrations consume DMR endpoints.
