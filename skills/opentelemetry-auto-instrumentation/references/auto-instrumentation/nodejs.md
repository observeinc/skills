# OpenTelemetry Node.js Auto-Instrumentation Reference

Most Node.js web frameworks work out of the box with the `@opentelemetry/auto-instrumentations-node` meta-package — one `npm install` and a CLI flag at startup. A few libraries ship their own OTel plugins that must be registered separately. This guide covers both cases.

Two decisions determine the setup:

1. **Does the framework or ORM need a separate plugin?** Most frameworks are covered by the meta-package alone. Some ship their own OTel instrumentation that must be installed and registered separately — check the reference doc for your specific stack.
2. **Is the app CommonJS or ESM?** This determines the startup command. Check for `"type": "module"` in `package.json` or `.mjs` file extensions. For TypeScript, the compiled output format (not the source `.ts` format) determines which flags to use.

## Resolve latest versions

Before installing anything, query the npm registry for the latest stable versions. Use these resolved versions in all subsequent install commands.

#### POSIX shell (`sh`, `bash`, `zsh`)

```sh
OTEL_AUTO_NODE_VERSION=$(npm view @opentelemetry/auto-instrumentations-node dist-tags.latest)
echo "@opentelemetry/auto-instrumentations-node@${OTEL_AUTO_NODE_VERSION}"
```

If the app uses Fastify, also resolve:

```sh
FASTIFY_OTEL_VERSION=$(npm view @fastify/otel dist-tags.latest)
echo "@fastify/otel@${FASTIFY_OTEL_VERSION}"
```

If the app uses Prisma, also resolve:

```sh
PRISMA_INSTRUMENTATION_VERSION=$(npm view @prisma/instrumentation dist-tags.latest)
echo "@prisma/instrumentation@${PRISMA_INSTRUMENTATION_VERSION}"
```

#### PowerShell

```powershell
$OTEL_AUTO_NODE_VERSION = (npm view @opentelemetry/auto-instrumentations-node dist-tags.latest)
Write-Host "@opentelemetry/auto-instrumentations-node@$OTEL_AUTO_NODE_VERSION"
```

If the app uses Fastify, also resolve:

```powershell
$FASTIFY_OTEL_VERSION = (npm view @fastify/otel dist-tags.latest)
Write-Host "@fastify/otel@$FASTIFY_OTEL_VERSION"
```

If the app uses Prisma, also resolve:

```powershell
$PRISMA_INSTRUMENTATION_VERSION = (npm view @prisma/instrumentation dist-tags.latest)
Write-Host "@prisma/instrumentation@$PRISMA_INSTRUMENTATION_VERSION"
```

If any version fails to resolve, do not proceed with installation.

## Setup

Install the auto-instrumentation meta-package using the resolved version:

```bash
npm install @opentelemetry/auto-instrumentations-node@"${OTEL_AUTO_NODE_VERSION}"
```

### CJS vs ESM

The startup command depends on your application's module system — CommonJS (CJS) or ECMAScript Modules (ESM). An `.mjs` extension or `"type": "module"` in `package.json` indicates ESM. TypeScript projects can compile to either; check the `module` setting in `tsconfig.json`.

| Module system            | Startup command                                                                                                                         |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| CJS                      | `node --require @opentelemetry/auto-instrumentations-node/register app.js`                                                              |
| ESM (Node.js >= 18.19.0) | `node --experimental-loader=@opentelemetry/instrumentation/hook.mjs --import @opentelemetry/auto-instrumentations-node/register app.js` |

For TypeScript, compile first (`tsc`, `nest build`, etc.) and run the compiled entrypoint. The module system of the **compiled output** determines which flag to use.

For more detail on ESM support, see the [upstream documentation](https://github.com/open-telemetry/opentelemetry-js/blob/main/doc/esm-support.md).

That's it for **Express, NestJS, Koa, Hapi, and Restify**. The meta-package automatically instruments the HTTP layer, the web framework's routing, and the database driver. ORMs like Sequelize, TypeORM, and Knex are captured through their underlying driver instrumentation (`pg`, `mysql2`, etc.).

The following sections cover libraries that require additional setup.

## Fastify

Fastify's built-in OTel plugin, [`@fastify/otel`](https://github.com/fastify/fastify-otel), produces richer request-handler spans than the generic HTTP instrumentation alone. It is **not** included in the auto-instrumentations meta-package.

Install alongside the auto-instrumentations using the resolved version:

```bash
npm install @fastify/otel@"${FASTIFY_OTEL_VERSION}"
```

Register it as a Fastify plugin in your application code:

### CJS

```javascript
const { FastifyOtel } = require("@fastify/otel");
fastify.register(FastifyOtel, { serviceName: "<service-name>" });
```

### ESM / TypeScript

```typescript
import { FastifyOtel } from "@fastify/otel";
fastify.register(FastifyOtel, { serviceName: "<service-name>" });
```

Start with the same flags from [Setup](#setup). `@fastify/otel` reads from the global `TracerProvider` registered by the auto-instrumentations — no additional SDK setup is required.

## Prisma

Prisma ships its own instrumentation package, [`@prisma/instrumentation`](https://www.prisma.io/docs/orm/prisma-client/observability-and-logging/opentelemetry-tracing), which produces `prisma:engine:db_query` CLIENT spans with full SQL query text. It is **not** included in the auto-instrumentations meta-package.

Because `@prisma/instrumentation` must be registered programmatically, you need a short setup file instead of the `register` shorthand.

Install alongside the auto-instrumentations using the resolved version:

```bash
npm install @prisma/instrumentation@"${PRISMA_INSTRUMENTATION_VERSION}"
```

Create an instrumentation setup file at the project root:

**CJS** — `instrumentation.cjs`

```javascript
const { NodeSDK } = require("@opentelemetry/sdk-node");
const {
  getNodeAutoInstrumentations,
} = require("@opentelemetry/auto-instrumentations-node");
const { PrismaInstrumentation } = require("@prisma/instrumentation");

const sdk = new NodeSDK({
  instrumentations: [
    getNodeAutoInstrumentations(),
    new PrismaInstrumentation(),
  ],
});

sdk.start();
```

**ESM / TypeScript** — `instrumentation.mjs` or `instrumentation.ts`

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { PrismaInstrumentation } from "@prisma/instrumentation";

const sdk = new NodeSDK({
  instrumentations: [
    getNodeAutoInstrumentations(),
    new PrismaInstrumentation(),
  ],
});

sdk.start();
```

Start with `--require` or `--import` pointing at the setup file:

| Module system            | Startup command                                                                                            |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- |
| CJS                      | `node --require ./instrumentation.cjs app.js`                                                              |
| ESM (Node.js >= 18.19.0) | `node --experimental-loader=@opentelemetry/instrumentation/hook.mjs --import ./instrumentation.mjs app.js` |

If you are using **both** Fastify and Prisma, register `@fastify/otel` as a Fastify plugin in your application code. The setup file above already configures the global `TracerProvider` that `@fastify/otel` reads from.
