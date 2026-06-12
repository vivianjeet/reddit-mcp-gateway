# MCP Gateway for Reddit

A self-hosted [MCP](https://modelcontextprotocol.io) server that connects your Reddit account to AI assistants like Claude. Read your feed, catch up on replies, search semantically, research what Reddit thinks about a topic, and draft posts and replies that only go live after you confirm them. Java 21 / Spring Boot 3 for the gateway, a Python model server for the ML features, n8n for scheduled jobs.

**Status: pre-release (v0).** Reddit Data API access request is pending, so the whole thing currently runs against a mock Reddit adapter with recorded fixtures. The live adapter gets swapped in once approval comes through.

## Example asks

- "What happened on my Reddit today? Draft replies to anything that needs one."
- "Where should I share this repo? Check each community's rules and prepare a distinct post for the top three."
- "What does r/selfhosted think of Cloudflare Tunnel? Summarize the consensus."
- "Watch r/developersIndia for hiring threads mentioning Java and send me a daily digest."
- "Organize my saved posts, then answer questions from them."
- "Alert me if the tone on my r/java post turns hostile."

## Tools

| Tool | What it does | Status |
|---|---|---|
| `get_my_feed` | your personalized home feed | mock-backed |
| `get_inbox` | replies and mentions, "what happened today" | mock-backed |
| `my_activity` | your recent posts/comments and how they're doing | mock-backed |
| `search_posts` | search within a subreddit or site-wide | mock-backed |
| `get_comments` | comment tree for a post | mock-backed |
| `get_saved` | your saved posts | mock-backed |
| `subreddit_info` | rules, flairs, posting requirements | mock-backed |
| `find_communities` | shortlist subreddits where given content actually fits | in progress |
| `check_post_fit` | does this draft follow that subreddit's rules | planned |
| `semantic_search` | meaning-based search over ingested posts | in progress |
| `find_similar` / `asked_before` | related threads, near-duplicate check | in progress |
| `summarize_thread` | TL;DR of a long comment tree | planned |
| `analyze_sentiment` / `emotion_profile` | aggregate mood of a discussion | in progress |
| `topic_map` | main themes in a subreddit over a time window | planned |
| `organize_saved` | cluster your saved posts into collections | planned |
| `draft_reply` / `draft_post` / `edit_own` | prepare a write, returns preview + token | planned |
| `confirm_write` | execute a previously previewed write | planned |
| `track_post` | follow performance and replies on your own post | planned |
| `save_post` / `subscribe` | small account actions, also gated | planned |

n8n covers the async side: subreddit watchers with sentiment and trend alerts, daily digests, job-thread listening, ingestion for the vector index. The n8n side is read-only by construction.

## Write safety

Nothing posts in one step. Every write tool returns a preview and a one-time token; a separate `confirm_write` call executes it. You see the exact text in chat before it exists on Reddit. On top of that the gateway enforces:

- write budgets (default 5/hour, 20/day, configurable down) and a cooldown between posts
- a similarity check that refuses near-identical content going to multiple subreddits
- a full audit log of every write
- no DMs, no crossposting, no scheduled or autonomous writes, ever

This doubles as prompt-injection protection: text fetched from Reddit can't trick the assistant into silently posting, because there is no silent path.

## Sharing your own work

Blasting the same announcement across subreddits is banned by Reddit and refused by this gateway. What it supports instead: `find_communities` shortlists the few places a project genuinely fits, `check_post_fit` surfaces each community's self-promotion rules and flair requirements, you write a different post for each (the similarity check verifies they're actually different), and you confirm them one at a time over days. Then `track_post` and `get_inbox` help you stick around and answer comments, which is the part that makes sharing welcome.

## ML

Twelve capabilities from five pieces of machinery, so the system stays maintainable:

- **One embedding backbone** (sentence-transformer) powers semantic search, related threads, duplicate detection, saved-post clustering, BERTopic topic maps, drift-based trend detection, community matching, and the write gate's similarity check. Evals: recall@k / MRR, silhouette, topic coherence.
- **Three classifier heads**, trained only on pre-existing licensed datasets (GoEmotions, Jigsaw), never on data fetched from the Reddit API: sentiment, 27-emotion profile, thread-health score. Outputs are thread/subreddit aggregates. Eval: F1 against a TF-IDF baseline.
- **One small summarizer** (BART/T5 fine-tuned on SAMSum) with an extractive fallback. Eval: ROUGE.
- Classical algorithms fill the rest: MinHash LSH for dedup, burst detection for trends, KeyBERT + NER for entity trends.

Deliberately not built: engagement prediction (would need training on Reddit-derived labels, which the Data API terms prohibit), per-user scoring or profiling of any kind, and inference of sensitive characteristics. See PRIVACY.md.

## Architecture

Three parts:

- **Gateway (Spring Boot).** The MCP server itself (Streamable HTTP), tool registry, the write gate, Reddit OAuth client with refresh, encrypted token storage. All Reddit calls go through a `RedditPort` interface, so the mock and live adapters are interchangeable.
- **n8n.** Read-only ingestion, monitoring and digest workflows against the gateway's REST API.
- **Model server (FastAPI).** The embedding backbone, classifier heads and summarizer, with vectors in Postgres + pgvector.

Setup guide and a proper diagram will follow with v1.

## Hosted deployment (v2)

Same codebase, `multi` Spring profile. Each user authorizes their own Reddit account via OAuth; tokens are encrypted and isolated per user; per-user request budgets and shared caching keep the instance inside Reddit's rate limits; write features are off by default and opt-in per user, behind the same gate. Disconnecting deletes your tokens, cached content and embeddings. Inbound auth from AI clients follows the MCP OAuth 2.1 spec (PKCE, dynamic client registration). Details in [PRIVACY.md](PRIVACY.md).

## Data handling

Personal, non-commercial. Stays well under Reddit's free-tier rate limits.

- OAuth scopes: identity, read, history, mysubreddits, privatemessages (inbox read only, never sending), submit, edit, save, subscribe, vote (off by default), flair.
- No model training on data fetched from the Reddit API, ever. Models are trained only on pre-existing public datasets under their own licenses.
- Analysis outputs are aggregates about content, never assessments of individual users.
- Short cache TTLs, embeddings pruned on a schedule, deletions on Reddit honored.
- Nothing redistributed or resold. Descriptive User-Agent, published rate limits respected.

## Roadmap

- **v0** (now): gateway against the mock adapter — MCP server, read tools, write gate, token vault, n8n workflows, docker-compose stack
- **v1**: live Reddit integration after the access request is approved, demo video
- **v1.5**: ML layer complete with published evals (F1, recall@k, ROUGE, topic coherence)
- **v2**: hosted multi-user deployment — OAuth 2.1 resource server, multi-tenant token vault, per-user budgets and write opt-in

## Stack

Java 21 · Spring Boot 3 · Spring AI MCP · Spring Security · Postgres + pgvector · n8n · FastAPI · Hugging Face Transformers · sentence-transformers · BERTopic · Docker Compose

## Contact

[vivianjeetsingh@gmail.com](mailto:vivianjeetsingh@gmail.com)

## License

MIT. See [LICENSE](LICENSE).

Reddit is a trademark of Reddit, Inc. This project isn't affiliated with or endorsed by Reddit.
