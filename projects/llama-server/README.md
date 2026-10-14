# ComfyUI MCP — host 192.168.30.254

Out-of-cluster setup: the podman-hosted ComfyUI (NVIDIA P40, `nvidia.com/gpu=1`)
exposed as an **authenticated MCP server** (Streamable HTTP, Bearer key) so that
Open WebUI (in-cluster) and laptop agents can drive it. **Zero Kubernetes changes**
— `mcpo` stays disabled. Design + rationale: `tmp/plans/2026-10-10-comfyui-mcp-host.md`
(scratch, not versioned).

This directory is the **source of truth** for the two files it ships; the copies
on `192.168.30.254` are deployed from here (manual — nothing here is
Flux-synced).

## Architecture

```
Open WebUI pods / laptop agents / in-cluster agents
   |  Bearer $COMFYUI_MCP_KEY
   v
comfyui-mcp :8001    supergateway (stateless Streamable HTTP, /mcp + /healthz)
   |  stdio
   v
comfy-mcp            official Comfy-Org MCP server (server_info, system_stats,
   |  subprocess        models/nodes, run_workflow, jobs, upload/download, ...)
   v
comfy-cli            COMFY_LOCAL_URL=http://192.168.30.254:8188
   |
   v
comfyui :8188        ComfyUI v0.34.0 on nvidia.com/gpu=1 (P40)
```

Two compose services, **one image** (`Containerfile`):

- `comfyui` — unchanged from today; 8188 stays published.
- `comfyui-mcp` — gateway only (no GPU); publishes 8001.

Key properties:

- **`COMFY_LOCAL_URL`, not `COMFYUI_URL`.** `comfy-mcp`'s `COMFYUI_URL` only
  remaps the submit/job tools; `server_info`, `system_stats` and model/node
  discovery are *not* remoted and would hit dead `127.0.0.1:8188` inside the
  gateway container. `COMFY_LOCAL_URL` (comfy-cli level) redirects every
  local-targeting command.
- **LAN-IP target** (`http://192.168.30.254:8188`, not the compose name
  `http://comfyui:8188`) so the `/view?fn=...` output URLs returned by MCP
  tools are fetchable from cluster pods and the laptop. Cost: one extra ufw
  rule for the podman bridge subnet (below).
- The gateway container **shares the `models` bind** with `comfyui`, so
  models downloaded via MCP land where ComfyUI can see them.
- Auth = `supergateway --apiKey` (fail-closed Bearer) on 8001 + ufw IP
  allowlists for 8188 **and** 8001. The key lives in the host-local `.env`
  (gitignored) — never in this repo.

## The image

`Containerfile` (replaces `~/.podman/Containerfiles/comfyui` on the host).
Base: `pytorch/pytorch:2.14.0-cuda12.6-cudnn9-runtime` (Ubuntu 24.04, Python
3.12.3). The MCP layer installs as the non-root `comfyui` user (user site /
user prefix, no root):

| Piece | Pin | Notes |
|---|---|---|
| `comfy-cli` | `>=1.14.0` (floor `comfy-mcp` enforces) | pip user site → `~comfyui/.local/bin`; `comfy set-default /opt/ComfyUI` sets the workspace |
| `comfy-mcp` | latest at build | official stdio MCP server |
| Node.js | `v22.23.3` tarball + sha256 (ARG `NODE_SHA256`) | base has **no node, no curl**; extracted to `~comfyui/.local` |
| `supergateway` | exact npm version `4.1.0` (ARG) | Node app (needs node ≥ 20); `npm install --prefix ~comfyui/.local`. The npm integrity hash is verified by npm itself |

`ENV PATH="/home/comfyui/.local/bin:${PATH}"` is what makes `comfy`,
`comfy-mcp`, `node` and `supergateway` resolvable at runtime — do not drop it.

**Tag scheme:** `localhost/comfyui:v0.34.0-<N>` — bump `N` on every rebuild
and update the image lines in `compose.yaml`. The live host
currently runs `-1` (pre-MCP); the first MCP build is `-2` (already pinned in
the compose file).

### Build (on 192.168.30.254)

```bash
git clone git@github.com:nullvar2000/ai-server.git ~/ai-server   # first time only
git -C ~/ai-server pull                                          # subsequent builds
podman build -f ~/ai-server/projects/llama-server/Containerfile \
    -t localhost/comfyui:v0.34.0-2 ~/ai-server/projects/llama-server
```

Post-build sanity (no GPU needed):

```bash
podman run --rm localhost/comfyui:v0.34.0-2 sh -c \
  'comfy --version && comfy-mcp --help | head -3 && supergateway --help | head -3'
# expect: comfy-cli version (>= 1.14.0), comfy-mcp usage, supergateway usage
```

## One-time bootstrap (on 192.168.30.254)

1. **Key:**
   ```bash
   openssl rand -hex 32
   printf 'COMFYUI_MCP_KEY="%s"\n' "<hex>" > /home/paul/llama-server/.env
   ```
   (No `.env` exists in that project dir today; the llama services use
   `--api-key-file`, no collision. Keep the quotes — compose parses unquoted
   values loosely.)
2. **Compose:** diff this repo's `compose.yaml` against the live
   `/home/paul/llama-server/container-compose.yaml` (expected delta: the
   `comfyui-mcp` service + the comfyui image tag), then copy it over as
   `compose.yaml` (compose-spec default name — `podman compose` auto-detects
   it), delete the legacy `container-compose.yaml`, and
   `cd /home/paul/llama-server && podman compose up -d`.
   Note: because the comfyui image tag changes, `up -d` recreates the
   comfyui container (~1 min GPU downtime — expected once).
3. **ufw** — see runbook below.
4. **Verify** — see below.
5. **Consumers** — see below.

## ufw runbook (on 192.168.30.254)

First `sudo ufw status`: if ufw is **inactive**, 8188 is currently open to
the whole LAN and `ufw enable` is a deliberate baseline change for all ports
(`ufw allow OpenSSH` first, `--dry-run` preview, verify SSH survives before
closing the terminal).

```bash
sudo ufw allow OpenSSH          # SAFETY FIRST
for ip in 192.168.30.101 192.168.30.102 192.168.30.103 \
          192.168.30.111 192.168.30.112 192.168.30.113 192.168.30.114 \
          192.168.30.121 192.168.30.122 192.168.30.123; do
  sudo ufw allow from "$ip" to any port 8188,8001 proto tcp
done
sudo ufw allow from 10.244.0.0/16 to any port 8188,8001 proto tcp   # k8s pod CIDR
sudo ufw allow from 10.89.0.0/24  to any port 8188 proto tcp        # podman bridge: mcp -> comfyui
sudo ufw allow from <LAPTOP_IP> to any port 8188,8001 proto tcp
sudo ufw --dry-run enable
sudo ufw enable
sudo ufw status verbose
```

- Node IPs: the 10 current Talos nodes (`kubectl get nodes -o wide`); add any
  future node's IP. Both node IPs and pod CIDR are allowed because pod→LAN
  egress may carry the pod IP (Calico, no SNAT) or the node IP (SNATed
  ClusterIP traffic).
- `10.89.0.0/24` is the compose bridge (`llama-server_default`); needed only
  because the gateway reaches ComfyUI via the host's published 8188 (hairpin).
  8001 stays closed to it.
- Also check the router for port-forwards of 8188/8001 to the internet —
  disable if present (they'd bypass the allowlist).

**Hairpin fallback:** if, after enabling, the gateway can no longer reach
`192.168.30.254:8188` (ufw routes the hairpin through FORWARD instead of
INPUT), either add `sudo ufw route allow from 10.89.0.0/24 to any port 8188
proto tcp` or switch the service's env to
`COMFY_LOCAL_URL=http://comfyui:8188` (pure L2, firewall-immune; then
tool-returned output URLs are only resolvable inside the compose network).
Verify either way with:

```bash
podman exec llama-server_comfyui-mcp_1 python -c \
  "import urllib.request; print(urllib.request.urlopen('http://192.168.30.254:8188/api/system_stats').status)"
# expect: 200
```

## Rebuild / deploy (recurring)

1. Edit `Containerfile` here (bump an ARG pin if a component moved).
2. Bump the tag counter (`v0.34.0-<N+1>`) and update both image lines in
   `compose.yaml`.
3. Push/merge; on `.254`: `git -C ~/ai-server pull && podman build ... && podman
   compose up -d` (recreates both comfyui containers — brief GPU downtime).
4. Re-run the sanity command + the verification below.

## Verification

```bash
curl -s http://192.168.30.254:8001/healthz        # expect: ok
# no key -> 401:
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://192.168.30.254:8001/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"smoke","version":"0"}}}'
# with key -> serverInfo (comfy-mcp):
KEY="<the .env value>"
curl -s -X POST http://192.168.30.254:8001/mcp -H "Authorization: Bearer $KEY" \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"smoke","version":"0"}}}'
```

Gates:

1. **Remote-target gate** — via MCP Inspector (Streamable HTTP,
   `Authorization: Bearer <key>` header) or an OWUI chat: `server_info`
   reports ComfyUI at `192.168.30.254:8188` running (v0.34.0);
   `system_stats` shows the **Tesla P40 24 GB**; the checkpoint/model list
   returns real names from the P40 box. This proves `COMFY_LOCAL_URL` is
   honored end-to-end.
2. **E2E** — `run_workflow` with a known workflow JSON → job appears in
   `jobs` → P40 renders → output retrievable (`fetch_outputs` / `/view?`).
3. **Firewall matrix** — laptop + a cluster node: 200 on 8188 and
   `:8001/healthz`; an off-list LAN device: timeout/refused on both.
4. **No cluster impact** — `kubectl get pods -A` unchanged.

## Consumer configuration (manual)

1. **Open WebUI** (v0.11.3, native MCP): Admin → Integrations → **External
   Tool Servers** → new → Type **MCP (Streamable HTTP)**; Name `comfyui`;
   URL `http://192.168.30.254:8001/mcp`; auth **Bearer** = the
   `COMFYUI_MCP_KEY` value.
2. **Laptop** (Claude Code; other clients: equivalent `mcpServers` entry
   with an `Authorization` header):
   ```bash
   claude mcp add --transport http comfyui http://192.168.30.254:8001/mcp \
     --header "Authorization: Bearer $COMFYUI_MCP_KEY"
   ```
3. **In-cluster agents** (e.g. hermes-agent): same URL — the
   cluster→LAN precedent is open-webui's `ollamaUrls`
   (`flux/apps/ai/open-webui/helm-release.yaml`).
4. **Optional uptime-kuma monitor**: HTTP(s), `http://192.168.30.254:8001/healthz`,
   60 s — with `--healthCheck server` this is a real MCP initialize+ping, not
   just a port check.

## Ops

- **Key rotation:** `supergateway` accepts multiple keys simultaneously —
  add the new key (recreate the container), migrate clients, remove the old.
- **Stateless→stateful fallback:** if a client shows session-related errors,
  add `--stateful --sessionTimeout 60000` to the `comfyui-mcp` command.
- **Licensing** (if the image is ever published): `comfy-mcp`
  AGPL-3.0-or-later or Comfy Commercial; `comfy-cli` GPL-3.0; `supergateway`
  MIT. Private home use is unencumbered.
- **ComfyUI 8188 stays unauthenticated** for allowlisted hosts (accepted
  home-LAN posture). ComfyUI-Login remains a possible follow-up — first
  verify `comfy-cli` sends a Bearer token to a local server.

## Rollback

1. Remove the `comfyui-mcp` service from the live compose; `podman compose
   up -d` (8001 gone). Revert the comfyui image tag if desired.
2. Delete the ufw rules added above; `sudo ufw reload`.
3. Consumers: delete the OWUI tool server, `claude mcp remove comfyui`,
   remove the uptime-kuma monitor.
4. Repo: revert this directory's commit (inert on the host). No cluster
   changes to revert — `mcpo` stays disabled throughout.
