# Migrating a Remix Project to Cloudflare


# Migrating a Remix Project to Cloudflare

Recently I developed a project using `remix`. I hadn't done full-stack development before; I'd used `reactjs` a bit but wasn't very familiar with it, and I was using `Nextjs`, which felt very hard to use — mainly because after version `13` switched to the `app router`, I was somewhat at a loss.

Later I discovered `remix`, which uses filename-based routing directly and is much more convenient, and the `loader` and `action` approach is also very intuitive, so I chose `remix`.

Once the project was mostly developed, I thought about how to deploy it. I had originally planned to use `Vercel`, but one day I saw `https://blog.meathill.com/infra/vercel-vs-cloudflare-pages-how-to-choose-to-deploy-your-website.html` and still felt `cloudflare` was impressive (good for free riding).

So I prepared to migrate to `cloudflare`.

## Steps

`remix` supports creating a project that deploys directly to `cloudflare`, but mine is already a project in development, so I can't create it from the template.

### Copy files

First create a `cloudflare` template project with `remix`: `pnpm create cloudflare@latest my-remix-app -- --framework=remix`

After creating it, compare it with my own project:
![project](remix-proj.png)
These extra files appeared; copy them over.

### Install libraries

```
pnpm add -d vite @cloudflare/workers-types autoprefixer postcss vite-tsconfig-paths wrangler
pnpm add @remix-run/cloudflare @remix-run/cloudflare-pages
```

### File modifications

#### Add to `tsconfig.json`:

```JSON
"compilerOptions": {
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "types": [
      "@remix-run/cloudflare",
      "vite/client",
      "@cloudflare/workers-types/2023-07-01"
  ],
```

#### wrangler.toml

Configure `wrangler.toml` as needed — configure whatever you use. I used `d1_database`, `r2_buckets`, etc.

#### postcss.config.cjs[js]

```javascript
module.exports = {
    plugins: {
      tailwindcss: {},
      autoprefixer: {},
    },
  };
```

#### tailwind.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

export default {}
```

#### entry.server.tsx

```tsx
import type { AppLoadContext, EntryContext } from "@remix-run/cloudflare";
import { RemixServer } from "@remix-run/react";
import { isbot } from "isbot";
import { renderToReadableStream } from "react-dom/server";

export default async function handleRequest(
  request: Request,
  responseStatusCode: number,
  responseHeaders: Headers,
  remixContext: EntryContext,
  // This is ignored so we can keep it in the template for visibility.  Feel
  // free to delete this parameter in your app if you're not using it!
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  loadContext: AppLoadContext
) {
  const body = await renderToReadableStream(
    <RemixServer context={remixContext} url={request.url} />,
    {
      signal: request.signal,
      onError(error: unknown) {
        // Log streaming rendering errors from inside the shell
        console.error(error);
        responseStatusCode = 500;
      },
    }
  );

  if (isbot(request.headers.get("user-agent") || "")) {
    await body.allReady;
  }

  responseHeaders.set("Content-Type", "text/html");
  return new Response(body, {
    headers: responseHeaders,
    status: responseStatusCode,
  });
}
```

#### root.tsx

```tsx
import { cssBundleHref } from '@remix-run/css-bundle';
import type { LinksFunction } from '@remix-run/node';
import {
  Links,
  LiveReload,
  Meta,
  Outlet,
  Scripts,
  ScrollRestoration,
} from '@remix-run/react';

import styles from './tailwind.css?url';
import { ThemeProvider } from './context/themeContext';

// export const links: LinksFunction = () => [{ rel: 'stylesheet', href: './tailwind.css' }];
export const links = () => {
  return [
    { rel: "stylesheet", href: styles }
  ];
}

export default function App() {
  return (
    <ThemeProvider>
      <html lang="en">
        <head>
          <meta charSet="utf-8" />
          <meta name="viewport" content="width=device-width, initial-scale=1" />
          <Meta />
          <Links />
        </head>
        <body> 
            <Outlet />
            <ScrollRestoration />
            <Scripts />
            <LiveReload />
        </body>
      </html>
    </ThemeProvider>
  );
}
```

Note this: `import styles from './tailwind.css?url';`, there's an extra `?url`

#### package.json

```JSON
"scripts": {
    "build": "remix vite:build",
    "deploy": "pnpm run build && wrangler pages deploy ./build/client",
    "dev": "remix vite:dev",
    "lint": "eslint --ignore-path .gitignore --cache --cache-location ./node_modules/.cache/eslint .",
    "start": "wrangler pages dev ./build/client",
    "typecheck": "tsc",
    "typegen": "wrangler types",
    "preview": "pnpm run build && wrangler pages dev ./build/client",
    "build-cf-types": "wrangler types"
  },
```

### Node compatibility

According to the official ![documentation](https://developers.cloudflare.com/workers/wrangler/configuration/#node-compatibility), some core `nodejs` libraries cannot run directly in `worker` `pages`. There are two methods; the first is `Use runtime APIs directly`, which I didn't get working. I used the second, `Add polyfills using Wrangler`, adding to `wrangler.toml`: `node_compat = true`. Note that the two are not compatible.

### Prisma migration

The database was previously `PostgresSQL`, and now it needs to be migrated to `D1 Database`. The common `Prisma` documentation: `https://www.prisma.io/docs/orm/prisma-client/deployment/edge/deploy-to-cloudflare`

Assume the database name is `my-d1`. First generate the `sql`:
` pnpm npx wrangler d1 migrations create my-d1 --local`
This generates a migration directory locally. Then execute:
```
pnpm npx prisma migrate diff \
  --from-empty \
  --to-schema-datamodel ./prisma/schema.prisma \
  --script \
  >> migrations/0001_init_.sql
```
The first time you generate a migration file, it will be stored in `migrations/0001_init_.sql`.
Execute the migration file: `pnpm npx wrangler d1 migrations apply my-d1 --local`
At this point the migration is done, but sure enough, something went wrong:
```
near "CONSTRAINT": syntax error at offset 41 [code: 7500]
```
This thing doesn't say which line of code either; the hinted `41` isn't it at all. I had to delete line by line and experiment, and finally found it was the error from: `ALTER TABLE "tablename" ADD CONSTRAINT`
I looked it up: `https://developers.cloudflare.com/d1/reference/migrations/#foreign-key-constraints`

### Using bindings

When using `bindings` in `cloudflare pages`, such as `KV` and `R2`, I kept failing to read these bindings. After studying and testing carefully many times, I found some steps:

#### `functions/[[path]].ts`

```typescript
export const onRequest = createPagesFunctionHandler({
	build,
	mode: process.env.NODE_ENV,
	getLoadContext: context => ({ env: context.context.cloudflare.env }),
});
```
getLoadContext needs to set `env: context.context.cloudflare.env`

#### wrangler.toml

In the file you need to configure the information for the corresponding `binding`, and in the `dashboard` of `cloudflare pages`, you also need to configure the `binding` for the corresponding environment (`production`, `preview`), with the same name


#### typegen

After modifying the `bindings` in `wrangler.toml` or the `dashboard`, you need to run: `pnpm run types`, that is: `wrangler types`, which will generate the `worker-configuration.d.ts` file, and the corresponding settings in `wrangler.toml` will be updated:

```typescript
interface Env {
	SOME_KV: KVNamespace;
	SOME_R2: R2Bucket;
}
```

#### Using on the `remix` server side:

```typescript
context.env.SOME_KV.put(key, 'some value');
```

