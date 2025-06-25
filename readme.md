# sdk-code-generator

![test](https://github.com/ehmpathy/sdk-code-generator/workflows/test/badge.svg)
![publish](https://github.com/ehmpathy/sdk-code-generator/workflows/publish/badge.svg)

sdk codegen via endpoint introspection. generate a pit-of-success sdk for any api

# purpose

automate sdk maintenance via codegen to...
- sync types
- sync endpoints
- automatically

# install

```sh
npm install @ehmpathy/sdk-code-generator
```

# use

### **api-side**: generate the introspection endpoint

before an sdk can be generated, the api must be introspectable. to do so, create an endpoint that publishes a JsonSchema of your api.

this package can help do so automatically

#### 1. declare which endpoints to introspect

in your introspection config file, `codegen.sdk.ts`, specify where to find the endpoints to introspect

for example, choose endpoints via glob paths

```ts
// codegen.sdk.ts
import { ConfigIntrospectApi } from '@ehmpathy/sdk-code-generator';

export const introspect: ConfigIntrospectApi = {
  input: { glob: '@src/contract/endpoints/*' },
  output: '@src/contract/endpoints', // declare where to publish the introspect endpoint, as-endpoint
}
```

or

for example, choose endpoints via imports

```ts
// codegen.sdk.introspect.ts
import { ApiEndpoint } from 'as-endpoint';
import { ConfigIntrospectApi } from '@ehmpathy/sdk-code-generator';

import { endpoint as getPlantPot } from '@src/contract/endpoints/getPlantPot';
import { endpoint as setPlantPot } from '@src/contract/endpoints/setPlantPot';
import { endpoints } from '@src/contract/endpoints';

const toIntrospect: ApiEndpoint[] = [getPlantPot, setPlantPot, ...endpoints];

export const introspect: ConfigIntrospectApi = {
  input: toIntrospect,
  output: '@src/contract/endpoints', // declare where to publish the introspect endpoint, as-endpoint
}
```

##### 2. execute the introspection codegen

once you've declared which endpoints to introspect, you can invoke introspection via the below cli command

```sh
npx @ehmpathy/sdk-code-generator introspect -c codegen.sdk.introspect.ts
```

which will introspect your endpoints and create a new endpoint


### **sdk-side**: generate the sdk

now that an introspection endpoint is available on the api-side, we can leverage it to generate the sdk

#### 1. declare which apis to generate sdks for

in your config file, `codegen.sdk.ts`, specify which api's to generate sdk's for

```ts
import { ConfigGenerateSdk } from '@ehmpathy/sdk-code-generator';

export const generate: ConfigGenerateSdk = {
  input: {
    apis: [
      {
        interface: { lambda: 'svc-jokes' }
      },
      {
        interface: { lambda: 'svc-comedians' }
      },
      {
        interface: { lambda: 'svc-recommendation' }
      },
      // ...
    ]
  },
  output: '@src/data/sdk/',
}
```

#### 2. execute the sdk codegen

```sh
npx @ehmpathy/sdk-code-generator introspect -c codegen.sdk.introspect.ts
```

which will introspect and generate an sdk for each of the api's listed

for example

```ts
// @src/data/sdk/svcJokes.ts

import { createCache } from 'simple-in-memory-cache';
import { invokeLambdaFunction } from 'simple-lambda-client';
import { HasMetadata } from 'type-fns';

import { withExpectOutkey } from 'procedure-fns';
import { withExpectOutput } from 'procedure-fns';

const service = 'svc-jokes';

export interface SvcJokesJoke {
  uuid?: string;
  createdAt?: string;
  updatedAt?: string;

  semanticHash: string | null;
  variantHash: string | null;

  deliveryAudioUrl: string | null;
  deliveryVideoUrl: string | null;
  deliveryWords: string;
}
export class SvcJokesJoke implements DomainEntity<SvcJokesJoke> extends SvcJokesJoke {
  public static primary = ['uuid'] as const
  public static unique = ['semanticHash', 'variantHash'] as const
}

const getJoke = (event: {
  by: PickOne<{
    primary: RefByPrimary<typeof SvcJokesJoke>,
    unique: RefByUnique<typeof SvcJokesJoke>
  }>;
}): Promise<{
  joke: HasMetadata<SvcJokesJoke> | null;
}> =>
  invokeLambdaFunction({
    service,
    stage,
    function: 'getJoke',
    event,
    logDebug: log.debug,
  });


const setJoke = (event: {
  by: PickOne<{
    upsert: SvcJokesJoke,
    finsert: SvcJokesJoke,
    update: Ref<typeof SvcJokesJoke> & Partial<SvcJokesJoke>,
  }>;
}): Promise<{
  joke: HasMetadata<SvcJokesJoke> | null;
}> =>
  invokeLambdaFunction({
    service,
    stage,
    function: 'setJoke',
    event,
    logDebug: log.debug,
  });

export const svcHomeServices = {
  getJoke: withExpectOutkey(getJoke),
  setJoke,
};
```
