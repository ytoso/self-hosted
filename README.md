# selfhosted

My self-hosted services. Each folder is a self-contained stack; not everything
runs in Docker.

| Service | Folder |
| --- | --- |
| RSSHub + FreshRSS (with Redis, browserless, mistral-proxy) | [`rss/`](rss/README.md) |

## Usage

```bash
cd rss
cp .env.example .env     # fill in credentials
docker compose up -d
```

Real `.env` files are git-ignored; only `.env.example` templates are committed.
See each folder's README for details.
