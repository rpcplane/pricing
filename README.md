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

## Included providers

| Provider | Model | $/M std call | $/M getBlock | $/M DAS | Source date |
|---|---|---|---|---|---|
| Helius | credits | $5.00 | $50.00 (10×) | $50.00 (10×) | 2026-09-05 |
| QuickNode | credits | $15.00 | $15.00 (no premium) | $30.00 (2×) | 2026-09-05 |
| Triton One | req + bandwidth | $10.00 + BW | $10.00 + BW | $50.00 + BW | 2026-09-05 |
| Alchemy | compute units | $4.50 | $18.00 (4×) | — | 2026-09-05 |
| Chainstack | request units | $5.00 | $10.00 (archive 2×) | — | 2026-05-14 |

BW = $0.08/GB response bandwidth (Triton).

Alchemy's CU weights are no longer marked unverified: Alchemy now publishes a Solana-specific compute
unit table and it confirms the values already in `pricing.json` exactly (standard 10, sendTransaction /
getProgramAccounts / getSlot 20, history 40).

Standard call reference = `getAccountInfo`. DAS = Metaplex Digital Asset Standard (`getAsset`, `getAssetsByOwner`, etc.).

## Pending providers

These providers were checked but excluded because per-method pricing is not publicly documented. Add them when the data becomes available.

| Provider | Status | What's missing | Where to check |
|---|---|---|---|
| GetBlock | Has CU model, $0.41/M CU (Pro tier) | CU weight per Solana method not documented | https://getblock.io/pricing/ |
| Ankr | Flat $50/M all Solana, no differentiation | No per-method breakdown | https://www.ankr.com/rpc/pricing/ |
| Shyft | Flat-rate RPS subscriptions, no per-call metering | No PAYG model currently | https://shyft.to/solana-rpc-grpc-pricing |
| Syndica | Enterprise/contact-sales only | No public pricing page | https://syndica.io (has cost estimator) |

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
