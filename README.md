# Svalin Index

**Is the price real?** Every NEAR token with a Rhea Finance price, NEAR and USDC included, graded A
to F by the real money behind it, with 30 days of price and backing per token. Rebuilt every few
hours.

See it at **[svalin.noobshole.workers.dev/ratings](https://svalin.noobshole.workers.dev/ratings)**, or
read it as JSON from the [Svalin API](https://svalin-api.noobshole.workers.dev/v1/index/near-tokens).

## How grades work

A quoted price is only a quote. Anyone can send a wallet a token whose price is fake or stale, and a
tool or AI agent that trusts it sees a fortune that cannot be sold.

Svalin grades a token by one question: **how much of a $1,000 sale would be lost to price
movement?** That depends on the token's **real backing**: the money in its Rhea Finance pools,
classic and concentrated liquidity, held in tokens Svalin trusts, such as NEAR, USDC, USDT and
liquid-staked NEAR.
Selling a token can never take out more than that, and two unvetted tokens paired with each other
add nothing.

Backing has to **hold for 24 hours** before it counts, and a token is graded on its lowest backing in
that time. A pool added this morning cannot lift a grade today, and a pool pulled for an hour still
lowers it.

| Grade | Real backing | A $1,000 sale loses |
| --- | --- | --- |
| A | $1M or more | under 0.1% |
| B | $100K or more | under 1% |
| C | $20K or more | under 5% |
| D | $3K or more | 25% or less |
| F | under $3K | more than 25% |

Grades describe how sellable a token is, not whether its project is good. They are not financial
advice. Since 26 November 2026 the trusted tokens are graded too, on the other trusted tokens in
their pools and never on themselves. Only Rhea Finance pools are counted so far. Editions from
22 October 2026 use methodology `near-token-backing-v1`; earlier ones are `near-token-backing-v0`
(classic pools only, no 24-hour hold) and still verify with current code.

## Don't trust it, verify it

Every grade can be recomputed by anyone. Each edition archives the exact Rhea Finance data it was
computed from (Ref Finance's before 26 November 2026) and the backing history the 24-hour rule was
applied from, and records their SHA-256 hashes, the hash of the index file and the Svalin code
commit. Its **fingerprint**, the hash of that record, changes if a single grade, input or date is
changed. Once a day the edition is sealed into [`editions/`](editions), which is never rewritten.

To check an edition yourself:

```bash
git clone https://github.com/noobshole/svalin-agent && cd svalin-agent
git checkout <code.commit from edition.json>
pnpm install && pnpm build
git clone --branch latest https://github.com/noobshole/svalin-index ../svalin-index
pnpm --filter @svalin-agent/api run index:verify ../../svalin-index
```

For a sealed edition, point the last command at `../../svalin-index/editions/<day>` on `main`
instead. It prints `Verified` when every hash matches and all grades are reproduced; otherwise it
lists what differs. Market cap, holder data, symbols read from token metadata and `notes` come from
FastNEAR and NearBlocks, so they are pinned by the index hash but not recomputed.

## Files

The [`latest`](https://github.com/noobshole/svalin-index/tree/latest) branch holds the current
edition:

- `near-token-index.json`: `generatedAt`, `methodology`, grade `counts`, `rows`, one per token, and
  from 4 November 2026 `notes`:

  ```text
  [contract, symbol, priceUsd, backingUsd, grade, marketCapUsd, top10Pct, holders]
  ```

  `symbol` is Rhea Finance's, or the token's own (`ft_metadata`) when Rhea has none.
  `marketCapUsd` is the quoted price times total supply. `top10Pct` is the share of supply held by
  the ten largest holders outside exchange pools, bridges, NEAR Intents and burn addresses.
  `holders` comes from NearBlocks, or is a lower bound from FastNEAR's list of the largest holders
  where NearBlocks has no count. All three are `null` for tokens without backing, except the market
  cap of the highest-priced ones. Before 4 November 2026 a `holders` of 0 means unknown.

  `notes` maps a contract to what a row cannot say: `holdersAtLeast` (the count is a lower bound),
  and `selfDeclaredTest` with the token's on-chain `name` when that name says it is a test or not to
  be bought. Neither changes a grade.
- `edition.json`: the sealed record: hashes of the inputs and the index, the code commit and the
  fingerprint.
- `inputs/prices.json.gz` and `inputs/pools.json.gz`: the exact Rhea Finance responses (Ref
  Finance's indexer before 26 November 2026).
- `inputs/dcl-pools.json.gz`: every concentrated-liquidity pool on `dclv2.ref-labs.near`, with the
  fields the grades use.
- `inputs/history.json.gz`: each token's backing over the last 24 hours, which the grades are
  taken from.
- `near-dcl-depth.json`: the backing each token gets from concentrated-liquidity pools, which the
  Svalin API adds when valuing wallets.
- `near-token-stats.json`: total supply, holder data and token metadata, refreshed a few tokens per
  run: those graded D or better about daily, the rest over about ten days.
- `near-token-daily.json`: one point a day per token for the last 30 days, its quoted price and the
  backing its grade used, which the ratings page draws as charts. It is built from the editions,
  is not part of the sealed edition and changes no grade.
- `near-token-tree.bin`: the edition's Merkle tree, every level from the leaves up, 32 bytes a node.
  The leaf for a token is `svalin-index/1|<day>|<token>|<grade>|<backing_usd>`. Svalin's API serves
  proofs from it (`/v1/proof/{token}`).
- `near-token-root.json`: the tree's root and the edition's fingerprint, as published on NEAR.

`editions/<day>/` on `main` holds each day's sealed edition: `edition.json`, the index and its
inputs, and from 5 November 2026 the tree and root. The roots of earlier editions were put on NEAR
from those sealed files.

## The v2 trial

Methodology `near-token-backing-v2` runs beside v1 as a trial from 1 December 2026 (UTC). v1 stays
the published grade, and the only one put on NEAR, until v2 replaces it. v2 keeps v1's grade bands, trusted tokens, prices
and 24-hour hold, and changes three things:

- **Every Rhea pool, read from NEAR at one block.** v2 reads all of the pools on
  `v2.ref-finance.near` and `dclv2.ref-labs.near` at one final block (9,024 classic pools on
  27 November 2026, against the 3,175 in Rhea's list), and records the block in `edition.json`.
- **Why each grade changed.** `changes` in the index file lists every grade that changed since the
  latest sealed edition before that day, with one reason: `price` (mostly NEAR's price moved),
  `money-out`, `money-in`, `price-and-money`, or `method`. `inputs/backing-detail.json.gz` keeps
  the trusted tokens behind every point of the backing history, so the reason is exact even inside
  the 24-hour hold. Where it does not reach back far enough (the trial's first days), the reason is
  marked `estimated`.
- **A second opinion.** `near-intents-quotes.json` gives what NEAR Intents would pay in USDC for
  $1,000 of each token it lists (dry quotes: nothing is signed or sent), flagged when that is two
  grade bands or more from ours. It is not part of the grade or of the sealed edition.

The trial is published on the
[`latest-v2`](https://github.com/noobshole/svalin-index/tree/latest-v2) branch, with the same files
as `latest` plus the Intents quotes, the backing detail and `near-methodology-trial.json`, which
compares each run's v1 and v2 grades. Each day's trial edition is sealed into `editions-v2/<day>/`
on `main`. The Svalin API serves it at `/v1/index/near-tokens?methodology=near-token-backing-v2`.
Verifying a v2 edition can
also recompute its reasons from the sealed edition they are measured from (the day in
`changes.since`, under `editions/` on the trial's first day), and read its pools again from a NEAR
archive node at the recorded block:

```bash
git clone --branch latest-v2 https://github.com/noobshole/svalin-index ../svalin-index-v2
git clone https://github.com/noobshole/svalin-index ../svalin-index-record
pnpm --filter @svalin-agent/api run index:verify ../../svalin-index-v2 --previous ../../svalin-index-record/editions-v2/<day> --chain
```

The method is in
[RISK_METHODOLOGY.md](https://github.com/noobshole/svalin-agent/blob/main/docs/RISK_METHODOLOGY.md#the-v2-trial-near-token-backing-v2).

## How it is built

[`.github/workflows/build.yml`](.github/workflows/build.yml) is scheduled every hour; GitHub runs it every few hours. It checks out the
[Svalin code](https://github.com/noobshole/svalin-agent), runs
[`apps/api/scripts/near-index.ts`](https://github.com/noobshole/svalin-agent/blob/main/apps/api/scripts/near-index.ts)
against Rhea Finance, FastNEAR and NearBlocks, verifies the result with
[`verify-index.ts`](https://github.com/noobshole/svalin-agent/blob/main/apps/api/scripts/verify-index.ts),
and publishes it to `latest`. The grading code is open and deterministic: the same inputs always
give the same grades.

The job that runs code can only read; a second job with write access only copies the verified
edition and pushes it. When a day's edition is sealed, a third job puts its root on NEAR, in the
Svalin Ratings registry `svalin-ratings-v1.testnet`
([`register-root.ts`](https://github.com/noobshole/svalin-agent/blob/main/apps/api/scripts/register-root.ts)).
It signs with a key that can call nothing but the registry's `publish`, and has no write access to
this repository. A new root can be checked against the sealed files during the registry's 1-hour
challenge window, and retracted if it is wrong. That job installs its packages with their install
scripts off and builds nothing. Third-party actions are pinned to exact commits. The v2 trial is
built and published by two more jobs, split the same way. v1 never waits for them, and nothing of
v2 is put on NEAR.

## License

Apache License 2.0. See [LICENSE](LICENSE). Please credit Svalin when you use the data.
