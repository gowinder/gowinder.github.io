# 批量删除`Cloudflare KV`中的`key`


# 批量删除`Cloudflare KV`中的`key`

## 取得`key`列表

先执行

```shell
pnpm wrangler kv:key list --namespace-id=${namespaceID} > PURGE.json`
```

拿到完整`key`列表, 如果要条件,就加上: `--prefix="prefix"`

## 批量删除

```shell
cat purge.json | jq -r '.[].name' | xargs -I {} pnpm wrangler kv:key delete --namespace-id=${namespaceID} {}
```


