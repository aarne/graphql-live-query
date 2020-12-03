# @n1ru4l/graphql-live-query-patch

[![npm version](https://img.shields.io/npm/v/@n1ru4l/graphql-live-query-patch.svg)](https://www.npmjs.com/package/@n1ru4l/graphql-live-query-patch) [![npm downloads](https://img.shields.io/npm/dm/@n1ru4l/graphql-live-query-patch.svg)](https://www.npmjs.com/package/@n1ru4l/graphql-live-query-patch)

Smaller live query payloads with [JSON patches (RFC6902)](https://tools.ietf.org/html/rfc6902).

When having big query results JSON patches might be able to drastically reduce the payload sent to clients. Everytime a new execution result is published a JSON patch is generated by diffing the previous and the next execution result. The patch operations are then sent to the client where they are applied to the initial execution result.

**Query**

```graphql
query post($id: ID!) @live {
  post(id: $id) {
    id
    title
    totalLikeCount
  }
}
```

**Initial result**

```json
{
  "data": {
    "post": {
      "id": "1",
      "title": "foo",
      "totalLikeCount": 10
    }
  },
  "revision": 1
}
```

**Patch result (increase totalLikeCount)**

```json
{
  "patch": [
    {
      "op": "replace",
      "path": "post/totalLikeCount",
      "value": 11
    }
  ],
  "revision": 2
}
```

For a full example usage check out the [todo-example client & server code](https://github.com/n1ru4l/graphql-live-query/tree/main/packages/todo-example).

## Install Instructions

```bash
yarn add -E @n1ru4l/graphql-live-query-patch
```

## API

### `createApplyLiveQueryPatchGenerator`

Wrap a `execute` result and apply a live query patch generator middleware.

```ts
import { execute } from "graphql";
import { applyLiveQueryPatchDeflator } from "@n1ru4l/graphql-live-query-patch";
import { schema } from "./schema";

const applyLiveQueryPatchGenerator = createApplyLiveQueryPatchGenerator();

const result = applyLiveQueryPatchGenerator(
  execute({
    schema,
    operationDocument: parse(/* GraphQL */ `
      query todosQuery @live {
        todos {
          id
          content
          isComplete
        }
      }
    `),
    rootValue: rootValue,
    contextValue: {},
    variableValues: null,
    operationName: "todosQuery",
  })
);
```

### `createApplyLiveQueryPatch`

Inflate the execution patch results on the client side.

```ts
import { createApplyLiveQueryPatch } from "@n1ru4l/graphql-live-query-patch";

const applyLiveQueryPatch = createApplyLiveQueryPatch();

const asyncIterable = applyLiveQueryPatch(
  # networkLayer.execute returns an AsyncIterable
  networkLayer.execute({
    operation: /* GraphQL */ `
      query todosQuery @live {
        todos {
          id
          content
          isComplete
        }
      }
    `,
  })
);
```

AsyncIterators make composing async logic super easy. In case your GraphQL transport does not return a AsyncIterator you can use the [`@n1ru4l/push-pull-async-iterable-iterator`](https://www.npmjs.com/package/@n1ru4l/push-pull-async-iterable-iterator) package for wrapping the result as a AsyncIterator.

```ts
import { createApplyLiveQueryPatch } from "@n1ru4l/graphql-live-query-patch";
import { makeAsyncIterableIteratorFromSink } from "@n1ru4l/push-pull-async-iterable-iterator";
import { createClient } from 'graphql-ws/lib/use/ws';

const client = createClient({
  url: "ws://localhost:3000/graphql"
});

const asyncIterableIterator = makeAsyncIterableIteratorFromSink(sink => {
  const dispose = client.subscribe(
    {
      query: "query @live { hello }"
    },
    {
      next: sink.next,
      error: sink.error,
      complete: sink.complete
    }
  );
  return () => dispose();
});

const applyLiveQueryPatch = createApplyLiveQueryPatch()
const wrappedAsyncIterableIterator = applyLiveQueryPatch(asyncIterableIterator)

for await (const value of asyncIterableIterator) {
  console.log(value);
}
```


### `createLiveQueryPatchGenerator`

In most cases using `createApplyLiveQueryPatchGenerator` is the best solution. However, some special implementations might need a more flexible and direct way of applying the patch middleware.

```ts
import { execute } from "graphql";
import { createLiveQueryPatchGenerator } from "@n1ru4l/graphql-live-query-patch";
import { schema } from "./schema";

execute({
  schema,
  operationDocument: parse(/* GraphQL */ `
    query todosQuery @live {
      todos {
        id
        content
        isComplete
      }
    }
  `),
  rootValue: rootValue,
  contextValue: {},
  variableValues: null,
  operationName: "todosQuery",
}).then(async (result) => {
  if (isAsyncIterable(result)) {
    const makePatches = createLiveQueryPatchGenerator();
    for (const value of makePatches(result)) {
      console.log(value);
    }
  }
});
```