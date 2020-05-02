# React Fetching Hooks

[![npm](https://img.shields.io/npm/v/react-fetching-hooks)](https://www.npmjs.com/package/react-fetching-hooks)

> ⚠ This library is work-in-progress and published only for some real-life testing. Expect bugs, documentation inconsistency and breaking changes on every release until version 1.0.0.

Library for querying and mutating data for any backend, heavily inspired by [Apollo](https://www.apollographql.com/).

-   Simple and customizable (globally and per-request)
-   Full SSR support: requests on server, works anywhere (not only Next.js)
-   Caching with single shared cache: component's data can be updated by other query or mutation
-   Errors are also cached
-   Different fetch policies
-   Integration with redux-devtools (work in progress)
-   Network requests optimization and (some) race conditions handling
-   Tree-shakable (ES6 modules)
-   Written in typescript

## Quick start

Add the library and error serializer:

`yarn add react-fetching-hooks serialize-error`

Create a client:

```typescript
import {
    Client,
    Cache,
    ClientOptions,
    getIdUrl,
    getUrlDefault,
    processResponseRestfulJson,
    mergeShallow,
} from 'react-fetching-hooks';
import { serializeError, deserializeError } from 'serialize-error';

const SSR_MODE = typeof window === 'undefined';

let CLIENT_INSTANCE: Client;

export function getClient({ fetch }: Partial<ClientOptions> = {}) {
    // Return new client instance for every request in case of SSR
    if (SSR_MODE) {
        return createClient({ fetch });
    }

    // Return single client instance otherwise
    if (CLIENT_INSTANCE) {
        return CLIENT_INSTANCE;
    }

    // Create single client instance if there is no one
    return (CLIENT_INSTANCE = createClient({ fetch }));
}

function createClient({ fetch }: Partial<ClientOptions>) {
    return new Client({
        cache: new Cache({
            // If client-side, populate cache with preloaded data and errors
            initialSerializableState: !SSR_MODE ? window.__CACHE__ : undefined,
            // Enable redux-devtools integration
            enableDevTools: true,
            // Tell cache how to serialize errors. You can pass your own functions
            serializeError,
            deserializeError,
        }),
        // In case of SSR pass fetch (i.e. from node-fetch package)
        fetch,
        generalRequestData: {
            // Default base url
            root: 'http://localhost:3001',
            // Default fetch policy
            fetchPolicy: 'cache-and-network',
            // Treat all errors as data
            applyFetchPolicyToError: true,
            // Make it less possible to overwrite mutation result with old query
            rerunLoadingQueriesAfterMutation: true,
            // Default request id generator
            getId: getIdUrl,
            // Default request URL generator
            getUrl: getUrlDefault,
            // Default merger of general request data and concrete request data
            merge: mergeShallow,
            // Default response handler
            processResponse: processResponseRestfulJson,
        },
    });
}
```

Provide the client to your app via `Provider`:

```typescript jsx
import { Provider, Client } from 'react-fetching-hooks';
import React, { FC } from 'react';

interface AppProps {
    client: Client;
}

export const App: FC<AppProps> = ({ client }) => <Provider client={client}>Your app</Provider>;
```

Preload data and errors on server:

```typescript jsx
import { getDataFromTree, Client } from 'react-fetching-hooks';
import { renderToString } from 'react-dom/server';
import fetch from 'node-fetch';
import { getClient } from './path/to/getClient';
import { App } from './path/to/App';
import { getHtml } from './path/to/getHtml';

async function handleRequest(): Promise<string> {
    // Get client. Pass fetch, because there is no one in Node
    const client = getClient({ fetch });

    // Create App element, providing it with client
    const AppElement = React.createElement(App, { client });

    // Fill cache with data/errors
    await getDataFromTree(AppElement);

    // Get HTML to send
    const content = renderToString(AppElement);

    // Get data and errors as object. You can safely stringify it with JSON.stringify()
    const cache = client.extract();

    // Generate full page HTML that will include content and cache
    // For instance, you can make the cache available as window.__CACHE__
    return getHtml(content, cache);
}
```

Render the app in browser:

```typescript jsx
import * as React from 'react';
import * as ReactDOM from 'react-dom';
import { App } from './path/to/App';
import { getClient } from './path/to/getClient';

ReactDOM.hydrate(React.createElement(App, { client: getClient() }), document.getElementById('root'));
```

Query and mutate data in your components:

```typescript jsx
import React, { FC } from 'react';
import { PartialRequestData, useQuery, getIdBase64, ResponseError, useMutation } from 'react-fetching-hooks';

// Component props
interface Props {
    id: string;
}

// Data from API
interface ResponseData {
    field: string;
}

// Concrete request (query)
const dataQuery: PartialRequestData<any, ResponseData, ResponseError<any>, { id: string }> = {
    // Default path params to make typescript happy. You may choose different approach
    pathParams: { id: '1' },
    // Concrete request's url
    path: '/data/:id',
};

// Concrete request (mutation)
const dataMutation = { ...dataQuery, method: 'DELETE' };

const DataDisplay: FC<Props> = ({ id }) => {
    // You'll likely create your own useQuery and hide quirky logic there
    const { data } = useQuery(
        {
            ...dataQuery,
            // Override concrete request's path params with component's prop
            pathParams: { id },
        },
        {
            // Request id generator
            getPartialRequestId: getIdBase64,
        },
    );

    // You'll likely create your own useMutation that would accept PartialRequestData and return mutation state
    const { mutate } = useMutation();

    return (
        <div>
            {data}
            <button onClick={mutate({ ...dataMutation, pathParams: { id } })}>Delete</button>
        </div>
    );
};
```

## In-depth

### Queries and mutations

Queries and mutations use the same `RequestData` objects.

Rule of thumb: if a request doesn't change any backend state by the fact of its execution, it's a query. Otherwise, it's a mutation.

I.e. request, returning current time, is a query, even though it returns new data every time.

### RequestData

Requests are represented as `RequestData` objects. Due to flexibility, `RequestData` isn't a class, but rather a type defining a specific shape.

`RequestData` is native fetch's `RequestInit` with additional fields:

-   `fetchPolicy` - string, defines cache and network usage.
-   `root` - string, backend host, like `https://my-app.com`. Must be absolute on server, may be relative on client. Will be processed by `getUrl` function.
-   `path` - string, like `/data/:id`. Will be processed by `getUrl` function.
-   `pathParams` - object of params in path, like `{id: "1"}`. Will be processed by `getUrl` function.
-   `queryParams` - object of params in query. Will be processed by `getUrl` function.
-   `body` - body of the request.
-   `headers` - headers of the request.
-   `lazy` - boolean, lazy requests are not performed automatically.
-   `applyFetchPolicyToError` - boolean or function, if `true`, cached error will be treated as sufficient data for request, even if there is no cached data.
-   `rerunLoadingQueriesAfterMutation` - boolean, if `true` and this `RequestData` object is used in mutation, loading queries will be forced to refetch from network.
-   `getUrl` - function for generating request's URL.
-   `getId` - function for generating request's id. _Requests with the same id are considered the same_.
-   `processResponse` - function that returns data or throws an error based on request's `Response` (native).
-   `merge` - function that merges `GeneralRequestData` and `PartialRequestData` into `RequestData`.
-   `toCache`, `fromCache` - functions for writing data to and reading data from cache.

Every field may be configured either globally on `Client` instance or on concrete request.

`RequestData` consists of `GeneralRequestData` and `PartialRequestData`.

-   `GeneralRequestData` is defined on `Client` instance as `generalRequestData`. Essentially it contains default values for every request. You can define any `RequestData` fields except `path`.
-   `PartialRequestData` represents concrete request. All `RequestData` fields are optional except `path` (and possibly `pathParams`, `queryParams`, `body` and `headers`, depending on types).

### RequestData as generic

`RequestData` (as well as `GeneralRequestData` and `PartialRequestData`) is actually a generic:
`RequestData<SharedData, ResponseData, Error, PathParams, QueryParams, Body, Headers>`, where

-   `SharedData` - same for every request, type of shared cache.
-   `ResponseData` - type of successful response's data (returned by `processResponse` function).
-   `Error` - type of error, thrown by `processResponse`.
-   `PathParams`, `QueryParams`, `Body`, `Headers` - types of path params, query params, body and headers respectively.

### Request functions

All functions are replaceable. The library provides default/example ones:

-   `getIdUrl` - returns URL as id.
-   `getUrlDefault` - returns URL, using `query-string` and `path-to-regexp` packages. If you don't want to include them, you can provide your own function and they will be tree-shaked.
-   `mergeShallow` - merges `GeneralRequestData` and `PartialRequestData` objects shallowly.
-   `processResponseRestfulJson` - assumes that 2xx response is successful (and returns JSON), throws `ResponseError` on non-2xx response.

`toCache` and `fromCache` functions are completely up to you. Here are some rules:

-   Think in redux way. `toCache` should work like reducer. `fromCache` should return the same object every time.
-   They should never throw.
-   Return `undefined`, if there is no data in cache.

### Cache

Cache is responsible for storing all cacheable data.

Cache consists of two parts: `requestStates` and `sharedData`.

`requestStates` stores states of every individual request by its id. Request state consists of:

-   `data` - result of successful request from `processResponse` function.
-   `loading` - boolean.
-   `error` - error thrown from `processResponse` function.

`requestStates` is for internal usage only.

`sharedData` stores data from `processResponse` function, normalized by `toCache` function (from all requests). Shape of `sharedData` is completely up to you, but generally it should be as normalized and deduplicated as possible.

If request result updates `sharedData`, `data` field in `requestStates` is set to `undefined` to prevent duplication.

Cache options:

-   `serializeError` - function that converts error into serializable error object,
-   `deserializeError` - function that converts serializable error object into error,
-   `initialSerializableState` - object that will be used for cache initialization (both `requestStates` and `sharedData`). You can provide SSR result here. Defaults to empty cache.
-   `enableDevTools` - enable redux-devtools bindings. Defaults to `false`.
-   `enableDataDuplication` - if `true`, `data` field in `requestStates` will always be set. Defaults to `false`.

Queries are always cached in `requestStates`. If `toCache` is provided, the query will also be cached to `sharedData`.

Mutations are never cached to `requestStates`. If `toCache` is provided, the mutation will be cached to `sharedData`.

### Client

Client is responsible for performing requests. It also proxies cache (you should't work with cache directly).

On retrieving data from cache `fromCache` function is prioritized over `data` field from request state from `requestStates`.

You must provide new `Client` instance for every request in case of SSR.

Due to tree-shakeable design, you have to specify all parts of `generalRequestData` manually.

### Fetch policies

Queries:

-   `cache-only` - never makes network request. Returns data from cache or `undefined`.
-   `cache-first` - only makes network request, if there is no data in cache. Returns data from cache or `undefined` and then data/error from network, if there was network request.
-   `cache-and-network` - always makes network request. Returns data or `undefined` from cache and data/error from network.
-   `no-cache` - always makes network request. Returns data or `undefined` from cache and data/error from network. It's cached on per-caller basis and never touches `sharedData`, so initially it always returns `undefined`.

Mutations:

-   `cache-only`, `cache-first`, `cache-and-network` - returns data/error and updates `sharedData`
-   `no-cache` - returns data/error without touching `sharedData`

### Render details

Queries are rendered differently on mount and update:

-   On mount query is always rendered in non-loading state, if cached data is sufficient.
-   On update query is always rendered in loading state, if it's going to be loaded from network.

The difference may seem subtle, so consider an example:

-   Query with `fetchPolicy: 'cache-and-network'` has cached data.
-   It is mounted in non-loading state, so SSR result doesn't contain preloaders, and then it is updated to loading state because of the network request.
-   The network request is finished, and the query is updated into non-loading state.
-   Then it is updated by i.e. `pathParams` change into loading state, so it's impossible to have new `pathParams` and non-loading state. That allows things like `const loading = firstQueryLoading || secondQueryLoading`.

### useQuery

> You should build your own useQuery (using useQuery from the library) with API that is convenient for your app.

```typescript
const { data, loading, error, abort, refetch } = useQuery(request, { getPartialRequestId: getIdBase64 });
```

Options:

-   `getPartialRequestId` - function that calculates id of passed `PartialRequestData` object. It's optional with fallback to request's `getId`, but that's unreliable. In real world you should provide it (i.e. default `getIdBase64`).

Return values:

-   `data` - query data (actual or previous). May appear as (and never switches to) `undefined` (which means absence of data).
-   `loading` - boolean, indicating network request.
-   `error` - query error. Always swicthes to `undefined` on network request start.
-   `abort` - function for aborting query.
-   `refetch` - function for forcing network request.

Abort:

-   `abort()` without arguments won't abort request, if there are any other callers.
-   `abort(true)` will force abort for all callers.

Refetch:

-   `refetch()` without arguments won't start new network request if there is one already.
-   `refetch(true)` will always start new network request.

If some query is loading, and another caller (hook) executes the same query (using `RequestData` with the same id), this query is seamlessly merged with the first one, as if they were started simultaneously. This two queries will result in single network request.

### useMutation

> You should build your own useMutation (using useMutation from the library) with API that is convenient for your app.

```typescript
const { mutate } = useMutation();
```

-   `mutate` - function that performs mutation with given request.

## Known issues

-   Errors in external code may lead to different errors in state and promise rejections (i.e. `query().catch()`). It's likely not going to be fixed, but you should get diverged error warnings in development.
-   No reliable way to reset cache (i.e. on logout). You can call `cacheInstance.purge()` directly, but data from requests started before purge will affect the new cache. As a workaround, you can reload the page on logout.
-   All queries with the same request id are updated with loading state regardless of query initiator. That's probably undesirable, but fixing that would require SSR-friendly hook ids, which leads to API complication is case of code-splitting (see [react-uid](https://www.npmjs.com/package/react-uid)).
-   You can't return `undefined` as query data (and probably shouldn't, because empty data can be represented as `null`)
-   Race conditions handling is not 100% reliable, though no idea how to make it so (is it possible?).
-   Redux-devtools integration is limited.
-   No optimistic responses.
-   No `refetchQueries`.
-   It's probably better to move request functions to separate package.
-   Tests are really poor.
