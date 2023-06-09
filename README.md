# EventCatalog with PNPM repro

Repro for showing the issue to make EventCalog work with PNPM.

- [EventCatalog with PNPM repro](#eventcatalog-with-pnpm-repro)
  - [Objective](#objective)
  - [Context](#context)
  - [Steps to reproduce](#steps-to-reproduce)
  - [Issue analysis](#issue-analysis)
  - [Workaround resolution](#workaround-resolution)

## Objective

Create an EventCatalog application within a pnpm workspace (monorepo) using the as much as possible the default configuration. The created application should work as whatever application created within a pnpm monorepo.

This implies:
- The application should be able to run in development mode
- The application should be able to run in production mode
- The application should be able to build
- The application should be able to packaged & distributed

## Context

| Package                                               | version |
| ----------------------------------------------------- | ------- |
| [`pnpm`](https://pnpm.io/)                            | `8.6.1` |
| [`@eventcatalog/core`](https://www.eventcatalog.dev/) | `0.6.8` |
| `@eventcatalog/type`                                  | `0.4.1` |

## Steps to reproduce

1. Create a new pnpm workspace
   ```bash
   pnpm init -w
   ```

2. Create an EventCatalog application
   ```bash
    cd ./apps
    pnpm dlx @eventcatalog/create-eventcatalog@latest docs-catalog
    ```

3. Remove npm related stuff
   ```bash
   rm -rf ./apps/docs-catalog/node_modules
   rm -rf ./apps/docs-catalog/package-lock.json
   ```

4. Install dependencies
   ```bash
   pnpm install
   ```

5. Build the application
   ```bash
    cd ./apps/docs-catalog
    pnpm run build
    ```

Using the default configuration, the build fails with the following error:

```bash
> docs-catalog@0.0.1 build /repos/repro-pnpm-peer-deps-wrong-path/apps/docs-catalog
> eventcatalog build


> @eventcatalog/core@0.6.8 scripts:move-schema-for-download
> node scripts/move-schemas-for-download.js


> @eventcatalog/core@0.6.8 build
> next build && next export

node:internal/modules/cjs/loader:1029
  throw err;
  ^

Error: Cannot find module '/repos/next@12.3.4_@babel+core@7.12.9_react-dom@17.0.2_react@17.0.2/node_modules/next/dist/bin/next'
    at Function.Module._resolveFilename (node:internal/modules/cjs/loader:1026:15)
    at Function.Module._load (node:internal/modules/cjs/loader:871:27)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:81:12)
    at node:internal/main/run_main_module:22:47 {
  code: 'MODULE_NOT_FOUND',
  requireStack: []
}
```

## Issue analysis

The issue is caused by the way pnpm handles the `node_modules` folder. By default, pnpm creates a `node_modules` folder in the root of the monorepo and symlinks the dependencies of each package to that folder. This is the default behaviour of pnpm and it is called _hoisting_.

EventCatalog use `NextJS`and the built core application in the folder `.eventcatalog-core` is not able to find the `next` dependency in the `node_modules` folder trying to resolvethe required deps in the wrong location even outside the monorepo

```
.
└── monorepo root/
    ├── apps/
    │   └── app/
    │       ├── .eventcatalog-core/
    │       │   └── node_modules/
    │       │       └── bin/
    │       │           └── next ----> ../../../../../../next@
    │       └── node_modules/
    │           └── bin/
    │               └── next ----> ../../../../node_modules/.pnpm/next@
    └── node_modules/
        └── .pnpm/
            └── next
```

## Workaround resolution

To solve the issue it is required to change the pnpm behavior for dependencies linking telling to pnpm 
use the `node-linker=hoisted`. You can find more information about this configuration in the [pnpm documentation](https://pnpm.io/npmrc#node-linker). 

To do that:

1. Create a `.npmrc` file in the root of the monorepo with the following content:
   ```bash
   node-linker=hoisted
   ```

After that clean the _cache_ and reinstall the dependencies:

1. Delete root `node_modules` folder
   ```bash
   rm -rf ./node_modules
   ```
2. Delete docs-catalog `node_modules` folder
   ```bash
   rm -rf ./apps/docs-catalog/node_modules
   re -rf ./apps/docs-catalog/.eventcatalog-core
   ```

3. Delete roor lock file
   ```bash
   rm -rf ./pnpm-lock.json
   ```

4. Install dependencies
   ```bash
    pnpm install
    ```

Now the docs-catalog app build should work as expected:

    ```bash
    cd ./apps/docs-catalog
    pnpm run build
    ```
