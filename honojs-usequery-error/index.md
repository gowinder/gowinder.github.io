# useQuery直接使用honojs client报错: TypeError: Cannot read properties of undefined (reading 'replace')


# useQuery直接使用honojs client报错: TypeError: Cannot read properties of undefined (reading 'replace')

最近开发了一个项目, 使用`nextjs`, `honojs`, `drizzle-orm`, 部署在`cloudflare pages`上, 因为`honojs`对于边缘计算比较友好,所以采用.

## honojs 查询接口定义

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

## hono app的route设置

```typescript

const app = new Hono<{ Bindings: Bindings }>().basePath('/api');
const routes = app.route('/sometable', someTable);

export const GET = handle(app);
export const POST = handle(app);

export type AppType = typeof routes;
```

## useQuery 调用

```typescript

export const useGetSomeTable = () => {
  const query = useQuery({
    queryKey: ['someTable'], // 使用 filters 作为查询键
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

## 调用时报错

```
TypeError: Cannot read properties of undefined (reading 'replace')
```

问题出在: `const response = await client.api.someTable.$get();` 这里.
调了两天, `chatgpt`问了, `gemini pro`问了,试了好多方法,没有结果.

正要放弃时,又回去看了视频教程: `https://www.youtube.com/watch?v=N_uNKAus0II&t=23162s`, 我是照着来的, 突然发现一个东西:

```typescript
export const client = hc<AppType>(process.env.NEXT_PUBLIC_APP_URL!);
```

一去检查发现, 我在`wrangler.toml`中定义了`NEXT_PUBLIC_APP_URL`, 但是我是`bun dev`本地启动的, 在`.env`文件中没有, 于是在`.env`中增加`NEXT_PUBLIC_APP_URL="http://localhost:3000"`.

## 结果

好了, 没有这个变量, `hono/client/hc` 输入一个`undefined` 他也不会报错,太奇怪. 记录一下, 喜欢遇到同样问题的人有帮助.

