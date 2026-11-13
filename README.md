# selfhosted

Docker Compose stacks for the services I run at home on `mechrev-laptop`
(Windows + Docker Desktop, driven from WSL).

Each top-level folder is one self-contained compose project.

| Folder | Project | Services | Exposed on |
| --- | --- | --- | --- |
| [`rss/`](rss/README.md) | `yutong` | RSSHub, FreshRSS, Redis, browserless, mistral-proxy | `1200` (RSSHub), `127.0.0.1:8080` (FreshRSS) |

Planned / future stacks get their own folder next to `rss/`, e.g. `media/`,
`monitoring/`, `notes/`.

```
selfhosted/
├── README.md          # this file
├── .gitignore         # ignores .env, overrides, logs
└── rss/               # RSS stack (RSSHub + FreshRSS + mistral-proxy)
```

## Conventions

- **One folder per stack**, each with its own `docker-compose.yml` and README.
- **Project names are pinned** in each compose file (`name: yutong` for `rss/`) so
  container names and named volumes survive folder moves.
- **Secrets stay out of git.** Every stack ships an `.env.example`; the real `.env`
  is git-ignored. Nothing that grants access (tokens, cookies, keys) belongs in a
  compose file, a README, or a commit.
- Ports that should not be reachable from the LAN are bound to `127.0.0.1` and
  fronted by Tailscale (`svc:freshrss`, `svc:rsshub`).

## Usage

```bash
cd rss
cp .env.example .env     # then fill in credentials
docker compose up -d
docker compose ps
```

See each stack's README for its own fixes, quirks and rotation notes.
