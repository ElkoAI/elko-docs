# Docker & Configuration — elko-fatcat-mcp

## Quick Run

```bash
docker run -d --name elko-fatcat \
  -p 8083:8083 \
  -v elko-fatcat-data:/data \
  jsoprych/elko-fatcat-mcp:latest
```

## With API Keys

```bash
docker run -d --name elko-fatcat \
  -p 8083:8083 \
  -v elko-fatcat-data:/data \
  -e CONGRESS_API_KEY=your_congress_key \
  -e FEC_API_KEY=your_fec_key \
  -e ELKO_AUDIT_HMAC_SECRET=$(openssl rand -hex 32) \
  jsoprych/elko-fatcat-mcp:latest
```

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ELKO_EASY_KEY` | bundled | 91-day rate-limit token baked at build time |
| `ELKO_KEY_SECRET` | — | Enable HMAC pro-key validation |
| `ELKO_REVOCATION_URL` | — | Pro-key revocation list URL |
| `CONGRESS_API_KEY` | — | Required for `fatcat_member_legislation`; free from api.congress.gov |
| `FEC_API_KEY` | DEMO_KEY | Recommended for `fatcat_member_donors` — DEMO_KEY rate limits are tight |
| `ELKO_AUDIT_HMAC_SECRET` | — | Enables HMAC-chained `audit_log_entries` (tamper-evident audit trail) |
| `POSTMARK_SERVER_TOKEN` | — | Enables weekly digest email delivery |
| `POSTMARK_FROM_EMAIL` | — | "From" address for digest emails |

---

## Docker Compose

```yaml
services:
  elko-fatcat:
    image: jsoprych/elko-fatcat-mcp:latest
    container_name: elko-fatcat
    ports:
      - "8083:8083"
    volumes:
      - elko-fatcat-data:/data
    restart: unless-stopped
    environment:
      - CONGRESS_API_KEY=${CONGRESS_API_KEY:-}
      - FEC_API_KEY=${FEC_API_KEY:-}
      - ELKO_AUDIT_HMAC_SECRET=${ELKO_AUDIT_HMAC_SECRET:-}
      - POSTMARK_SERVER_TOKEN=${POSTMARK_SERVER_TOKEN:-}
      - POSTMARK_FROM_EMAIL=${POSTMARK_FROM_EMAIL:-}

volumes:
  elko-fatcat-data:
```

---

## Port Configuration

Default port is `8083`. Override with `-p`:

```bash
# Run on port 9083 (host) → 8083 (container)
docker run -p 9083:8083 ... jsoprych/elko-fatcat-mcp:latest
```

---

## Data Persistence

SQLite database at `/data/elko-fatcat.db` inside the container. The warehouse grows over time as ingest runs:

| After | Approximate DB size |
|-------|--------------------|
| First quarter of 13F + Form 4 | ~100 MB |
| 1 year of all sources | ~600 MB |
| 5 years backfilled | ~3 GB |

```bash
# Named volume (recommended — survives container removal)
docker run -v elko-fatcat-data:/data ...

# Host directory
docker run -v /path/on/host:/data ...
```

---

## Disable Background Ingest

By default the scheduler starts on boot. To run the server in query-only mode (useful in CI, testing, or when ingest happens elsewhere):

```bash
docker run -d --name elko-fatcat \
  -p 8083:8083 \
  -v elko-fatcat-data:/data \
  jsoprych/elko-fatcat-mcp:latest \
  serve --no-sync
```

---

## Container Management

```bash
docker stop elko-fatcat           # stop
docker start elko-fatcat          # start
docker restart elko-fatcat        # restart
docker logs elko-fatcat           # view logs
docker logs -f elko-fatcat        # follow logs

# Remove container (data volume preserved)
docker rm elko-fatcat

# Full removal including data
docker rm elko-fatcat && docker volume rm elko-fatcat-data
```

---

## GHCR Alternative

Images are also published to GitHub Container Registry:

```bash
docker run -d --name elko-fatcat \
  -p 8083:8083 \
  -v elko-fatcat-data:/data \
  ghcr.io/jsoprych/elko-fatcat-mcp:latest
```


---

*Built by [Elko.AI](https://elko.ai) & [DARKFABRIK.AI](https://darkfabrik.ai) · [dev@elko.ai](mailto:dev@elko.ai)*
