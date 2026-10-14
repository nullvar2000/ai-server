# ai-server

Compose projects for the AI services running on podman hosts.

## Layout

```
.
├── README.md
├── .gitignore                 # ignores every .env (secrets never committed)
└── projects/
    └── <name>/                # one directory per compose project
        ├── README.md          # architecture, build, deploy, runbook
        ├── compose.yaml       # the compose file (compose-spec default name)
        ├── Containerfile      # image build (podman), colocated with compose
        ├── .dockerignore
        ├── .env.example       # committed template of every variable
        └── .env               # host-local secrets (gitignored)
```

## Conventions

- **Project dir = host dir.** Each `projects/<name>/` maps 1:1 onto a
  directory on the target host (e.g. `projects/llama-server` →
  `/home/paul/llama-server` on 192.168.30.254). The compose file, `.env` and
  the project's data directories all live together there.
- **`compose.yaml`** is the compose-spec default filename; `podman compose`
  and `docker compose` both auto-detect it (no `-f` needed when running from
  the project dir).
- **`Containerfile`** (not `Dockerfile`) — the stack is podman-only; it is
  podman's conventional build-file name.
- **Secrets:** every variable is templated in `.env.example` (committed);
  the real `.env` is created host-local and gitignored by the root
  `.gitignore`. Compose references required keys as `${VAR:?set VAR in .env}`
  so a missing key fails the startup, not the first request.
- **Volumes:** each project keeps its data inside its own host directory
  (bind-mounted in `compose.yaml`), e.g. `models/`, `secrets/`, and
  service-specific state subdirs — nothing scattered under `$HOME` root or
  `~/.podman/volumes/` unless a service requires it.

## Projects

| Project | Host | What it runs |
|---|---|---|
| [`llama-server`](projects/llama-server/) | 192.168.30.254 | llama.cpp LLM servers (ROCm + CUDA) |
| [`comfyui`](projects/comfyui/) | 192.168.30.254 | ComfyUI (P40) + ComfyUI MCP gateway |
