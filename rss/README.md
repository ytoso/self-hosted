# rss

Self-contained compose project for the `rsshub` / `freshrss` / `mistral-proxy` stack
on `mechrev-laptop` (Windows + Docker Desktop, driven from WSL). Part of the
[`selfhosted`](../README.md) repo.

Project name is pinned (`name: yutong`) so container names (`yutong-*`) and volumes
(`yutong_redis-data`, `yutong_freshrss_data`, `yutong_freshrss_extensions`) stay the
same as before the move.

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

## Everyday commands

Run these from this directory (`selfhosted/rss`):

```bash
docker compose up -d              # start / apply changes
docker compose ps
docker compose logs -f rsshub
docker compose up -d --build mistral-proxy   # after editing mistral-proxy/*
```

## Fix 1 — mistral-proxy no longer bind-mounts from a Windows path

**Symptom:** container shows `Up`, `nginx -t` passes, logs look normal, but
`http://mistral-proxy/rss` from `rsshub` never answers.

**Cause:** the config was bind-mounted (`./mistral-proxy:/etc/nginx/conf.d:ro`).
If the container is created before Docker Desktop has materialised that path, the
source is cached as an *empty directory*. nginx then has zero server blocks
(`include /etc/nginx/conf.d/*.conf` matches nothing), so nothing listens on `:80`
and `nginx -t` still reports success. It also survived `docker restart` — a restart
does not re-resolve a bind mount, only a recreate does.

Check for it with:

```bash
docker exec yutong-mistral-proxy-1 ls -la /etc/nginx/conf.d   # empty = broken
docker exec yutong-mistral-proxy-1 sh -c 'ss -ltn || netstat -ltn'  # port 80 missing
```

**Now:** the config is `COPY`-ed into the image (`mistral-proxy/Dockerfile`), so there
is no host path involved and the failure mode cannot recur. The image is rebuilt
with `--build`.

`default.conf` itself is unchanged in intent: it proxies `/rss` to
`https://mistral.ai/news/rss`, and uses a variable in `proxy_pass` plus
`resolver 127.0.0.11` so nginx re-resolves per request instead of pinning the IP it
saw at boot (a stale fake-IP from the local proxy/VPN was the original 404 source).

## Fix 2 — credentials out of docker-compose.yml

All tokens/cookies/keys now come from `.env` via `${VAR}` substitution:

| Variable | Used by |
| --- | --- |
| `BROWSERLESS_TOKEN` | `rsshub` (CDP URL), `browserless` |
| `TWITTER_USERNAME`, `TWITTER_PASSWORD`, `TWITTER_AUTH_TOKEN` | `rsshub` |
| `INSTAGRAM_COOKIE` | `rsshub` |
| `YOUTUBE_KEY` | `rsshub` |
| `GITHUB_ACCESS_TOKEN` | `rsshub` |
| `ZHIHU_COOKIES` | `rsshub` |

`.gitignore` excludes `.env`, and `.env.example` documents the shape. Useful checks:

```bash
docker compose config            # rendered config, secrets inlined
docker inspect yutong-rsshub-1 --format '{{range .Config.Env}}{{println .}}{{end}}'
```

Values are single-quoted inside `.env` so characters like `@ / + = ; |` are taken
literally by Compose's `.env` parser.

Note: `C:\Users\yutong` is **not** redirected to OneDrive (Desktop/Documents/Downloads
are local `%USERPROFILE%` paths), so this directory is not cloud-synced. If that ever
changes, move `.env` out of the synced tree and point Compose at it with `--env-file`.

## Rotation

Every credential above was previously stored in plaintext and has been visible in
`docker-compose.yml`, shell history, logs and this repo's history. Treat them as
compromised and rotate: GitHub PAT, YouTube API key, Twitter auth token/password,
Instagram session cookie, Zhihu cookies, Browserless token.

## Migration notes

- Moved here from `C:\Users\yutong\docker-compose.yml` + `C:\Users\yutong\mistral-proxy\`;
  the old copies are gone so there is only one source of truth.
- `CRON_MIN: "2,32"` was dropped from `mistral-proxy`: it is a FreshRSS setting and
  was never read by nginx.
- Renamed from `stack/` to `rss/` when this became the root of the `selfhosted` repo,
  since it only holds the RSS services.
- `zotero-arxiv-daily` is a separate compose project and was left untouched.
- Tagged Tailscale services `svc:freshrss` (→ `127.0.0.1:8080`) and `svc:rsshub`
  (→ `127.0.0.1:1200`) front this stack; they need `mechrev-laptop` to keep the
  `tag:freshrss` / `tag:rsshub` tags.
