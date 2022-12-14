<p align="center">
  <img src="https://avatars.githubusercontent.com/u/10090828?s=200&v=4" width="200px" alt="logo"/>
</p>
<h1 align="center">@immobiliarelabs/backstage-plugin-ldap-auth-backend</h1>

[![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](https://github.com/prettier/prettier?style=flat-square)
[![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg?style=flat-square)](https://github.com/semantic-release/semantic-release)
![license](https://img.shields.io/github/license/immobiliare/backstage-plugin-ldap-auth?style=flat-square)
![npm (scoped)](https://img.shields.io/npm/v/@immobiliarelabs/backstage-plugin-ldap-auth-backend?style=flat-square)

> Customizable Authentication backend provider for LDAP servers for your [Backstage](https://backstage.io/) deployment

Works either on simple stand-alone process or scaled infrastracture spanning multiple deployments using the shared PostgreSQL instance that Backstage already uses!

This plugin is not meant to be used alone but in pair with:

-   The official [@backstage/plugin-catalog-backend-module-ldap](https://www.npmjs.com/package/@backstage/plugin-catalog-backend-module-ldap) which keeps in sync your LDAP users with Backstage user catalogs!
-   Its sibling frontend package [@immobiliarelabs/backstage-plugin-ldap-auth](https://www.npmjs.com/package/@immobiliarelabs/backstage-plugin-ldap-auth)

All the current LTS versions are supported.

## Table of Content

<!-- toc -->

-   [Installation](#installation)
-   [Configurations](#configurations)
    -   [Connection Configuration](#connection-configuration)
    -   [Setup Backstage official LDAP plugin](#setup-backstage-official-ldap-plugin)
    -   [Add authentication backend](#add-authentication-backend)
    -   [Add the login form](#add-the-login-form)
-   [Powered Apps](#powered-apps)
-   [Support & Contribute](#support--contribute)
-   [License](#license)

<!-- tocstop -->

## Installation

> These packages are available on npm.

You can install them in your backstage installation using `yarn workspace`

```bash
# install yarn if you don't have it
$ npm install -g yarn
# install backend plugin
$ yarn workspace backend add @immobiliarelabs/backstage-plugin-ldap-auth-backend
# install frontend plugin
$ yarn workspace app add @immobiliarelabs/backstage-plugin-ldap-auth
```

## Configurations

> This documentation assumes that you have already scaffolded your Backstage instance from the official `@backstage/create-app`, all files that we're going to customize here are the one already created by the CLI!

### Connection Configuration

> Adds connection configuration inside your backstage YAML config file, eg: `app-config.yaml`

This fields are all required

| key                  | description                                                      |
| -------------------- | ---------------------------------------------------------------- |
| `url`                | Array of ldap connection strings                                 |
| `rejectUnauthorized` | Reject non HTTPS traffic, this also set secure cookies when true |
| `userDn`             | User distinguished name directory location                       |
| `userSearchBase`     | Userbase search location                                         |

```yml
auth:
    environment: ENV_NAME
    providers:
        ldap:
            ENV_NAME:
                url:
                    - 'ldaps://123.123.123.123'
                rejectUnauthorized: true
                userDn: 'ou=usr,dc=ns,dc=frm'
                userSearchBase: 'dc=ns,dc=frm'
```

### Setup Backstage official LDAP plugin

If you didn't have already, we need to configure the official LDAP plugin to imports and keep in syncs users

> `packages/backend/src/plugins/catalog.ts`

```ts
import type { Router } from 'express';
import type { PluginEnvironment } from '../types';

import { CatalogBuilder } from '@backstage/plugin-catalog-backend';
import { ScaffolderEntitiesProcessor } from '@backstage/plugin-scaffolder-backend';
import {
  LdapOrgEntityProvider,
} from '@backstage/plugin-catalog-backend-module-ldap';

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  const builder = await CatalogBuilder.create(env);

  builder.addEntityProvider(
    LdapOrgEntityProvider.fromConfig(env.config, {
      id: '<YOUR-ID>',
      target: 'ldaps://<YOUR-ADDRESS>',
      logger: env.logger,
      schedule: env.scheduler.createScheduledTaskRunner({
        frequency: // whatever
        timeout: // whatever
      }),
    }),
  );

  builder.addProcessor(new ScaffolderEntitiesProcessor());
  const { processingEngine, router } = await builder.build();
  await processingEngine.start();
  return router;
}
```

### Add authentication backend

This assumes a basic usage: single process without custom auth function or user object customization and in-memory token storage

For more uses cases you can see the [example folders](https://github.com/immobiliare/backstage-plugin-ldap-auth/examples/)

> `packages/backend/src/plugins/auth.ts`

```ts
import { createRouter } from '@backstage/plugin-auth-backend';
import { Router } from 'express';
import { PluginEnvironment } from '../types';
import { ldap } from '@immobiliarelabs/backstage-plugin-ldap-auth-backend';

export default async function createPlugin(
    env: PluginEnvironment
): Promise<Router> {
    return await createRouter({
        logger: env.logger,
        config: env.config,
        database: env.database,
        discovery: env.discovery,
        tokenManager: env.tokenManager,
        providerFactories: {
            ldap: ldap.create({}),
        },
    });
}
```

### Add the login form

> More on this in the frontend plugin documentation [here](../ldap-auth/README.md)

We need to replace the existing Backstage demo authentication page with our custom one!

In the `App.tsx` file, change the `createApp` function adding a `components` with our custom `SignInPage`In the `App.tsx` file change the `createApp` function to provide use our custom `SignInPage` in the `components` key.

**Note:** This components isn't only UI, it also brings all the token state management and HTTP API calls to the backstage auth routes we already configured in the backend part.

> `packages/app/src/App.tsx`

```tsx
import { LdapAuthFrontendPage } from '@immobiliarelabs/backstage-plugin-ldap-auth';

const app = createApp({
    // ...
    components: {
        SignInPage: (props) => (
            <LdapAuthFrontendPage {...props} provider="ldap" />
        ),
    },
    // ...
});
```

And you're ready to go! If you need more use cases, like having multiple processes and need a shared token store instead of in-memory look at the [example folders](https://github.com/immobiliare/backstage-plugin-ldap-auth/examples/)

## Powered Apps

Backstage Plugin LDAP Auth was created by the amazing Node.js team at [ImmobiliareLabs](http://labs.immobiliare.it/), the Tech dept of [Immobiliare.it](https://www.immobiliare.it), the #1 real estate company in Italy.

We are currently using Backstage Plugin LDAP Auth in our products as well as our internal toolings.

**If you are using Backstage Plugin LDAP Auth in production [drop us a message](mailto:opensource@immobiliare.it)**.

## Support & Contribute

Made with ?????? by [ImmobiliareLabs](https://github.com/immobiliare) & [Contributors](https://github.com/immobiliare/backstage-plugin-ldap-auth/CONTRIBUTING.md#contributors)

We'd love for you to contribute to Backstage Plugin LDAP Auth!
If you have any questions on how to use Backstage Plugin LDAP Auth, bugs and enhancement please feel free to reach out by opening a [GitHub Issue](https://github.com/immobiliare/backstage-plugin-ldap-auth).

## License

Backstage Plugin LDAP Auth is licensed under the MIT license.  
See the [LICENSE](https://github.com/immobiliare/backstage-plugin-ldap-auth/LICENSE) file for more information.
