---
id: "doc.pocketcloud.runbook.web-fleet"
title: "Web Fleet Deployment & Server Operations"
types: ["implementation-guide", "reference"]
project_id: "main"
status: "active"
tags: ["pocketcloud", "pocketcli", "web-fleet", "deploy", "servers", "vultr", "resend"]
owners: ["@steven"]
version: "1.11.0"
last_updated: "2026-10-04T00:00:00Z"
last_reviewed: "2026-10-04"
is_archived: false
---
# 5. Web Fleet Deployment & Server Operations

[← Back to Runbook Index](index.md)

> [!IMPORTANT]
> **Run every `pocket web …` command from the workspace root.** These verbs
> resolve their env config *relative to the current working directory*. From a
> subdirectory they fail with a doubled path that names a location nobody configured:
> `Site configuration not found: …/pocket-cloud-web-workspace/pocket-cloud-web-workspace/envs/pocketcloud/.env.site`.
> The fix is to run from the workspace root (or export `POCKET_CLOUD_CONFIG_DIR`).

```bash
pocket web deploy <site> main
pocket web verify <site> main
pocket web status
pocket web presence               # fleet presence health; exits non-zero on failing targets
pocket web logs <site> main       # PM2 logs from the active release
pocket web exec <site> main "<command>"   # run a command in the active release dir
pocket web db sync --site <site> [--via-ssh]  # snapshot fleet DBs into the local mirror
pocket web rollback <site> main   # back to the previous release
```

---

## Which Sites Exist

There is **no `pocket web site list`**. The inventory is the config repo's
`envs/` directory — one directory per site key, and the directory name *is* the
key you pass to every verb:

```bash
ls web-fleet/pocket-cloud-web-config/envs/
```

A site does **not** need a local repo to be deployed: deploy clones from the
site's git remote on the server. Several configured sites (`developer`, `lora`,
`pocketadmin`) have no checkout under `web-fleet/` and deploy exactly like the
ones that do.

---

## What is Actually Deployed Right Now?

Ask the running release directly on the server:

```bash
# one site
pocket web exec <site> main "node -p \"require('./server/node_modules/@pocketcloud/web-lib/package.json').version\""

# the whole fleet, from the workspace root
for s in clevercoding pocketcloud stevenmoon defensivelove emergencyreadyapp lora developer pocketadmin wanderlore; do
  V=$(pocket web exec $s main "node -p \"require('./server/node_modules/@pocketcloud/web-lib/package.json').version\"" 2>/dev/null \
      | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | head -1)
  printf '%-20s %s\n' "$s" "${V:-unresolved}"
done
```

Note the path: web-lib installs under **`server/node_modules/`**, not the
release root.

---

## Redeploying the Whole Fleet

Sequentially, never in parallel: the releases share a server, an npm cache, and
a disk.

```bash
cd /path/to/pocket-cloud-workspace
for s in defensivelove lora developer clevercoding stevenmoon emergencyreadyapp pocketadmin wanderlore pocketcloud; do
  pocket web deploy "$s" main || echo "FAILED: $s"
done
```

Each deploy hot-swaps and waits for HTTP readiness before finalising, so a
failure is reported rather than left half-applied. Budget ~2–4 minutes per site.

---

## Web-Lib Resolution Precedence

Each site repo's `server/scripts/resolve-web-lib.cjs` runs as `postinstall`. Its **first check
wins and is not what the docs above it describe**: if `npm install` has already
placed a package at `server/node_modules/@pocketcloud/web-lib`, the resolver
logs *"Using the bundled @pocketcloud/web-lib package"* and returns
immediately. Only when nothing is installed does it consult, in order,
`WEB_LIB_REF` → `webLib.ref` in the site's manifest → the parent repo's branch → `main`.

`pocket web exec` is also how you read **deployed source** when a satellite
site repo's `git fetch` hangs:

```bash
pocket web exec pocketcloud main "grep -n 'user' server/src/demo/demo.routes.ts"
```

The daily fleet presence audit is installed as a preset:
```bash
pocket task preset install web-fleet-presence-audit
```

---

## Beyond "Is it Up" — Verbs That Judge

```bash
pocket web health <site>          # warehouse measurement health, judged over SSH
pocket web products [--site <s>] [--json]   # declared products, funnel readout, config drift
pocket web analytics              # bot/AI-agent traffic intelligence across the fleet
```

Write verbs:
```bash
pocket web rollback <site> main   # previous release
pocket web backup <site> main     # snapshot the site database
pocket web purge <site> [--urls …]  # Cloudflare edge cache
```

---

## Two Healthy States That Read as Failures

1. **Catch-all domains sit in `pocket web status` as `stopped` with enormous restart counts.**
   `bettergolfapp`, `summacoding`, `supportingadoption` and `i-calq-newborn-screening` are served by
   the shared `pocket-cloud-catchall` process (ADR-0072) and run no process of their own.
2. **`pocket web verify` skips checks that do not apply.**
   On `pocketadmin` the GEO contract reports *"No robots.txt on this site; skipping the GEO contract"* —
   correct for a private control plane.

---

## 7.0 Fleet Config is Per-Machine

`envs/**/.env.site`, `.env.main`, `.env.develop` and `envs/.env.deploy` are
**gitignored** — they carry secrets. The config repo tracks only an encrypted
bundle:

```bash
# in the config repo (web-fleet/pocket-cloud-web-config):
# after cloning or pulling:
POCKET_CLOUD_CONFIG_PASSPHRASE=… ./scripts/env-bundle.sh unpack

# after changing a lane:
POCKET_CLOUD_CONFIG_PASSPHRASE=… ./scripts/env-bundle.sh pack
git add vault/envs.bundle.enc && git commit
```

`POCKET_CLOUD_CONFIG_DIR` overrides where the CLI looks, and it must name the
**`envs` directory itself**, not its parent:
```bash
export POCKET_CLOUD_CONFIG_DIR=/path/to/pocket-cloud-web-config/envs
```

---

## 7.1 Fleet Servers (ADR-0084)

Servers are **declared tiers**. `POCKET_SERVER_KEYS` in `envs/.env.deploy` is the
whole set; a key that is not listed fails.

Read-only:
```bash
pocket web server list
pocket web server show prod
pocket web server plans
```

Writes (dry-run by default; `--apply` required):
```bash
pocket web server snapshot prod
pocket web server lease pr-482 --ttl 4h
pocket web server reap
```

Ephemeral leases carry their expiry in the Vultr label and are destroyed by the hourly reaper:
```bash
pocket task preset install ephemeral-lease-reaper
```

---

## 7.2 Production Clones and Divergence

```bash
pocket web server verify-clone stage          # can it rejoin prod's tunnel?
pocket web server diverge stage               # dry run: what is still production-like
pocket web server diverge stage --apply --provision-tunnel
```

`diverge` stops inherited PM2/cron, confirms the inherited tunnel is gone, and
with `--provision-tunnel` creates the tier's **own** Cloudflare tunnel.

---

## 7.2b When a Deploy Fails

Deploy already attempts recovery automatically: if activation fails, deploy
restores the previous proven release, re-checks readiness, and *then* rethrows
the original error.

Manual fallback:
```bash
pocket web rollback <site> main
pocket web verify <site> main
```

---

## 7.3 OS Upgrades — Rehearse on Stage

```bash
pocket web server snapshot <tier> --apply     # the restore point
pocket web server os-upgrade <tier>           # DRY RUN by default
pocket web server os-upgrade <tier> --apply
pocket web server os-upgrade <tier> --status  # poll an upgrade already running
```

It will not start without a completed snapshot for that tier.

Set `POCKET_VULTR_VIA_SSH=prod` in `.env.deploy` so API calls egress via SOCKS
from the production node's allowlisted static IP.

---

## 7.4 Transactional Mail — Resend (ADR-0128)

Sites send magic links and password resets through Resend over HTTPS.

```bash
# Add client key via stdin (never argv):
pbpaste | pocketcloud-secrets --set-stdin t/<scope>/RESEND_API_KEY
pocketcloud-secrets --list-keys | grep '^t/'

# Register a sending domain:
pocket web mail domain add example.com                  # plan
pocket web mail domain add example.com --apply          # register, write DNS, verify
pocket web mail domain status example.com --verify      # Resend view

# Mint lane key:
pocket web mail key mint --site <site> --lane main --domain example.com --apply
pocket web mail send-test --from 'Brand <noreply@example.com>' --to you@example.com
```
