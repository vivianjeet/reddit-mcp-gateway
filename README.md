# MCP Gateway for Reddit

A self-hosted [MCP](https://modelcontextprotocol.io) server that connects your Reddit account to AI assistants like Claude. Java 21 / Spring Boot 3, with n8n handling scheduled jobs and a small Python service for sentiment analysis and semantic search.

**Status: pre-release (v0).** Reddit's app review for API access is pending, so the whole thing currently runs against a mock Reddit adapter with recorded fixtures. The live adapter gets swapped in once approval comes through.

## Tools

Add the gateway as a custom connector and you get:

| Tool | What it does | Status |
|---|---|---|
| `search_posts` | search within a subreddit or site-wide | mock-backed |
| `get_my_feed` | your personalized home feed | mock-backed |
| `get_comments` | comment tree for a post | mock-backed |
| `semantic_search` | vector search over ingested posts | in progress |
| `analyze_sentiment` | aggregate sentiment for a subreddit or topic | in progress |

n8n covers the async side: subreddit watchers with sentiment-threshold alerts, daily feed digests.

## Architecture

Three parts:

- **Gateway (Spring Boot).** The MCP server itself (Streamable HTTP), tool registry, Reddit OAuth client with refresh, encrypted token storage. All Reddit calls go through a `RedditPort` interface, so the mock and live adapters are interchangeable.
- **n8n.** Ingestion, monitoring and notification workflows that hit the gateway's REST API.
- **ML service (FastAPI).** Sentiment classifier fine-tuned on [GoEmotions](https://github.com/google-research/google-research/tree/master/goemotions), sentence-transformer embeddings stored in Postgres + pgvector.

Setup guide and a proper diagram will follow with v1.

## Data handling

Personal, non-commercial. Stays well under Reddit's free-tier rate limits.

- OAuth with read-mostly scopes (`identity`, `read`, `history`, `mysubreddits`). The only account accessed is my own.
- No model training on Reddit data, ever. The classifier is trained on GoEmotions (public, licensed); live content only passes through inference when I query it.
- Short cache TTLs, embeddings pruned on a schedule, deletions on Reddit honored.
- Nothing redistributed or resold.
- Descriptive User-Agent on every request, published rate limits respected.

## Roadmap

- **v0** (now): gateway against the mock adapter — MCP server, tools, token vault, n8n workflows, docker-compose stack
- **v1**: live Reddit integration after app review, demo video
- **v1.5**: ML evals — sentiment F1 vs. a TF-IDF baseline, retrieval recall@k
- **v2**: multi-user — OAuth 2.1 resource server (PKCE, dynamic client registration), multi-tenant token vault, per-user request budgets

## Stack

Java 21 · Spring Boot 3 · Spring AI MCP · Spring Security · Postgres + pgvector · n8n · FastAPI · Hugging Face Transformers · Docker Compose

## License

MIT. See [LICENSE](LICENSE).

Reddit is a trademark of Reddit, Inc. This project isn't affiliated with or endorsed by Reddit.
