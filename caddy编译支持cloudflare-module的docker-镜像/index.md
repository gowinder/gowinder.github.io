# caddy编译支持cloudflare module的docker 镜像


# `caddy`编译支持`cloudflare module`的`docker` 镜像

## `caddy`使用`cloudflare`的`dns`时,会报错

出现: `Error: adapting config using caddyfile: parsing caddyfile tokens for 'tls': /etc/caddy/Caddyfile:3 - Error during parsing: getting module named 'dns.providers.cloudflare': module not registered: dns.providers.cloudflare`,查了下,需要自已编辑`module`

## Dockerfile

```dockerfile
FROM caddy:2.6.3-builder-alpine AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:2.6.3-alpine

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

## Caddyfile

```Caddyfile
(cors) {
        @origin{args.0} header Origin {args.0}
        header @origin{args.0} Access-Control-Allow-Origin "{args.0}"
}

{
        "module": "acme",
        "challenges": {
                "dns": {
                        "provider": {
                                "name": "cloudflare",
                                "api_token": "{env.CLOUDFLARE_DNS_API_TOKEN}"
                        }
                }
        }
}
yourdomain {
        tls {
                dns cloudflare
        }
        root * /var/www/html
        # Enable the static file server.
        file_server
        header Access-Control-Allow-Methods "POST, GET, OPTIONS, PUT"
        header Access-Control-Allow-Headers "*"
}
```

```Caddyfile
gowinder.work {
    log {
        output file /etc/caddy/caddy.log
    }
    tls {
        protocols tls1.2 tls1.3
        ciphers TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
        curves x25519
    }
    root * /var/www/html
    # Enable the static file server.
    file_server
    header Access-Control-Allow-Methods "POST, GET, OPTIONS, PUT"
    header Access-Control-Allow-Headers "*"
}
```

