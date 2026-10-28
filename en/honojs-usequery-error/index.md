# useQuery Error When Using the honojs Client Directly: TypeError: Cannot read properties of undefined (reading 'replace')


# useQuery Error When Using the honojs Client Directly: TypeError: Cannot read properties of undefined (reading 'replace')

Recently I developed a project using `nextjs`, `honojs`, and `drizzle-orm`, deployed on `cloudflare pages`. I chose `honojs` because it is friendly to edge computing.

## honojs Query Endpoint Definition

```typescript
export const runtime = 'edge';

type Bindings = {
    DB: D1Database;
};

const app = new Hono<{ Bindings: Bindings }>()
  .get('/', async (c) => {
    try {
      const conditions: Record<string, any> = {};
      const results = await getDb().query.someTable.findMany({});
      return c.json(results);
      // return c.json(resultData);
    } catch (error: Error | any) {
      console.error(error);
      return c.json(error);
    }
  });

export default app;
```

## Route Setup for the hono app

```typescript
const app = new Hono<{ Bindings: Bindings }>().basePath('/api');
const routes = app.route('/sometable', someTable);

export const GET = handle(app);
export const POST = handle(app);

export type AppType = typeof routes;
```

## Calling with useQuery

```typescript
export const useGetSomeTable = () => {
  const query = useQuery({
    queryKey: ['someTable'], // Use filters as the query key
    queryFn: async () => {
      const response = await client.api.someTable.$get();
      console.log('API Response: ', response);
      if (!response.ok) {
        throw new Error('Failed to fetch sometable');
      }
      return response.json();
    },
    throwOnError: true,
      refetchOnWindowFocus: false,
      staleTime: 1000 * 60 * 5,
  });

  return query;
};
```

## The Error When Calling

```
TypeError: Cannot read properties of undefined (reading 'replace')
```

The problem was here: `const response = await client.api.someTable.$get();`
I debugged it for two days, asked `chatgpt`, asked `gemini pro`, tried many methods, with no result.

Just as I was about to give up, I went back and watched the video tutorial: `https://www.youtube.com/watch?v=N_uNKAus0II&t=23162s`. I had been following along, and suddenly I noticed something:

```typescript
export const client = hc<AppType>(process.env.NEXT_PUBLIC_APP_URL!);
```

I went to check and found that I had defined `NEXT_PUBLIC_APP_URL` in `wrangler.toml`, but I started locally with `bun dev`, where the `.env` file didn't have it. So I added `NEXT_PUBLIC_APP_URL="http://localhost:3000"` to `.env`.

## Result

And that fixed it. Without this variable, `hono/client/hc` receives an `undefined` and doesn't throw an error, which is really strange. Recording this here in the hope that it helps people who run into the same problem.

