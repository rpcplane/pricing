# rpcplane/pricing

Provider pricing data for [RPC Plane](https://rpcplane.dev). Served as a static JSON file at `pricing.rpcplane.dev`.

**Scope:** this is reference data, not a routing input. Cost-aware routing is on hold and does not
exist in `rpc-plane` — the binary does not fetch this file. It backs the human-readable comparisons at
[docs.rpcplane.dev/provider-pricing](https://docs.rpcplane.dev/provider-pricing/) and the calculator at
[rpcplane.dev/provider-pricing](https://rpcplane.dev/provider-pricing/), and is public so anyone can use it.

## Files

| File | Description |
|---|---|
| `pricing.json` | Live pricing data, schema v1 |

## Schema

```jsonc
{
  "schema_version": "1",
  "updated_at": "YYYY-MM-DD",
  "methodology": "...",
  "providers": {
    "<key>": {
      "display_name": "...",
      "url_patterns": ["*.example.com"],   // matched against configured provider URLs
      "model": "credits | compute_units | request_units | requests_plus_bandwidth",
      "overage_usd_per_million_credits": 0.0,  // field name varies by model
      "bandwidth_usd_per_gb": null,             // present when model = requests_plus_bandwidth
      "pricing_url": "...",                     // provider's public pricing page
      "method_credits_source": "...",           // where per-method weights were sourced
      "method_credits": {
        "default": 1,                           // weight for any unlisted method
        "<method_name>": 10                     // weight for specific expensive methods
      },
      "updated_at": "YYYY-MM-DD",
      "notes": "..."
    }
  }
}
```

**Effective cost per call:**
```
cost_usd = (method_credits[method] / 1_000_000) × overage_usd_per_million_<unit>
```

For `requests_plus_bandwidth` providers, add `response_bytes / 1_073_741_824 × bandwidth_usd_per_gb` per call.

**Check `unpriced_methods` before falling back to `default`.** A method listed there has no rate in
this file — either the provider does not serve it, or it does not publish a price. Four providers list
the Metaplex DAS methods, and OnFinality also lists `sendTransaction`. Treating those as `default`
would report a confident price for a call the provider cannot bill you for.

## Included providers

| Provider | Model | $/M std call | $/M getBlock | $/M DAS | Source date |
|---|---|---|---|---|---|
| Alchemy | compute units | $4.50 | $18.00 (4×) | — | 2026-09-05 |
| Helius | credits | $5.00 | $50.00 (10×) | $50.00 (10×) | 2026-09-05 |
| dRPC | compute units | $6.00 | $6.00 (no premium) | — | 2026-09-06 |
| OnFinality | request units | $7.50 | $7.50 (no premium) | — | 2026-09-06 |
| Chainstack | request units | $10.00 | $10.00–$20.00 (archive 2×) | — | 2026-09-06 |
| Triton One | req + bandwidth | $10.00 + BW | $10.00 + BW | $50.00 + BW | 2026-09-05 |
| QuickNode | credits | $15.00 | $15.00 (no premium) | $30.00 (2×) | 2026-09-05 |

BW = $0.08/GB response bandwidth (Triton).

Chainstack's `getBlock` is a range because its Solana archive rule is slot-based: 1 RU near the chain
tip, 2 RU only below `firstAvailable + 5,000`. Live indexing pays the lower figure; backfill pays the
higher one.

**Rates are each provider's best published non-enterprise rate**, per the `methodology` field. That
is not always the entry tier: Chainstack's $10.00 is Business ($499/mo), not Growth ($49, $15.00/M),
and QuickNode's $15.00 is Business, not Build. Each provider's `notes` carries its full tier ladder.

Alchemy's CU weights are no longer marked unverified: Alchemy now publishes a Solana-specific compute
unit table and it confirms the values already in `pricing.json` exactly (standard 10, sendTransaction /
getProgramAccounts / getSlot 20, history 40).

Standard call reference = `getAccountInfo`. DAS = Metaplex Digital Asset Standard (`getAsset`, `getAssetsByOwner`, etc.).

## Pending providers

A provider is included if its published material prices **every** method — that can be a per-method
weight table (Alchemy, Helius, QuickNode) or a documented flat rate (dRPC, OnFinality, Chainstack). It
is excluded if any part of the method surface is unpriced, because a partial table invites the exact
mistake this dataset exists to prevent: costing the calls that are published and assuming the rest
follow. Add one when the data becomes available.

| Provider | Status | What's missing | Where to check |
|---|---|---|---|
| GetBlock | Has CU model, $0.41/M CU (Pro tier) | CU weight per Solana method not documented | https://getblock.io/pricing/ |
| Ankr | Flat $50/M all Solana, no differentiation | No per-method breakdown | https://www.ankr.com/rpc/pricing/ |
| Shyft | Flat-rate RPS subscriptions, no per-call metering | No PAYG model currently | https://shyft.to/solana-rpc-grpc-pricing |
| Syndica | Free tier published (10M req/mo, 100 RPS, 100 GB) | Paid tiers quoted per plan; no per-request or per-method rate | https://syndica.io/pricing |
| RPC Fast | Standard Solana call = 1 CU, overage $5.00/M CU | No published weight for archive, history or DAS — a partial table | https://rpcfast.com/pricing |
| Blockdaemon | 3M CU free, Starter $600/mo for 15M CU | No published per-method CU weights | https://www.blockdaemon.com/pricing |
| Uniblock | Aggregation layer, not a node operator | Routes to upstream providers, so its cost is theirs plus margin; no per-method rate | https://uniblock.dev |

Checked 2026-09-06: Syndica, RPC Fast, Blockdaemon, Uniblock. Checked 2026-05-14: GetBlock, Ankr, Shyft.

## Updating prices

Prices are sourced directly from each provider's public pricing pages. Check and update **quarterly** (January, April, July, October).

For each provider:
1. Visit `pricing_url` — update `overage_usd_per_million_<unit>` if the rate changed.
2. Visit `method_credits_source` — verify method weight table matches `method_credits` entries.
3. Update `updated_at` on the provider entry and the top-level `updated_at`.
4. Open a PR — changes auto-deploy to `pricing.rpcplane.dev` on merge.

The rendered site is `index.html`. It is aimed at a different query than the two `/provider-pricing/`
pages — those own "compare provider pricing" and "price per method"; this one owns the dataset itself,
for someone who wants Solana RPC pricing as data to fetch rather than a table to read. Keep it that
way: do not paste the rate tables into it, or all three pages end up competing.

## Contributing

PRs to add missing providers or fix incorrect weights are welcome. Please include the source URL and date in your PR description.
