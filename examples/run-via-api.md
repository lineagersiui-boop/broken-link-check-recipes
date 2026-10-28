# Run via the Apify API

`input.json` is one of the example inputs with your own site in it. Only check sites you own or have permission to check; the Actor refuses runs without `"confirmPermission": true`. `APIFY_TOKEN` is your Apify API token; prefer a dedicated one for scripts.

## Small sitemaps: one synchronous call
For this call, add `"timeLimitSecs": 240` to `input.json`. The Actor then stops starting new checks after 240 seconds and finishes normally, with its results and `SUMMARY`. The call waits for the run and returns the dataset items (one per URL) as a JSON array:

```bash
curl -sS --fail -X POST \
  -H "Authorization: Bearer $APIFY_TOKEN" -H "Content-Type: application/json" \
  --data @input.json \
  "https://api.apify.com/v2/acts/lineagersiui~sitemap-url-health-checker/run-sync-get-dataset-items?timeout=280" \
  -o results.json
```

`timeout=280` is only a backstop: it kills the run after 280 seconds, which then ends `TIMED-OUT` without a `SUMMARY`, and the call may not return the items. `timeLimitSecs` is the clean way to stop. URLs that weren't reached aren't checked or charged either way. The synchronous endpoint waits only about 5 minutes. Requests to one site are at least 500 ms apart, so this suits sitemaps of a few hundred URLs at most. For bigger ones, use the asynchronous run below.

List only the broken URLs with `jq`:

```bash
jq -r '.[] | select(.flags | index("not_found") or index("server_error") or index("client_error")) | "\(.status)\t\(.url)"' results.json
```

## Larger sitemaps: start, poll, read
1. Start a run. The response's `data` has the run `id`, `defaultDatasetId` and `defaultKeyValueStoreId`:
   ```bash
   curl -sS --fail -X POST -H "Authorization: Bearer $APIFY_TOKEN" -H "Content-Type: application/json" \
     --data @input.json "https://api.apify.com/v2/acts/lineagersiui~sitemap-url-health-checker/runs"
   ```
2. Poll until `data.status` is `SUCCEEDED`, `FAILED`, `ABORTED` or `TIMED-OUT`. Each call waits up to 60 seconds:
   ```bash
   curl -sS -H "Authorization: Bearer $APIFY_TOKEN" "https://api.apify.com/v2/actor-runs/RUN_ID?waitForFinish=60"
   ```
3. Read the results and the totals:
   ```bash
   curl -sS -H "Authorization: Bearer $APIFY_TOKEN" "https://api.apify.com/v2/datasets/DATASET_ID/items?clean=true&format=json"
   curl -sS -H "Authorization: Bearer $APIFY_TOKEN" "https://api.apify.com/v2/key-value-stores/KEY_VALUE_STORE_ID/records/SUMMARY"
   ```
   Add `format=csv` instead of `format=json` for a spreadsheet.

[`github-actions/link-check.yml`](github-actions/link-check.yml) does these steps in Node, without `jq`.

## Free demo
Send `{}` as the input to run the free demo on 20 URLs of apify.com. It isn't billed and shows the output format.

Apify's API reference: https://docs.apify.com/api/v2
