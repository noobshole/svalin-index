# Svalin Index

**Is the price real?** Every NEAR token with a Ref Finance price, graded A to F by the real money
behind it. Rebuilt every few hours.

See it at **[svalin.noobshole.workers.dev/ratings](https://svalin.noobshole.workers.dev/ratings)**, or
read it as JSON from the [Svalin API](https://svalin-api.noobshole.workers.dev/v1/index/near-tokens).

## How grades work

A quoted price is only a quote. Anyone can send a wallet a token whose price is fake or stale, and a
tool or AI agent that trusts it sees a fortune that cannot be sold.

Svalin grades a token by one question: **how much of a $1,000 sale would be lost to price
movement?** That depends on the token's **real backing**: the money in its Ref Finance pools held in
tokens Svalin trusts (NEAR, USDC, USDT and liquid-staked NEAR). Selling a token can never take out
more than that, and two unvetted tokens paired with each other add nothing.

| Grade | Real backing | A $1,000 sale loses |
| --- | --- | --- |
| A | $1M or more | under 0.1% |
| B | $100K or more | under 1% |
| C | $20K or more | under 5% |
| D | $3K or more | under 25% |
| F | under $3K | 25% or more |

Grades describe how sellable a token is, not whether its project is good. They are not financial
advice. The trusted tokens are the yardstick and are not graded. Only Ref Finance's classic pools
are counted so far.

## Don't trust it, verify it

Every grade can be recomputed by anyone. Each edition archives the exact Ref Finance responses it
was computed from and records their SHA-256 hashes, the hash of the index file, and the Svalin code
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
lists what differs. Market cap and holder data come from FastNEAR and NearBlocks, so they are pinned
by the index hash but not recomputed.

## Files

The [`latest`](https://github.com/noobshole/svalin-index/tree/latest) branch holds the current
edition:

- `near-token-index.json`: `generatedAt`, `methodology`, grade `counts`, and `rows`, one per token:

  ```text
  [contract, symbol, priceUsd, backingUsd, grade, marketCapUsd, top10Pct, holders]
  ```

  `marketCapUsd` is the quoted price times total supply. `top10Pct` is the share of supply held by
  the ten largest holders outside exchange pools, bridges and NEAR Intents. `holders` comes from
  NearBlocks. All three are `null` for tokens without backing, except the market cap of the
  highest-priced ones.
- `edition.json`: the sealed record: hashes of the inputs and the index, the code commit and the
  fingerprint.
- `inputs/prices.json.gz` and `inputs/pools.json.gz`: the exact Ref Finance responses.
- `near-token-stats.json`: total supply and holder data, refreshed about once a day per token.

`editions/<day>/` on `main` holds each day's sealed edition with the same files.

## How it is built

[`.github/workflows/build.yml`](.github/workflows/build.yml) is scheduled every hour; GitHub runs it every few hours. It checks out the
[Svalin code](https://github.com/noobshole/svalin-agent), runs
[`apps/api/scripts/near-index.ts`](https://github.com/noobshole/svalin-agent/blob/main/apps/api/scripts/near-index.ts)
against Ref Finance, FastNEAR and NearBlocks, verifies the result with
[`verify-index.ts`](https://github.com/noobshole/svalin-agent/blob/main/apps/api/scripts/verify-index.ts),
and publishes it to `latest`. The grading code is open and deterministic: the same inputs always
give the same grades.

## License

Apache License 2.0. See [LICENSE](LICENSE). Please credit Svalin when you use the data.
