# rss

A self-contained Compose stack for reading feeds: **RSSHub** (feed generator),
**FreshRSS** (reader), **Redis** (RSSHub cache), **browserless** (headless
Chrome for Playwright-based routes), and **mistral-proxy** (a tiny nginx
sidecar that republishes Mistral's news RSS so RSSHub can fetch it).

```
rss/
├── docker-compose.yml
├── .env              # real credentials (git-ignored)
├── .env.example      # template
├── .gitignore
├── README.md
└── mistral-proxy/
    ├── Dockerfile
    └── default.conf
```

The Compose project name is pinned in `docker-compose.yml`, so container names
and named volumes stay stable when this folder is moved or renamed. The stack
uses named volumes (`redis-data`, `freshrss_data`, `freshrss_extensions`) — they
are the only place FreshRSS state lives, so back them up before changing the
project name.

## Quick start

```bash
cp .env.example .env      # fill in the credentials you actually use
docker compose up -d
docker compose ps
```

| Service | URL |
| --- | --- |
| RSSHub | `http://localhost:1200` |
| FreshRSS | `http://127.0.0.1:8080` |
| mistral-proxy | internal only, reached from RSSHub as `http://mistral-proxy/rss` |

FreshRSS is bound to `127.0.0.1`. To read it from another device, put it behind a
VPN (e.g. a Tailscale service or `tailscale serve`) rather than publishing the
port.

## Everyday commands

Run these from this directory:

```bash
docker compose up -d                          # start / apply changes
docker compose ps
docker compose logs -f rsshub
docker compose down                           # stop (volumes kept)
docker compose up -d --build mistral-proxy    # after editing mistral-proxy/*
```

## Fix 1 — don't bind-mount nginx config from a Windows host path

**Symptom:** the proxy container shows `Up`, `nginx -t` passes, logs look normal,
but `http://mistral-proxy/rss` from `rsshub` never answers.

**Cause:** the config was bind-mounted (`./mistral-proxy:/etc/nginx/conf.d:ro`).
If the container is created before Docker Desktop has materialised that path, the
source is cached as an *empty directory*. nginx then has zero server blocks
(`include /etc/nginx/conf.d/*.conf` matches nothing), so nothing listens on `:80`
and `nginx -t` still reports success. It also survives `docker restart` — a
restart does not re-resolve a bind mount, only a recreate does.

Check for it with:

```bash
docker compose exec mistral-proxy ls -la /etc/nginx/conf.d   # empty = broken
docker compose exec mistral-proxy sh -c 'ss -ltn || netstat -ltn'  # port 80 missing
```

**Now:** the config is `COPY`-ed into the image (`mistral-proxy/Dockerfile`), so
there is no host path involved and the failure mode cannot recur. Rebuild with
`--build` after editing it.

`default.conf` proxies `/rss` to `https://mistral.ai/news/rss` and uses a
variable in `proxy_pass` plus `resolver 127.0.0.11`, so nginx re-resolves per
request instead of pinning the IP it saw at boot (a stale fake-IP from a local
proxy/VPN was the original source of 404s).

## Fix 2 — credentials live in `.env`, not in compose

All tokens/cookies/keys come from `.env` via `${VAR}` substitution:

| Variable | Used by |
| --- | --- |
| `BROWSERLESS_TOKEN` | `rsshub` (CDP URL), `browserless` |
| `TWITTER_USERNAME`, `TWITTER_PASSWORD`, `TWITTER_AUTH_TOKEN` | `rsshub` |
| `INSTAGRAM_COOKIE` | `rsshub` |
| `YOUTUBE_KEY` | `rsshub` |
| `GITHUB_ACCESS_TOKEN` | `rsshub` |
| `ZHIHU_COOKIES` | `rsshub` |

`.gitignore` excludes `.env`, and `.env.example` documents the shape. Useful
checks:

```bash
docker compose config            # rendered config, secrets inlined
docker compose exec rsshub env   # what the container actually sees
```

Values are single-quoted inside `.env` so characters like `@ / + = ; |` are taken
literally by Compose's `.env` parser.

Every value is optional: missing ones become empty strings, except
`BROWSERLESS_TOKEN`, which defaults to `rsshub`.

## Rotation

Any credential that has ever been written into a compose file, a shell history,
a log, or a commit should be treated as compromised. If you adapt this stack,
rotate: GitHub PAT, YouTube API key, Twitter auth token/password, Instagram
session cookie, Zhihu cookies, and the Browserless token.

## Notes

- `CRON_MIN` is a FreshRSS setting and is not read by the nginx proxy — set it on
  the `freshrss` service if you need a custom refresh schedule.
- If the host's `$HOME` (or `%USERPROFILE%`) is cloud-synced (OneDrive, Dropbox,
  …), keep `.env` outside the synced tree and pass `--env-file /path/to/.env`.
