# Privacy & data handling

This covers both ways of running the gateway: self-hosted (you run it for your own account) and the planned hosted instance (you authorize your account on a deployment I operate). Self-hosters control their own instance; the defaults below are what ships.

## What gets stored

- **Reddit OAuth tokens.** Encrypted at rest (AES-GCM). On the hosted instance each user's tokens live in their own vault entry. No shared credentials, no cross-user access.
- **A short-lived content cache.** Posts and comments fetched from the API are cached for hours, not days, then expire.
- **Embeddings.** Vector representations of ingested posts, for semantic search, clustering and community matching. Pruned on a schedule; re-syncs drop anything deleted on Reddit.
- **A write audit log.** For every write: timestamp, type, target, and a hash of the content the user approved. Kept so there's an answer to "what did this app ever post," not for analysis.
- **Minimal request logs.** Timestamps and tool names for debugging and rate-limit accounting. No content in logs.

## How writes work

Nothing is posted automatically. A write tool produces a preview; the user confirms that exact text; only then does the gateway submit it. The gateway also enforces write budgets (default 5/hour, 20/day), a cooldown between posts, and a refusal to submit near-identical content to more than one subreddit. That last rule applies to sharing my own projects too: each community gets its own distinct post, checked against that subreddit's rules, confirmed individually. On the hosted instance, write features are off until a user turns them on for their own account. The scheduled-workflow side (n8n) has no access to write endpoints at all.

The app never sends private messages and never crossposts.

## What never happens

- No training. Data fetched from the Reddit API is never used to train or fine-tune any model. Models are trained on pre-existing, publicly licensed datasets (GoEmotions, Jigsaw, SAMSum) before they ever see live content.
- No per-user analysis. Sentiment, emotion, thread-health and topic outputs are aggregates about content: a thread, a subreddit, a time window. The system does not score, profile, or characterize individual Reddit users, and does not infer sensitive characteristics (health, politics, orientation, etc.) about anyone.
- No selling, sharing, or redistributing Reddit data. Output goes to the user who asked, with links back to the original threads.

## Deletion

Disconnect your account (or revoke the app on Reddit's side) and your tokens, cached content and embeddings are deleted. The write audit log is kept for 90 days after disconnect, then deleted. Content removed on Reddit drops out of the cache at TTL expiry and out of the vector index on the next sync.

## Rate limits

Single-user instances make a few hundred read calls a day and a handful of writes. The hosted instance enforces per-user budgets plus shared caching so total usage stays within Reddit's published limits.

## Contact

Questions, or want your data removed manually: [vivianjeetsingh@gmail.com](mailto:vivianjeetsingh@gmail.com).
