# selfhosted

A collection of self-contained Docker Compose stacks for running personal
services at home. Each top-level folder is one stack with its own
`docker-compose.yml`, `.env.example`, and README.

Windows + Docker Desktop (driven from WSL) is what this repo is developed on,
but the stacks are plain Compose and should run anywhere Docker runs.

## Stacks

| Folder | What it runs | Exposed on |
| --- | --- | --- |
| [`rss/`](rss/README.md) | RSSHub, FreshRSS, Redis, browserless, and a small nginx `/rss` proxy | `1200` (RSSHub), `127.0.0.1:8080` (FreshRSS) |

More stacks (`media/`, `monitoring/`, …) can be added alongside `rss/` without
touching the existing ones.

```
selfhosted/
├── README.md          # this file
├── .gitignore         # ignores .env, overrides, logs
└── rss/               # RSS stack
```

## Conventions

- **One folder per stack**, each with its own `docker-compose.yml` and README.
- **Compose project names are pinned** in each file so container names and named
  volumes stay stable when a folder is moved or renamed.
- **Secrets stay out of git.** Every stack ships an `.env.example`; the real
  `.env` is git-ignored. Tokens, cookies, and keys never belong in a compose
  file, a README, or a commit.
- **Bind internal services to `127.0.0.1`** and expose them through a VPN
  (Tailscale, WireGuard, …) instead of publishing them on the LAN.

## Usage

```bash
cd rss
cp .env.example .env     # fill in credentials
docker compose up -d
docker compose ps
```

See each stack's README for its own quirks, debugging notes, and rotation advice.

## License

Personal configuration, provided as-is under the MIT license. No warranty; read
each stack's README before running it against data you care about.
