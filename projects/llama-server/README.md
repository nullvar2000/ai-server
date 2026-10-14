# llama-server — host 192.168.30.254

The llama.cpp LLM servers (ROCm on the AMD GPUs, CUDA on a Tesla P40) that
back the LLM endpoints agents use. Both speak the OpenAI-compatible API with
API-key auth.

This directory is the **source of truth** for `compose.yaml`; the copy on
`192.168.30.254` is deployed from here (manually) into
`/home/paul/llama-server/` (project dir = host dir).

> **ComfyUI is not here.** It was split out into its own compose project
> ([`../comfyui`](../comfyui/)) so that building/redeploying ComfyUI can
> never touch these LLM containers. Do not add it back.

## Architecture

| Service | Port | GPU | Model |
|---|---|---|---|
| `llama-server-rocm` | 9931 | AMD (HIP devices 0,1) | `Qwen3.8-27B-UD-Q8_K_XL.gguf`, alias `qwen3.8-27b`, 262144 ctx, MTP draft spec |
| `llama-server-cuda` | 9932 | `nvidia.com/gpu=0` (P40) | preset models (`--models-preset`), max 1 loaded |

Both read the API key from `secrets/llama.key` (host dir, bind-mounted
read-only) via `--api-key-file` — there is no `.env` in this project.

```
/home/paul/llama-server/
├── compose.yaml        # deployed from this repo
├── models/             # GGUF checkpoints + models-preset.ini (bind-mounted to /models)
└── secrets/llama.key   # API key (host-local, never committed)
```

## Notes

- `HSA_OVERRIDE_GFX_VERSION=10.3.0` is required for the AMD GPUs (gfx1030);
  ROCm install details live in the host's `notes` file.
- NVIDIA side: driver 580 + container-toolkit 1.18.2 held at those versions
  (`apt-mark hold`), GPUs exposed via CDI (`nvidia.com/gpu=N`).
- `restart: unless-stopped` on both; these are the long-running inference
  endpoints — **redeploying this project takes the LLM endpoints down**, so
  prefer rolling a single service (`podman compose up -d llama-server-rocm`)
  over a full `up -d` unless both are meant to restart.

## Rebuild / deploy

```bash
# on 192.168.30.254, after updating /home/paul/llama-server/compose.yaml
cd /home/paul/llama-server && podman compose config -q && podman compose up -d
```

Image bumps = change the `image:` tag and `up -d` (recreates that service).
No images are built for this project (pre-built `ghcr.io/ggml-org/llama.cpp`
tags).

## Verify

```bash
curl -s http://192.168.30.254:9931/health          # rocm
curl -s http://192.168.30.254:9932/health          # cuda
# with key:
curl -s http://192.168.30.254:9931/v1/models -H "Authorization: Bearer $(cat /home/paul/llama-server/secrets/llama.key)"
```
