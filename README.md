<p align="center">
  <img src="https://user-images.githubusercontent.com/581999/153954565-61b219ee-c352-41b4-b8ff-3eba955b9b7d.png" alt="tRPC-SvelteKit" />
</p>
<h1 align="center">✨ tRPC-SvelteKit</h1>
<p align="center">
  <a href="https://npmjs.org/package/trpc-sveltekit">
    <img src="https://img.shields.io/npm/v/trpc-sveltekit.svg?style=flat-square" alt="NPM version" style="max-width: 100%;" />
  </a>
  <a href="/icflorescu/trpc-sveltekit/blob/main/LICENSE">
    <img src="http://img.shields.io/npm/l/trpc-sveltekit.svg?style=flat-square" alt="License" style="max-width: 100%;" />
  </a>
  <a href="https://npmjs.org/package/trpc-sveltekit">
    <img src="http://img.shields.io/npm/dm/trpc-sveltekit.svg?style=flat-square" alt="Downloads" style="max-width: 100%;" />
  </a>
</p>

<p align="center">
  End-to-end typesafe APIs with <a href="https://trpc.io">tRPC.io</a> in <a href="https://kit.svelte.dev">SvelteKit</a> applications.
  <br />
  No code generation, run-time bloat, or build pipeline.
</p>

## ❤️🇺🇦

See below.

## Key features

✅ Works with `@sveltejs/adapter-node` & `@sveltejs/adapter-vercel`  
✅ Works with SvelteKit's `load()` function for SSR  

## Example application with Prisma & superjson

👉 [tRPC-Sveltekit-Example](https://github.com/icflorescu/trpc-sveltekit-example)

## TL;DR

Add this in your SvelteKit app [hooks](https://kit.svelte.dev/docs/hooks):

```ts
// src/hooks.ts
import { createTRPCHandle } from 'trpc-sveltekit';
// create your tRPC router...

export const handle = createTRPCHandle({ url: '/trpc', router }); // 👈 add this handle
```

## How to use

1. Install this package

`npm install trpc-sveltekit`/`yarn add trpc-sveltekit`

2. Create your tRPC [routes](https://trpc.io/docs/router), [context](https://trpc.io/docs/context) and type exports:

```ts
// $lib/trpcServer.ts
import type { inferAsyncReturnType } from '@trpc/server';
import * as trpc from '@trpc/server';

// optional
export const createContext = () => {
  // ...
  return {
    /** context data */
  };
};

// optional
export const responseMeta = () => {
  // ...
  return {
    // { headers: ... }
  };
};

export const router = trpc
  .router<inferAsyncReturnType<typeof createContext>>()
  // queries and mutations...
  .query('hello', {
    resolve: () => 'world',
  });

export type Router = typeof router;
```

3. Add this handle to your application hooks (`src/hooks.ts` or `src/hooks/index.ts`):

```ts
// src/hooks.ts or src/hooks/index.ts
import { createContext, responseMeta, router } from '$lib/trpcServer';
import { createTRPCHandle } from 'trpc-sveltekit';

export const handle = createTRPCHandle({
  url: '/trpc', // optional; defaults to '/trpc'
  router,
  createContext, // optional
  reponseMeta, // optional
});
```

Learn more about SvelteKit hooks [here](https://kit.svelte.dev/docs/hooks).

4. Create a [tRPC client](https://trpc.io/docs/vanilla):

```ts
// $lib/trpcClient.ts
import type { Router } from '$lib/trpcServer'; // 👈 only the types are imported from the server
import * as trpc from '@trpc/client';

export default trpc.createTRPCClient<Router>({ url: '/trpc' });
```

5. Use the client like so:

```ts
// page.svelte
import trpcClient from '$lib/trpcClient';

const greeting = await trpcClient.query('hello');
console.log(greeting); // => 👈 world
```

## Recipes & caveats 🛠

### Usage with Prisma

When you're building your SvelteKit app for production, you must instantiate your [Prisma](https://www.prisma.io/) client **like this**: ✔️

```ts
// $lib/prismaClient.ts
import pkg from '@prisma/client';
const { PrismaClient } = pkg;

const prismaClient = new PrismaClient();
export default prismaClient;
```

This will **not** work: ❌

```ts
// $lib/prismaClient.ts
import { PrismaClient } from '@prisma/client';

const prismaClient = new PrismaClient();
export default prismaClient;
```

### Configure [superjson](https://github.com/blitz-js/superjson) to correctly handle [Decimal.js](https://mikemcl.github.io/decimal.js/) / Prisma.Decimal

❓ If you don't know why you'd want to use [superjson](https://github.com/blitz-js/superjson), learn more about tRPC data transformers [here](https://trpc.io/docs/data-transformers).

By default, superjson only supports built-in data types to keep the bundle-size as small as possible. Here's how you could extend it with Decimal.js / Prisma.Decimal support:

```ts
// $lib/trpcTransformer.ts
import Decimal from 'decimal.js';
import superjson from 'superjson';

superjson.registerCustom<Decimal, string>(
  {
    isApplicable: (v): v is Decimal => Decimal.isDecimal(v),
    serialize: (v) => v.toJSON(),
    deserialize: (v) => new Decimal(v),
  },
  'decimal.js'
);

export default superjson;
```

Then, configure your tRPC router like so:

```ts
// $lib/trpcServer.ts
import trpcTransformer from '$lib/trcpTransformer';
import * as trpc from '@trpc/server';

export const router = trpc
  .router()
  // .merge, .query, .mutation, etc.
  .transformer(trpcTransformer); // 👈

export type Router = typeof router;
```

...and don't forget to configure your tRPC client:

```ts
// $lib/trpcClient.ts
import type { Router } from '$lib/trpcServer';
import transformer from '$lib/trpcTransformer';
import * as trpc from '@trpc/client';

export default trpc.createTRPCClient<Router>({
  url: '/trpc',
  transformer, // 👈
});
```

⚠️ You'll also have to use this custom `svelte.config.js` in order to be able to build your application for production with `adapter-node`/`adapter-vercel`:

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-node'; // or Vercel 
import preprocess from 'svelte-preprocess';

/** @type {import('@sveltejs/kit').Config} */
const config = {
  preprocess: preprocess(),

  kit: {
    adapter: adapter(),
    vite:
      process.env.NODE_ENV === 'production'
        ? {
            ssr: {
              noExternal: ['superjson'],
            },
          }
        : undefined,
  },
};

export default config;
```

### Client-side helper types

It is often useful to wrap the functionality of your `@trpc/client` api within other functions. For this purpose, it's necessary to be able to infer input types, output types, and api paths generated by your @trpc/server router. Using [tRPC's inference helpers](https://trpc.io/docs/infer-types), you could do something like:

```ts
// $lib/trpcClient.ts
import type { Router } from '$lib/trpcServer';
import trpcTransformer from '$lib/trpcTransformer';
import * as trpc from '@trpc/client';
import type { inferProcedureInput, inferProcedureOutput } from '@trpc/server';

export default trpc.createTRPCClient<Router>({
  url: '/trpc',
  transformer: trpcTransformer,
});

type Query = keyof Router['_def']['queries'];
type Mutation = keyof Router['_def']['mutations'];

// Useful types 👇👇👇
export type InferQueryOutput<RouteKey extends Query> = inferProcedureOutput<Router['_def']['queries'][RouteKey]>;
export type InferQueryInput<RouteKey extends Query> = inferProcedureInput<Router['_def']['queries'][RouteKey]>;
export type InferMutationOutput<RouteKey extends Mutation> = inferProcedureOutput<
  Router['_def']['mutations'][RouteKey]
>;
export type InferMutationInput<RouteKey extends Mutation> = inferProcedureInput<Router['_def']['mutations'][RouteKey]>;
```

Then, you could use the inferred types like so:

```ts
// authors.svelte
<script lang="ts">
  let authors: InferQueryOutput<'authors:browse'> = [];

  const loadAuthors = async () => {
    authors = await trpc.query('authors:browse', { genre: 'fantasy' });
  };
</script>
```

### Server-Side Rendering

If you need to use the tRPC client in SvelteKit's `load()` function for SSR, make sure to initialize it like so:

```ts
// $lib/trpcClient.ts
import { browser } from '$app/env';
import type { Router } from '$lib/trpcServer';
import * as trpc from '@trpc/client';

const client = trpc.createTRPCClient<Router>({
  url: browser ? '/trpc' : 'http://localhost:3000/trpc', // 👈
});
```

### Vercel's Edge Cache for Serverless Functions

Your server responses must [satisfy some criteria](https://vercel.com/docs/concepts/functions/edge-caching) in order for them to be cached Verced Edge Network, and here's where tRPC's `responseMeta()` comes in handy. You could initialize your handle in `src/hooks.ts` like so: 

```ts
// src/hooks.ts or src/hooks/index.ts
import { router } from '$lib/trpcServer';
import { createTRPCHandle } from 'trpc-sveltekit';

export const handle = createTRPCHandle({
  url: '/trpc',
  router,
  responseMeta({ type, errors }) {
    if (type === 'query' && errors.length === 0) {
      const ONE_DAY_IN_SECONDS = 60 * 60 * 24;
      return {
        headers: {
          'cache-control': `s-maxage=1, stale-while-revalidate=${ONE_DAY_IN_SECONDS}`
        }
      };
    }
    return {};
  }
});
```

## Example

See an example with Prisma & superjson: ✨
- [Code](https://github.com/icflorescu/trpc-sveltekit-example)
- [Sandbox](https://githubbox.com/icflorescu/trpc-sveltekit-example)

## License

## Stand with Ukraine 

On 24th of February 2022 [Russia unlawfully invaded Ukraine](https://en.wikipedia.org/wiki/Russo-Ukrainian_War). This is an unjustified, unprovoked attack on the sovereignty of a neighboring country, but also an open affront to international peace and stability that has the potential to degenerate into a nuclear event threatening the very existence of humanity. I am an EU (Romanian) citizen, but I am doing everything in my power to stop this madness. I stand with Ukraine. The entire Svelte community ❤️🇺🇦. Here's [how you can show your support](https://www.stopputin.net/).

The [ISC License](https://github.com/icflorescu/trpc-sveltekit/blob/master/LICENSE).
