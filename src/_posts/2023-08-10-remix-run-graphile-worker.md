---
layout: post
title: "Setting up Graphile Worker with Remix.run"
date: 2023-08-10 10:28:33 -0300
categories: remix.run nodejs typescript
---

Coming from a Ruby on Rails background, it's pretty much always been required to implement some sort of background job processing in order to integrate with external APIs or to run long running tasks without blocking the web request cycle and add robustness to the integration (with automated retries in case of failures, etc). `NodeJS` being async means that this is mostly not needed as any IO will run asynchronoously without blocking the event loop, but if more advanced features are required such as retries, processing in order, auditing, etc; then this is not enough. Looking for tools similar to `que` or `GoodJob` but for NodeJS, I stumbled upon [`graphile worker`](LINK_GITHUB), which takes advantage of Postgres' features to efficiently run workers without causing excessive load in the database. My team is small and I don't want to run Redis in production, so this library fit very well.

Remix.run however needs some tweaking in order to work with [`graphile worker`](LINK_GITHUB), so below is the setup I needed to do in order to:
- Enable the usage of Typescript and importing of internal npm packages from within the Remix app.
- Produce a buildable javascript artifact that can be run in a separate process but alongside the rest of the Remix.run processes in development and production.

First of all, I installed the graphile worker package using `npm install --save graphile-worker` and then created the worker entrypoint module in `$PROJECT_ROOT/worker/index.ts`. This file holds the `graphile worker` bootstrap code. I chose to import each worker task separately and add them to a task list, instead of choosing a task directory (which seems to also be an option, but would be more of a challenge for applications that run through a traspilation / minification process).

```typescript
import { run } from "graphile-worker";
import hello from "./tasks/hello";

async function main() {
  const runner = await run({
    connectionString: process.env.DATABASE_URL,
    concurrency: Number(process.env.GRAPHILE_WORKER_CONCURRENCY || 5),
    noHandleSignals: false,
    pollInterval: 1000,
    taskList: {
      hello,
    },
  });

  await runner.promise;
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Afterwards, I wrote the task implementation (this is just an example but this code could be an external API call, etc.):

```typescript
import type { JobHelpers } from "graphile-worker";

const hello = async (payload: unknown, helpers: JobHelpers) => {
  if (!(typeof payload === "object" && payload && "name" in payload))
    throw new Error("Payload must be an object with a name property");

  const { name } = payload;
  helpers.logger.info(`Hello, ${name}`);
};

export default hello;
```

The two files above should be sufficient to determine "how to process a `hello` asynchronous task", but to actually enqueue the task from a Remix.run loader function (actions work just as well), I needed to build a custom `addJob` function:

```typescript
import type { LoaderArgs } from "@remix-run/server-runtime";
import { addJob } from "~/utils/worker";

export async function loader({ request }: LoaderArgs) {
  await addJob(
    "hello",
    { name: "Graphile Worker" }
  );

  return null
}
```

And this is the `~/utils/worker` implementation, which avoids creating duplicate database connections when automatically reloading code during development:

```typescript
import type { WorkerUtils } from "graphile-worker";
import { makeWorkerUtils } from "graphile-worker";
import { singleton } from "../singleton.server";

// This function is meant to be accessed through the singleton function,
// which will ensure that the workerUtils are only created once to prevent duplicate
// connections to the database.
const createWorkerUtils = async (): Promise<WorkerUtils> => {
  const workerUtils = await makeWorkerUtils({
    connectionString: process.env.DATABASE_URL,
  });

  return workerUtils;
};

export const addJob = async (
  identifier: string,
  payload: any,
  options?: { queueName?: string; runAt?: Date; maxAttempts?: number }
) => {
  const worker = await singleton("createWorkerUtils", createWorkerUtils);
  return worker.addJob(identifier, payload, options);
};
```

The singleton (`~/singleton.server.ts`) is taken from the Remix indie stack:

```typescript
// since the dev server re-requires the bundle, do some shenanigans to make
// certain things persist across that 😆
// Borrowed/modified from https://github.com/jenseng/abuse-the-platform/blob/2993a7e846c95ace693ce61626fa072174c8d9c7/app/utils/singleton.ts

export function singleton<Value>(name: string, value: () => Value): Value {
  const yolo = global as any
  yolo.__singletons ??= {}
  yolo.__singletons[name] ??= value()
  return yolo.__singletons[name]
}
```

In order to run the worker process independently of the Remix.run app server, I needed to write a custom esbuild script and updating the `package.json` tasks to produce development and production builds. I installed `tsx` (`npm install tsx --save-dev`), and added `build:worker`, `dev:worker` and `worker` tasks:

```json
{
  // ...
  "scripts": {
    // ...
    // The "dev" task, omitted here, runs "build" and automatically picks up any dev:* task.
    // The "build" task, omitted here, automatically picks up any build:* task.
    "build:worker": "tsx other/build-worker.ts",
    "dev:worker": "cross-env NODE_ENV=development GRAPHILE_LOGGER_DEBUG=1 nodemon --require dotenv/config build/worker.js --watch build/worker.js",
    "worker": "cross-env NODE_ENV=production node ./build/worker.js",
    //...
  }
  //...
}
```

With the `other/build-worker.ts` file being heavily inspired by the [Epic Stack server build script](LINK_GITHUB):

```typescript
import esbuild from "esbuild";
import pkg from "../package.json";

esbuild
  .build({
    entryPoints: ["./worker/index.ts"],
    platform: "node",
    outfile: "./build/worker.js",
    format: "cjs",
    bundle: true,
    external: [
      ...Object.keys(pkg.dependencies || {}),
      ...Object.keys(pkg.devDependencies || {}),
    ],
  })
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

Finally, I had to update our Dockerfile's entrypoint script to _stop_ running `npm run start`, which used to always start the web server directly, and instead configured our set up to run separate tasks for the web server and the worker. As this project was deployed in `fly.io`, I had to add a `[processes]` config to `fly.toml`, but this should be portable to docker compose, ecs or any other kind of production setup that allows overriding the Docker CMD command:

```toml
[processes]
app = "npm run app"
worker = "npm run worker"

```


## Addendum
- The `graphile worker` package allows running tasks of a given name _sequentially_ by specifying the "queueName" option. I found this very useful to be able to process incoming webhooks from Stripe avoiding race conditions (they send webhooks too quickly one after another!)
```typescript
await addJob("handleStripeWebhook", event, { queueName: "stripe-webhooks" });
```

- The `esbuild` package supports an option called `packages=external`, but it didn't work very well for me (I want to bundle all internal code, but reference any npm package via require), because I'm using custom import paths with `~/path/to/file`, and `packages=external` will not bundle anything that doesn't start with a relative or absolute directory path name, so I needed to fall back to specifying the dependencies via `package.json`.

