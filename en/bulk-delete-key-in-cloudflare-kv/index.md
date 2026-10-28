# Bulk Delete Keys in `Cloudflare KV`


# Bulk Delete Keys in `Cloudflare KV`

## Get the List of `key`s

Run this first

```shell
pnpm wrangler kv:key list --namespace-id=${namespaceID} > PURGE.json`
```

This gives you the complete list of `key`s; if you need a condition, just add: `--prefix="prefix"`

## Bulk Delete

```shell
cat purge.json | jq -r '.[].name' | xargs -I {} pnpm wrangler kv:key delete --namespace-id=${namespaceID} {}
```

