# 使用nestjs+honojs+drizzle+shadcnui开发部署在cloudflare pages上的web app


# 使用 nestjs+honojs+drizzle+shadcnui 开发部署在 cloudflare pages 上的 web app

最近一段时间,一直在撸 cloudflare 的羊毛, workers, pages 都有. 经过几个小项目的折腾, 读过了开始的摸索期, 现在分享一下使用这几个开源库,快速开发一个部署在`Cloudflare Pages`上的`Web App`的过程.

如果不想看过程, 可以直接调到末尾, 我分享的项目地址,`fork` [我的示例项目](https://github.com/gowinder/cloudflare-pages-starter)后就可以直接用了.

**注意**, 有部分代码又`AI`生成后再二次修改.

## 准备过程

- 准备好`Cloudflare`的账号, `Pages`, `Workers`都是免费的, 但是,免费的有一下限制, 有个比较重要的是: 编译好的二进制包不能超过`1MB`, 参见:[Limits | Cloudflare Workers docs](https://developers.cloudflare.com/workers/platform/limits). 如果你的`WebApp`编译好后超过`1MB`, 那么就需要付费了, 一个月 5 刀,也不算贵. 比自己找`vps`安装一堆服务要方便多了, 还可以开`Cloudflare CDN`.

- 建议直接在`github`上创建项目,这样部署设置里面,可以直接选择`预览`和`生成`环境对应的分支,只要提交了就自动编译,非常方便

- 安装`bun`: `npm install -g bun`

- 其他开发相关环境就不一一介绍了.

## 过程

### 创建工程

- 运行命令: `bun x create-next-app cloudflare-pages-starter`, 弹出的提示一律回车,默认就可以了. `cloudflare-pages-starter` 就是项目名称及目录名称

- `cd cloudflare-pages-starter && git remote add origin YOUR_GITHUB_REPO`, 这一步设置`git`远程地址

- 安装需要的库: `bun add @auth/core@^0.32.0 @auth/drizzle-adapter@^1.2.0 @hono/auth-js@1.0.8 drizzle-orm@^0.30.10 drizzle-zod hono@^4.4.2 shadcn-ui zod zustand @tanstack/react-query`, 开发库: `bun add -d @cloudflare/next-on-pages @cloudflare/workers-types autoprefixer drizzle-kit@0.21.2 postcss tailwindcss wrangler`

### 初始化项目

这段内容可以参考官方文档: [Get started | Full-stack (SSR) | Next.js apps</title><title>Get started | Cloudflare Pages docs](https://developers.cloudflare.com/pages/framework-guides/nextjs/ssr/get-started/)

项目根目录新建一个`wrangler.toml`,内容如下:

```toml
name = "cloudflare-pages-starter"
compatibility_date = "2024-04-05"
pages_build_output_dir = ".vercel/output/static"

compatibility_flags = ["nodejs_compat"]
```

`name`换成你自己的`pages`的项目名称

然后运行: `bun wrangler login`, 在弹出网页授权`wrangler`

删除`next.config.js`, 新建`next.config.mjs`, 内容:

```javascript
import { setupDevPlatform } from "@cloudflare/next-on-pages/next-dev";

/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  images: {
    domains: ["example.com", "images.unsplash.com"],
  },
};

// we only need to use the utility during development so we can check NODE_ENV
// (note: this check is recommended but completely optional)
if (process.env.NODE_ENV === "development") {
  // `await`ing the call is not necessary but it helps making sure that the setup has succeeded.
  //  If you cannot use top level awaits you could use the following to avoid an unhandled rejection:
  //  `setupDevPlatform().catch(e => console.error(e));`
  await setupDevPlatform();
}

export default nextConfig;
```

**注意**, 如果不这样做, `bun dev` 启动本地服务时,将无法连接数据库

#### 测试运行一下

`bun dev`

打开`http://localhost:3000`, 应该有`nextj.js`的页面

#### shadcn-ui 常用控件

首先按文档初始化: [Installation - shadcn/ui](https://ui.shadcn.com/docs/installation)

然后安装控件,我们一次装好:

```shell
bunx shadcn-ui add toast avatar badge button card dialog dropdown-menu input skeleton table use-toast
```

这些空间文件安装在`components/ui`下面

#### 数据库

##### tsconfig

修改`tsconfig.json`, `target`修改为 `ES2022`,如下:

```typescript
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

##### drizzle config

文件: `drizzle.config.ts`

```typescript
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./db/schema.ts",
  out: "./migrations",
  dialect: "sqlite", // 'postgresql' | 'mysql' | 'sqlite'
  verbose: true,
  driver: "d1",
  dbCredentials: {
    wranglerConfigPath: "./wrangler.toml",
    dbName: "pages-starter-preview",
  },
});
```

##### `db`

新建`/db/db.ts`

```typescript
// app/db/db.ts
import { drizzle } from "drizzle-orm/d1";
import * as schema from "./schema";

export const runtime = "edge";

// 全局函数，接受 context 参数，返回 drizzle 实例
export const getDb = () => {
  return drizzle((process.env as any).DB, { schema });
};
```

**注意**: `export const runtime = "edge";` 是部署在`Cloudflare Pages`上必要的.

这样定义后,在其他地方就可以直接使用`getDb`了

##### `schema`

新建: `db/schema.ts`, 表接口参见: [authjs](https://authjs.dev/reference/drizzle-adapter/lib/sqlite)

```typescript
import { AdapterAccountType } from "@auth/core/adapters";
import {
  integer,
  primaryKey,
  sqliteTable,
  text,
} from "drizzle-orm/sqlite-core";

export const users = sqliteTable("user", {
  id: text("id")
    .primaryKey()
    .$defaultFn(() => crypto.randomUUID()),
  name: text("name"),
  email: text("email").unique(),
  emailVerified: integer("emailVerified", { mode: "timestamp_ms" }),
  image: text("image"),
});

export const accounts = sqliteTable(
  "account",
  {
    userId: text("userId")
      .notNull()
      .references(() => users.id, { onDelete: "cascade" }),
    type: text("type").$type<AdapterAccountType>().notNull(),
    provider: text("provider").notNull(),
    providerAccountId: text("providerAccountId").notNull(),
    refresh_token: text("refresh_token"),
    access_token: text("access_token"),
    expires_at: integer("expires_at"),
    token_type: text("token_type"),
    scope: text("scope"),
    id_token: text("id_token"),
    session_state: text("session_state"),
  },
  (account) => ({
    compoundKey: primaryKey({
      columns: [account.provider, account.providerAccountId],
    }),
  })
);

export const sessions = sqliteTable("session", {
  sessionToken: text("sessionToken").primaryKey(),
  userId: text("userId")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" }),
  expires: integer("expires", { mode: "timestamp_ms" }).notNull(),
});

export const verificationTokens = sqliteTable(
  "verificationToken",
  {
    identifier: text("identifier").notNull(),
    token: text("token").notNull(),
    expires: integer("expires", { mode: "timestamp_ms" }).notNull(),
  },
  (verificationToken) => ({
    compositePk: primaryKey({
      columns: [verificationToken.identifier, verificationToken.token],
    }),
  })
);

export const authenticators = sqliteTable(
  "authenticator",
  {
    credentialID: text("credentialID").notNull().unique(),
    userId: text("userId")
      .notNull()
      .references(() => users.id, { onDelete: "cascade" }),
    providerAccountId: text("providerAccountId").notNull(),
    credentialPublicKey: text("credentialPublicKey").notNull(),
    counter: integer("counter").notNull(),
    credentialDeviceType: text("credentialDeviceType").notNull(),
    credentialBackedUp: integer("credentialBackedUp", {
      mode: "boolean",
    }).notNull(),
    transports: text("transports"),
  },
  (authenticator) => ({
    compositePK: primaryKey({
      columns: [authenticator.userId, authenticator.credentialID],
    }),
  })
);
```

#### `API`

##### `auth-config.ts` 验证相关

文件: `app/api/[[..route]]/auth-config.ts`

```typescript
import GitHub from "@auth/core/providers/github";
import Google from "@auth/core/providers/google";
import { DrizzleAdapter } from "@auth/drizzle-adapter";
import { AuthConfig } from "@hono/auth-js";
import { Context } from "hono";

import { getDb } from "@/db/db";

export function getAuthConfig(c: Context): AuthConfig {
  return {
    session: {
      strategy: "jwt",
      maxAge: 60 * 60 * 24 * 30, // 30 days
    },
    debug: true,
    secret: process.env.AUTH_SECRET,
    providers: [
      GitHub({
        clientId: process.env.GITHUB_CLIENT_ID,
        clientSecret: process.env.GITHUB_CLIENT_SECRET,
      }),
      Google({
        clientId: process.env.GOOGLE_CLIENT_ID,
        clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      }),
    ],
    callbacks: {
      async session({ session, token }) {
        const newSession = session as any;
        newSession["token"] = token;
        return session;
      },
      async jwt({ token, user, account, profile, session }) {
        return token;
      },
    },
    adapter: {
      ...DrizzleAdapter(getDb()),
    },
  };
}
```

这个是配置验证相关的, 我这里使用了两种`provider`: [Creating an OAuth app - GitHub Docs](https://docs.github.com/en/developers/apps/building-oauth-apps/creating-an-oauth-app), [Google OAuth2](https://developers.google.com/identity/protocols/oauth2)

##### users

文件: `app/api/[[...route]]/users.ts`

```typescript
import { eq } from "drizzle-orm";
import { Hono } from "hono";
import { HTTPException } from "hono/http-exception";
import { Bindings } from "hono/types";

import { getDb } from "@/db/db";
import { accounts } from "@/db/schema";

const app = new Hono<{ Bindings: Bindings }>().get("/:userId", async (c) => {
  const userId = c.req.param("userId");
  const db = getDb();
  const userAccount = await db.query.accounts.findFirst({
    where: eq(accounts.userId, userId),
    columns: {
      provider: true,
      providerAccountId: true,
      api_token: true,
      api_enabled: true,
    },
  });

  if (!userAccount) {
    throw new HTTPException(404, { message: "user not found" });
  }

  return c.json({
    ...userAccount,
  });
});

export default app;
```

这个用于查询用户信息

##### router

文件`app/api/[[..route]]/route.ts`

```typescript
import { D1Database } from "@cloudflare/workers-types";
import { authHandler, initAuthConfig } from "@hono/auth-js";
import { Hono } from "hono";
import { handle } from "hono/vercel";

import { getAuthConfig } from "./auth-config";

export const runtime = "edge";

// This ensures c.env.DB is correctly typed
type Bindings = {
  DB: D1Database;
};

const app = new Hono<{ Bindings: Bindings }>().basePath("/api");

app.use(
  "*",
  initAuthConfig((c) => {
    const config = getAuthConfig(c);
    return config;
  })
);

app.use("/auth/*", authHandler());

app.use("/protected", async (c, next) => {
  const auth = c.get("authUser");
  // console.log("c:", c);
  if (!auth) {
    return c.text("Unauthorized", 401);
  } else {
    return c.text(JSON.stringify(auth.session.user));
  }
});

app.use("/", async (c) => {
  return c.text("Hello World");
});

const routes = app.get("/routes", (c) => {
  const routes = app.routes;
  console.log("所有路由:");
  routes.forEach((route) => {
    console.log(`${route.method} ${route.path}`);
  });
  return c.json(
    routes.map((route) => ({ method: route.method, path: route.path }))
  );
});

export const GET = handle(app);
export const POST = handle(app);

export type AppType = typeof routes;
```

这样就使用`hono.js`来做路由了,他接管了`next.js`的路由系统.

**注意** 如果在其他文件定义子路由,需要直接在`route.ts`中`const routes = app.get().post().route()` 这样编写,不然可能不生效.

#### useQuery 相关

##### provider

文件: `providers/query-provider.tsx`

```typescript
// In Next.js, this file would be called: app/providers.jsx
"use client";

// Since QueryClientProvider relies on useContext under the hood, we have to put 'use client' on top
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        // With SSR, we usually want to set some default staleTime
        // above 0 to avoid refetching immediately on the client
        staleTime: 60 * 1000,
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined = undefined;

function getQueryClient() {
  if (typeof window === "undefined") {
    // Server: always make a new query client
    return makeQueryClient();
  } else {
    // Browser: make a new query client if we don't already have one
    // This is very important so we don't re-make a new client if React
    // suspends during the initial render. This may not be needed if we
    // have a suspense boundary BELOW the creation of the query client
    if (!browserQueryClient) browserQueryClient = makeQueryClient();
    return browserQueryClient;
  }
}

type Props = {
  children: React.ReactNode;
};

export function QueryProvider({ children }: Props) {
  // NOTE: Avoid useState when initializing the query client if you don't
  //       have a suspense boundary between this and the code that may
  //       suspend because React will throw away the client on the initial
  //       render if it suspends and there is no boundary
  const queryClient = getQueryClient();

  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

##### hono client

文件: `lib/hono.ts`, 这个用与提供给客户端来调用`hono`定义的`API`

```typescript
import { hc } from "hono/client";

import { AppType } from "@/app/api/[[...route]]/route";

console.log("process.env.NODE_ENV:", process.env.NODE_ENV);
console.log(
  "process.env.NEXT_PUBLIC_APP_URL:",
  process.env.NEXT_PUBLIC_APP_URL
);
export const client = hc<AppType>(process.env.NEXT_PUBLIC_APP_URL!);
```

##### useSession

文件: `features/use-session.ts`

```typescript
import { Session } from "@auth/core/types";
import { QueryClient, useQuery } from "@tanstack/react-query";

export interface CustomSession extends Session {
  token?: {
    sub?: string;
  };
}

export const useSession = (): { session: CustomSession; status: string } => {
  const { data, status } = useQuery(
    {
      queryKey: ["session"],
      queryFn: async () => {
        const res = await fetch("/api/auth/session");
        return res.json();
      },
      staleTime: 5 * (60 * 1000),
      gcTime: 10 * (60 * 1000),
      refetchOnWindowFocus: true,
    },
    new QueryClient()
  );
  return { session: data as CustomSession, status };
};
```

这个`useQuery`负责从服务端取回`session`

##### use-get-user-info.ts

文件: `features/use-get-user-info.ts`

```typescript
import { client } from "@/lib/hono";
import { useQuery } from "@tanstack/react-query";

export function useGetUserInfo(userId: string | undefined) {
  return useQuery({
    queryKey: ["userInfo", userId],
    queryFn: async () => {
      if (!userId) throw new Error("用户ID是必需的");
      const response = await client.api.users[":userId"].$get({
        param: { userId },
      });
      if (!response.ok) throw new Error("获取用户信息失败");
      return response.json();
    },
    enabled: !!userId,
  });
}
```

#### zustand 存储

我们使用 zustand 来存储本地数据和发送事件

文件: `stores/ui-event.store.ts`

```typescript
import { z } from "zod";
import { create } from "zustand";

import { DashboardPage } from "@/defines/dashboard-page";

const uiEventSchema = z.object({
  showLogin: z.boolean().default(false),
  dashboardPage: z.string().default(DashboardPage.Home),
  showSettings: z.boolean().default(false),
});

type UIEvent = z.infer<typeof uiEventSchema>;

interface UIEventStore {
  event: UIEvent;
  setEvent: (event: UIEvent) => void;
  reset: () => void;
  setShowLogin: (showLogin: boolean) => void;
  setDashboardPage: (dashboardPage: string) => void;
  setShowSettings: (showSettings: boolean) => void;
}

const useUIEventStore = create<UIEventStore>((set) => ({
  event: {
    showLogin: false,
    dashboardPage: DashboardPage.Home,
    showSettings: false,
  },
  setEvent: (event) => set({ event }),
  reset: () =>
    set({
      event: {
        showLogin: false,
        dashboardPage: DashboardPage.Home,
        showSettings: false,
      },
    }),
  setShowLogin: (showLogin) =>
    set((prevState) => ({
      event: { ...prevState.event, showLogin },
    })),
  setDashboardPage: (dashboardPage) =>
    set((prevState) => ({
      event: { ...prevState.event, dashboardPage },
    })),
  setShowSettings: (showSettings) =>
    set((prevState) => ({
      event: { ...prevState.event, showSettings },
    })),
}));

export default useUIEventStore;
```

#### `dashboard`相关页面

##### ##### layout.tsx

文件: `app/layout.tsx`

```typescript
import { QueryProvider } from "@/providers/query-provider";
import { Inter as FontSans } from "next/font/google";
import "./globals.css";
import { cn } from "@/lib/utils";

export const metadata = {
  title: "Create Next App",
  description: "Generated by create next app",
};

const fontSans = FontSans({
  subsets: ["latin"],
  variable: "--font-sans",
});

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body
        className={cn(
          "min-h-screen bg-background font-sans antialiased",
          fontSans.variable
        )}
      >
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  );
}
```

主要是增加`QueryProvider`

##### `page.tsx`

修改`app/page.tsx`为如下:

```typescript
"use client";

import Dashboard from "@/components/dashboard/dahsboard";
import { Toaster } from "@/components/ui/toaster";
import { useSession } from "@/hooks/use-session";

export default function Home() {
  const sessionData = useSession();
  console.log("sessionData:", sessionData);
  const session = sessionData?.session;

  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <Dashboard session={session} />
      <Toaster />
    </div>
  );
}
```

`useSession`我们后面在加

##### `dashboard.tsx`

```typescript
import { CustomSession } from "@/features/use-session";
import useUIEventStore from "@/stores/ui-event.store";
import { useEffect, useState } from "react";
import SignIn from "../sign-in";
import DashboardNavbar from "./dashboard-navbar";
import UserSettingsDialog from "./user-settings-dialog";

export default function Dashboard({ session }: { session: CustomSession }) {
  const { event, setShowSettings } = useUIEventStore();
  const [isDialogOpen, setIsDialogOpen] = useState(false);
  const [isSettingsOpen, setIsSettingsOpen] = useState(false);

  const handleCloseDialog = () => {
    console.log("Closing dialog");
    setIsDialogOpen(false);
  };

  useEffect(() => {
    setIsSettingsOpen(event.showSettings);
  }, [event.showSettings]);

  const handleCloseSettings = () => {
    setShowSettings(false);
  };

  console.log("Dashboard, Rendering Dashboard isDialogOpen:", isDialogOpen);

  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <DashboardNavbar session={session} />
      <div className="container p-4 mx-auto">
        <SignIn />
      </div>

      <UserSettingsDialog
        isOpen={isSettingsOpen}
        onClose={handleCloseSettings}
        session={session}
      />
    </div>
  );
}
```

##### `dashboard-navbar.tsx` 导航控件 (使用 ai 生成: <https://v0.dev/t/xYHqD5MkVkT>)

```typescript
/**
 * v0 by Vercel.
 * @see https://v0.dev/t/xYHqD5MkVkT
 * Documentation: https://v0.dev/docs#integrating-generated-code-into-your-nextjs-app
 */
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Button } from "@/components/ui/button";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { DashboardPage } from "@/defines/dashboard-page";
import { useLogout } from "@/features/use-logout";
import { CustomSession } from "@/features/use-session";
import useUIEventStore from "@/stores/ui-event.store";
import { LogOut, Settings } from "lucide-react";
import Link from "next/link";
import NavbarItem from "./navbar-item";

type SiteIconProps = {
  session?: CustomSession;
};

export default function DashboardNavbar({ session }: SiteIconProps) {
  const { setShowLogin, setDashboardPage, setShowSettings } = useUIEventStore();
  const logout = useLogout();

  const onSettingsClicked = () => {
    console.log("settings clicked:");
    setShowSettings(true);
  };

  const onLogoutClicked = () => {
    console.log("logout clicked:");
    // * 调用 useLogout
    logout.mutateAsync();
  };

  console.log("navbar, session:", session);
  console.log("navbar, image:", session?.user?.image);
  return (
    <nav className="fixed inset-x-0 top-0 z-50 bg-white shadow-sm dark:bg-gray-950/90">
      <div className="w-full px-4 mx-auto max-w-7xl">
        <div className="flex items-center justify-between h-14">
          <Link href="#" className="flex items-center" prefetch={false}>
            <SiteIcon className="w-6 h-6" />
            <span className="sr-only">Hub</span>
          </Link>
          <nav className="hidden gap-4 md:flex">
            <NavbarItem name={DashboardPage.Home} title="Home" />
          </nav>
          {session?.user?.name ? (
            <DropdownMenu>
              <DropdownMenuTrigger asChild>
                <Button
                  variant="ghost"
                  className="relative w-8 h-8 rounded-full"
                >
                  <Avatar>
                    <AvatarImage
                      src={session?.user?.image ?? ""}
                      alt="@shadcn"
                    />
                    <AvatarFallback>
                      {session?.user?.name.charAt(0) ?? ""}
                    </AvatarFallback>
                  </Avatar>
                </Button>
              </DropdownMenuTrigger>
              <DropdownMenuContent className="w-56" align="end" forceMount>
                <DropdownMenuItem onClick={() => onSettingsClicked()}>
                  <Settings className="w-4 h-4 mr-2" />
                  <span>Settings</span>
                </DropdownMenuItem>
                <DropdownMenuItem onClick={() => onLogoutClicked()}>
                  <LogOut className="w-4 h-4 mr-2" />
                  <span>Logout</span>
                </DropdownMenuItem>
              </DropdownMenuContent>
            </DropdownMenu>
          ) : (
            <div className="flex items-center gap-4">
              <Button
                variant="outline"
                size="sm"
                onClick={() => setShowLogin(true)}
              >
                Sign in
              </Button>
              <Button size="sm">Sign up</Button>
            </div>
          )}
        </div>
      </div>
    </nav>
  );
}

function SiteIcon(props: React.SVGProps<SVGSVGElement>) {
  return <div></div>;
}
```

**注意**, 如果导航栏要加入新的导航,就在`<NavbarItem name={DashboardPage.Home} title="Home" />`后面增加就可以了,并且`DashboardPage`枚举也要增加.

##### `navbar-item.tsx` 导航按键

```typescript
import useUIEventStore from "@/stores/ui-event.store";
import Link from "next/link";
type NavbarItemProps = {
  name: string;
  title: string;
};

export default function NavbarItem({ name, title }: NavbarItemProps) {
  const { setDashboardPage, event } = useUIEventStore();
  const isActive = event.dashboardPage === name;

  return (
    <div
      className={`relative px-4 py-2 ${
        isActive ? "bg-gray-100 backdrop-blur-sm rounded-md shadow-md" : ""
      }`}
    >
      <Link
        href="#"
        className="flex items-center text-sm font-medium transition-colors hover:underline"
        prefetch={false}
        onClick={() => setDashboardPage(name)}
      >
        {title}
      </Link>
    </div>
  );
}
```

#### 环境变量

文件: `.env`

```env
NEXT_PUBLIC_APP_URL='http://localhost:3000'  # 本地用
AUTH_SECRET="local-dev"
GITHUB_CLIENT_ID="..."
GITHUB_CLIENT_SECRET="..."
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."
```

注意, `.env`文件是给本地: `bun dev` 启动时用的,如果要部署到线上, 除` NEXT_PUBLIC_APP_URL``外,其他变量需要在 https://dash.cloudflare.com/ 上增加 `secret`变量. 对于`NEXT_PUBLIC_APP_URL`, 见下面:

#### wrangler.toml

修改`wrangler.toml`, 增加:

```toml
[env.preview.vars] # 预览环境变量, 如果有其他非secret变量,都需要放这里
BUN_VERSION = "1.1.8"
NEXT_PUBLIC_APP_URL = "https://develop.your-pages-name.pages.dev"

[env.production.vars] # 生产环境变量, 如果有其他非secret变量,都需要放这里
BUN_VERSION = "1.1.8"
NEXT_PUBLIC_APP_URL = "https://your-pages-name.pages.dev"

[[d1_databases]] # 测试环境(本地) D1 数据库
binding = "DB"
database_name = "your-d1-database-name"
database_id = "your-d1-database-id"
[[env.preview.d1_databases]] # 测试环境 D1 数据库
binding = "DB"
database_name = "your-d1-database-name"
database_id = "your-d1-database-id"
[[env.production.d1_databases]] # 生产环境 D1 数据库
binding = "DB"
database_name = "your-d1-database-name"
database_id = "your-d1-database-id"
```

去 <https://dash.cloudflare.com/> 上配置好`D1`数据库(就是一个 `sqlite3` 数据库,只不过部署在边缘计算节点上), 可以把测试库和生产库分开配置.

#### package.json

在`scripts`中增加:

```json
"build:local": "bun x @cloudflare/next-on-pages@1",
    "preview": "wrangler pages dev .vercel/output/static --live-reload --compatibility-flag=nodejs_compat --d1 DB=your_db_id --r2=your_r2_id",
    "drizzle-gen": "drizzle-kit generate --dialect sqlite --schema=./db/schema.ts --out=./migrations",
    "migrate-local": "wrangler d1 migrations apply page-starter-preview --local",
    "migrate-preview": "wrangler d1 migrations apply page-starter-preview --remote",
    "migrate-prod": "wrangler d1 migrations apply page-starter-prod",
    "commit": "./node_modules/cz-customizable/standalone.js"
```

### 启动

- 生成`drizzle`中间代码: `bun drizzle-gen``

#### 本地

```shell
bun migrate-local # 本地数据库迁移
bun dev
```

#### 预览

```shell
bun migrate-previe # 预览数据库迁移
```

#### 生成

```shell
bun migrate-prod # 生产数据库迁移
```

在`Cloudflare Pages Dashboard`上, 直接将`preview`环境绑定到`develop`分支, `production`绑定到`main`分支,就可以做到提交后自动发布对应的环境.

