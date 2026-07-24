# MVP — Critical Path

Web-first. Solo founder. Ship the thinnest end-to-end version a real user could use to find a gift and generate a merchant click. Everything not on this path is deferred.

Last updated: 2026-07-23.

## Definition of done

A visitor lands on the web feed, sees a mix of AI-generated and user-created collections drawn from a multi-merchant catalog, drills into a collection, and clicks through to a merchant. Registered users can publish their own collections back into the feed.

## The three pillars

### 1. Multi-merchant catalog (source)
**Today:** only El Corte Inglés is ingested.
**MVP:** ≥3 AWIN merchants across ≥2 categories, Layer A giftability filter applied at ingest, product embeddings computed at ingest.

### 2. Dynamic feed (surface)
**Today:** static AI collections seeded once at boot.
**MVP:** a feed that continuously mixes freshly-generated AI collections and user-published collections, ordered by recency + a light engagement signal.

### 3. User-published collections (loop)
**Today:** collections exist per-user in Firestore but aren't publicly visible.
**MVP:** registered users can create a collection, publish it to the feed, and see basic engagement (views / saves).

## User journeys (web only)

- **Anonymous:** feed → collection → product → merchant.
- **Registered:** sign in → create occasion → AI recommendations → optionally publish a curated selection to the feed.

## Punch list (execution order)

**Catalog — Pillar 1**
1. Land `feature/awin-feed-sync-mvp` in backend to `develop` (multi-merchant orchestrator, feed source, V5 migration, Product entity changes). Blocks everything downstream.
2. Wire Layer A giftability filter (`docs/core/giftability-rules.md`) into `ProductSyncService`.
3. Compute product embeddings at ingest inside `ProductSyncService` — currently missing; recommendations quality degrades without it.
4. Seed `merchant_programmes` with 2 additional merchants beyond ECI: Bikila (queued from 2026-07-10) + one from a different category (Tech or Home).
5. Coverage gate: verify a minimum product count across the canonical categories before Pillar 2 lights up (define threshold when Pillar 1 items 1–4 land).

**Feed — Pillar 2**
6. Feed ordering algorithm v1: recency + engagement weight. Owned by `recommendations-specialist`.
7. Feed API endpoint (paginated) + Flutter web feed screen.
8. Replace `CollectionStartupSeeder` boot-time generation with a scheduled generator producing new AI collections continuously (cron; frequency TBD).

**User collections — Pillar 3**
9. Publish/unpublish flow for user collections; light moderation floor (block/report already exists).
10. Feed inclusion rule for user collections (simplest v1: 1-in-N slots reserved for user collections; N configurable).

**Web deployment**
11. Firebase Hosting for Flutter web build — stabilise existing local setup, open a public URL.

## Explicitly deferred (do not work on until MVP ships)

- iOS/Android app-store submission
- Push notifications
- Amazon PA-API adapter (waiting on sales-velocity qualification)
- Fingerprint dedup / cross-merchant grouping
- 30-day purge job
- Browser extension (blocked pending legal review)
- Recommendation-quality A/B testing infrastructure
- Follow/unfollow social graph (already removed)

## How to use this doc

- Re-read the punch list before starting any work session.
- If a task doesn't ladder up to one of the three pillars, it's deferred.
- Update this doc when an item is done — move to `## Shipped` at the bottom rather than deleting, so the trajectory is visible.
- When the whole MVP is shipped, this doc collapses into `docs/decisions.md` as one entry and gets archived.
