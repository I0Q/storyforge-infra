# StoryForge / Infra Access Map (Claw)

Purpose: prevent “you have access / you don’t have access” confusion. This is the practical list of what Claw can reach *from the Mac Mini Infra host* and how.

## 1) DigitalOcean (API)

- **Access method:** DigitalOcean HTTP API via stored DO personal access token in OpenClaw auth profiles.
- **What this enables:**
  - Read/update App Platform app spec (env vars/secrets, domains, etc.)
  - Trigger App Platform deployments
  - Inspect deployment status
  - Manage many DO resources (per token permissions)
- **What this does NOT enable:**
  - Running shell commands *inside* a droplet.
  - Editing droplet filesystem directly.

## 2) DigitalOcean droplet: `sf-cloud-1` (cloud connector / VPC gateway host)

- **Public IP:** `159.65.251.41`
- **Private IP:** `10.108.0.3`
- **Tailscale IP:** `100.95.125.10`

### SSH access
- **Access method:** SSH from Mac Mini Infra.
- **User:** `root`
- **Key used:** `~/.ssh/id_ed25519_ubuntu_2026_01_31`
- **Verified command:** `ssh -i ~/.ssh/id_ed25519_ubuntu_2026_01_31 root@159.65.251.41 whoami` → `root`

### Services on `sf-cloud-1`
- **Tinybox compute node gateway (VPC-only):**
  - systemd: `tinybox-compute-node-gateway.service`
  - listen: `http://10.108.0.3:8791`
  - forwards to Tinybox over Tailscale.
  - tokens on droplet:
    - gateway auth token file: `/root/.config/storyforge-cloud/gateway_token`
    - tinybox bearer token file: `/root/.config/storyforge-cloud/tinybox_token`

### Recent change (2026-02-08)
- File changed: `/opt/tinybox-compute-node-gateway/app/main.py`
- Change: gateway previously proxied only `/v1/metrics` and `/v1/tts`.
- Fix: replaced with generic passthrough proxy for **all** `/v1/{path}` methods:
  - `GET/POST/PUT/PATCH/DELETE /v1/{anything}` → Tinybox `http://100.75.30.1:8790/v1/{anything}`
  - preserves query params
  - adds Tinybox `Authorization: Bearer <tinybox_token>`
  - returns raw body + status + content-type (no wrapper JSON)
- Service restarted: `systemctl restart tinybox-compute-node-gateway`

## 3) Tinybox (compute provider)

- **SSH alias:** `tinybox`
- **IP:** `192.168.5.199` (LAN)
- **User:** `tiny`
- **Key:** `~/.ssh/id_ed25519_ubuntu_2026_01_31`
- **Tailscale IP:** `100.75.30.1`

### Tinybox Compute Node API
- repo path: `/raid/tinybox-compute-node`
- service: `tinybox-compute-node.service`
- bind: `http://100.75.30.1:8790`
- token file (on Tinybox): `~/.config/tinybox-compute-node/token`

### Old local monitor app (Tinybox)
- path: `/raid/storyforge_test/monitor`
- listen: `http://192.168.5.199:8787`
- token-gated via `?t=<token>`
- token file: `/raid/storyforge_test/monitor/token.txt`
- repo root used by monitor: `/raid/storyforge_test`

## 4) StoryForge Cloud app (`storyforge.i0q.com`)

- **Access method:** public HTTPS + passphrase cookie OR token bootstrap endpoint.

### Automation bootstrap
- Endpoint: `POST /api/session`
- Auth: header `x-sf-todo-token: <TODO_API_TOKEN>`
- Effect: mints `sf_sid` cookie so browser automation can access passphrase-gated UI without typing.

## 5) Voice preset clips (Tinybox → Spaces)

### Default voice reference clips (Tinybox)
- Source refs discovered:
  - `/raid/storyforge_test/assets/voices/refs/cmu_arctic/*/ref.wav`

### Preset folder (for training UI)
- Preset root (created):
  - `/raid/storyforge_test/voice_presets/`
- Example presets created as symlinks:
  - `/raid/storyforge_test/voice_presets/cmu_arctic/{awb,bdl,clb,jmk,ksp,rms,slt}.wav`

### Important constraint
- At runtime, Tinybox must be able to fetch whatever assets it needs.
- Preferred: upload selected clips to **DigitalOcean Spaces** and pass Tinybox **Spaces URLs**.

## 6) DigitalOcean Spaces (StoryForge assets bucket)

- **Access method (local scripts):** `s3cmd` with local config file:
  - `storyforge/tools/s3cfg_storyforge` (contains Spaces access_key/secret_key; **do not paste into chat**)
- **Default bucket/region used by existing scripts:**
  - bucket: `storyforge-assets` (env override: `STORYFORGE_ASSETS_BUCKET`)
  - region: `sfo3` (env override: `STORYFORGE_ASSETS_REGION`)
  - public base example: `https://storyforge-assets.sfo3.digitaloceanspaces.com/assets/`
- **Example uploader script:**
  - `storyforge/tools/upload_sfx_tree_s3cmd.sh` (uploads `assets/sfx/` + `assets/credits/index.jsonl`)

### How this relates to the Cloud app
- The StoryForge App Platform service can upload to Spaces via `boto3` (we added code), but it currently needs its own env secrets:
  - `SPACES_KEY`, `SPACES_SECRET`, `SPACES_BUCKET`, `SPACES_REGION` (and optionally `SPACES_PUBLIC_BASE`).
- These values should match the StoryForge Spaces bucket above.

## 7) Known current blockers / gotchas

- **Spaces uploads from StoryForge Cloud** require env secrets:
  - `SPACES_KEY`, `SPACES_SECRET`, `SPACES_BUCKET`, `SPACES_REGION` (and optionally `SPACES_PUBLIC_BASE`).
  - Without these, `/api/voice_provider/preset_to_spaces` returns `spaces_not_configured`.

- **Gateway reachability from Mac Mini Infra:**
  - `http://10.108.0.3:8791` is VPC-only; it may not be reachable from the Mac Mini host.
  - Testing gateway endpoints should be done either:
    - from within DO VPC (e.g., on `sf-cloud-1`), or
    - via the StoryForge cloud app calling the gateway.
