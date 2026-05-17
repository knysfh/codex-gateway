# codex-gateway

[简体中文](./README.zh-CN.md)

`codex-gateway` is a portable Docker Compose bootstrap for routing Codex-compatible code-agent traffic through a VPS, CLIProxyAPI/CPA, and Cloudflare. It is designed for operators who already own a domain such as `abc.com`, already use Cloudflare, and want a small self-hosted gateway they can bring up quickly.

This repository does not try to provision your VPS, DNS, or reverse proxy automatically. Its scope is the runtime bundle: `config.yaml`, `docker-compose.yml`, local persistence directories, and an Nginx sample you can merge into an existing edge setup.

## What This Project Does

- Exposes a Codex-compatible `/v1` API behind `codex.abc.com`
- Exposes a CPA Manager web UI behind `cpam.abc.com`
- Keeps the container ports bound to `127.0.0.1` so Nginx and Cloudflare remain the public edge
- Provides a repeatable layout for auth state, logs, and local data

## Deployment Flow

### 1. Prerequisites

- A Linux VPS with Docker Engine and the Docker Compose plugin
- A domain already managed by Cloudflare, for example `abc.com`
- An HTTPS reverse proxy on the VPS. This repo includes [nginx.sample.conf](./nginx.sample.conf) as the reference layout.
- `openssl` installed on the VPS

### 2. Generate the Management Key and Client API Key

Generate two independent random values:

```bash
MANAGEMENT_KEY="$(openssl rand -hex 32)"
CLIENT_API_KEY="$(openssl rand -hex 32)"

printf 'Management Key: %s\n' "$MANAGEMENT_KEY"
printf 'Client API Key: %s\n' "$CLIENT_API_KEY"
```

Replace the two placeholders in [config.yaml](./config.yaml):

- `CHANGE_ME_MANAGEMENT_KEY` -> the value of `MANAGEMENT_KEY`
- `CHANGE_ME_CLIENT_API_KEY` -> the value of `CLIENT_API_KEY`

One repo-specific detail matters here: CPA Manager also reads the management key from `./secrets/cpa_management_key`, so keep the plaintext value and write it to that file before the first boot:

```bash
mkdir -p auths data logs secrets
printf '%s' "$MANAGEMENT_KEY" > secrets/cpa_management_key
chmod 600 secrets/cpa_management_key
```

Important: CLIProxyAPI may hash the management key back into `config.yaml` on first start. Keep the original plaintext `MANAGEMENT_KEY` somewhere safe, because you still need that plaintext to sign in to the management UI later.

### 3. Start Docker Compose

Bring the stack up:

```bash
docker compose up -d
docker compose logs -f --tail=100
```

Quick local checks:

```bash
curl -i http://127.0.0.1:8317/v1/models
curl -i http://127.0.0.1:18317/health
```

At this stage the services should remain local-only. Do not publish `8317`, `18317`, or `1455` directly to the internet.

### 4. Put Nginx in Front

Merge the two server blocks from [nginx.sample.conf](./nginx.sample.conf) into your existing Nginx configuration and adjust certificate paths for your environment.

Expected routing:

- `codex.abc.com` -> `127.0.0.1:8317`
- `cpam.abc.com` -> `127.0.0.1:18317`

Reload Nginx after the configuration is in place.

### 5. Add the Cloudflare DNS Records

In Cloudflare DNS, add:

- `A` record: `codex.abc.com` -> your VPS public IP
- `A` record: `cpam.abc.com` -> your VPS public IP

Recommended Cloudflare settings:

- Proxy both records through the orange cloud
- Set `SSL/TLS` to `Full (strict)`
- Bypass caching for `codex.abc.com/*` and `cpam.abc.com/*`
- Do not enable an interactive challenge page on `codex.abc.com`, or CLI clients may fail

### 6. Management UI Screenshot

![Cloudfalre DNS Config](./docs/images/cloudflare-dns-config.jpg)

![Cloudfalre Security Rule](./docs/images/cloudflare-security-rule.jpg)

### 7. Sign In to the Service

Open the management UI in your browser:

```text
https://cpam.abc.com/
```

When prompted, enter the original plaintext `CHANGE_ME_MANAGEMENT_KEY` value you generated with `openssl`. That completes the management login.

### 8. Complete Codex OAuth

From the management UI, start the Codex OAuth flow. During the login process you will get a long callback URL that points to `http://localhost:1455/...`.

Copy that full `localhost:1455` callback URL and paste it back into the page. Once the callback is accepted and credentials are written into `auths/`, the gateway deployment is complete.

## Operational Notes

- Keep `auths/`, `data/`, `logs/`, and `secrets/` out of Git and out of any public file serving path.
- `1455` is only for the OAuth callback flow and should stay bound to `127.0.0.1`.
- If you need stricter protection for `cpam.abc.com`, add Cloudflare Access in front of it.

## Acknowledgements

This setup stands on a few solid open source projects:

- [router-for-me/CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)
- [seakee/CPA-Manager](https://github.com/seakee/CPA-Manager)
- [nginx/nginx](https://github.com/nginx/nginx)
- [docker/compose](https://github.com/docker/compose)
