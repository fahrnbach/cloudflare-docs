---
pcx_content_type: how-to
title: Deploy an Analog site
meta:
  description: The fullstack Angular meta-framework
---

# Deploy an Analog site

[Analog](https://analogjs.org/) is a fullstack meta-framework for Angular, powered by [Vite](https://vitejs.dev/) and [Nitro](https://nitro.unjs.io/).

In this guide, you will create a new Analog application and deploy it using Cloudflare Pages.

## Create a new project with `create-cloudflare`

The easiest way to create a new Analog project and deploy to Cloudflare Pages is to use the [`create-cloudflare`](https://www.npmjs.com/package/create-cloudflare) CLI (also known as C3). To get started, open a terminal and run:

{{<tabs labels="npm | pnpm | bun">}}
{{<tab label="npm" default="true">}}

```sh
$ npm create cloudflare@latest my-analog-app -- --framework=analog
```

{{</tab>}}
{{<tab label="pnpm">}}

```sh
$ pnpm create cloudflare@latest my-analog-app --framework=analog
```

{{</tab>}}
{{<tab label="bun">}}

```sh
$ bun create cloudflare@latest my-analog-app --framework=analog
```

{{</tab>}}
{{</tabs>}}

C3 will walk you through the setup process and create a new project using `create-analog`, the official Analog creation tool. It will also install the necessary adapters along with the [Wrangler CLI](/workers/wrangler/install-and-update/#check-your-wrangler-version).

{{<Aside type="note" header="Deployment">}}

The final step of the C3 workflow will offer to deploy your application to Cloudflare. For more information on deployment options, see the [Deployment](#deployment) section below.

{{</Aside>}}

## Bindings

{{<render file="/_framework-guides/_bindings_definition.md">}}

In Analog, server-side code can be added via [API Routes](https://analogjs.org/docs/features/api/overview). The `defineEventHandler()` method is used to define your API endpoints in which you can access Cloudflare's context via the provided `context` field. The `context` field allows you to access any bindings set for your application.

The following code block shows an example of accessing a KV namespace in Analog.

```typescript
---
filename: src/server/routes/v1/hello.ts
highlight: [2]
---
export default defineEventHandler(async ({ context }) => {
  const { MY_KV } = context.cloudflare.env;
  const greeting = (await MY_KV.get("greeting")) ?? "hello";

  return {
    greeting,
  };
});
```

### Setup bindings in development

Projects created via C3 come installed with a Nitro module that simplifies the process of working with bindings during development:

```typescript
---
filename: vite.config.ts
---
const devBindingsModule = async (nitro: Nitro) => {
  if (nitro.options.dev) {
    nitro.options.plugins.push('./src/dev-bindings.ts');
  }
};

export default defineConfig({
  ...
  plugins: [analog({
    nitro: {
      preset: "cloudflare-pages",
      modules: [devBindingsModule]
    }
  })],
  ...
});
```

This module in turn loads a plugin which adds bindings to the event context in dev:

```typescript
---
filename: src/dev-bindings.ts
---
import { NitroApp } from 'nitropack';
import { defineNitroPlugin } from 'nitropack/dist/runtime/plugin';

export default defineNitroPlugin((nitroApp: NitroApp) => {
  nitroApp.hooks.hook('request', async (event) => {
    const _pkg = 'wrangler'; // Bypass bundling!
    const { getPlatformProxy } = (await import(
      _pkg
    )) as typeof import('wrangler');
    const platform = await getPlatformProxy();

    event.context.cf = platform['cf'];
    event.context.cloudflare = {
      env: platform['env'] as unknown as Env,
      context: platform['ctx'],
    };
  });
});
```

In the code above, the `getPlatformProxy` helper function will automatically detect any bindings defined in your project's `wrangler.toml` file and emulate those bindings in local development. Review [Wrangler configuration information on bindings](/workers/wrangler/configuration/#bindings) for more information on how to configure bindings in `wrangler.toml`.

A new type definition for the `Env` type (used by `context.cloudflare.env`) can be generated from `wrangler.toml` with the following command:

```sh
$ npm run cf-typegen
```

This should be done any time you add new bindings to `wrangler.toml`.

### Setup bindings in deployed applications

In order to access bindings in a deployed application, you will need to [configure your bindings](/pages/functions/bindings/) in the Cloudflare dashboard.

## Deployment

When creating your new project, C3 will give you the option of deploying an initial version of your application via [Direct Upload](/pages/how-to/use-direct-upload-with-continuous-integration/). You can redeploy your application at any time by running following command inside your project directory:

```sh
$ npm run deploy
```

{{<render file="/_framework-guides/_git-integration.md">}}

### Create a GitHub repository

{{<render file="/_framework-guides/_create-gh-repo.md">}}

```sh
# Skip the following three commands if you have built your application
# using C3 or already committed your changes
$ git init
$ git add .
$ git commit -m "Initial commit"

$ git branch -M main
$ git remote add origin https://github.com/<YOUR_GH_USERNAME>/<REPOSITORY_NAME>
$ git push -u origin main
```

### Create a Pages project

1. Log in to the [Cloudflare dashboard](https://dash.cloudflare.com/) and select your account.
2. Go to **Workers & Pages** > **Create application** > **Pages** > **Connect to Git** and create a new Pages project.

You will be asked to authorize access to your GitHub account if you have not already done so. Cloudflare needs this so that it can monitor and deploy your projects from the source. You may narrow access to specific repositories if you prefer; however, you will have to manually update this list [within your GitHub settings](https://github.com/settings/installations) when you want to add more repositories to Cloudflare Pages.

3. Select the new GitHub repository that you created and, in the **Set up builds and deployments** section, provide the following information:

{{<pages-build-preset framework="analog">}}

Optionally, you can customize the **Project name** field. It defaults to the GitHub repository's name, but it does not need to match. The **Project name** value is assigned as your `*.pages.dev` subdomain.

4. After completing configuration, select the **Save and Deploy**.

Review your first deploy pipeline in progress. Pages installs all dependencies and builds the project as specified. Cloudflare Pages will automatically rebuild your project and deploy it on every new pushed commit.

Additionally, you will have access to [preview deployments](/pages/configuration/preview-deployments/), which repeat the build-and-deploy process for pull requests. With these, you can preview changes to your project with a real URL before deploying your changes to production.