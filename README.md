# pocketbase-db-collection

A [PocketBase](https://pocketbase.io) collection adapter for [TanStack DB](https://tanstack.com/db). It lets you use a PocketBase `RecordService` as a real-time, local-first data source for a TanStack DB collection.

## Features

- Real-time sync via PocketBase's `subscribe()` mechanism
- Initial data fetch with `getFullList()` after subscribing (no missed events)
- Optimistic mutations forwarded to PocketBase (`create`, `update`, `delete`)
- Optional `StandardSchema` integration for typed records
- Automatic unsubscribe on collection cleanup

## Installation

```bash
npm install pocketbase-db-collection
# or
bun add pocketbase-db-collection
# or
pnpm add pocketbase-db-collection
```

You also need to install the peer dependencies if they are not already in your project:

```bash
npm install @tanstack/db pocketbase
```

## Peer dependencies

- `@tanstack/db` `>=0.6.0`
- `pocketbase` `>=0.26.0`

## Usage

### Basic

```typescript
import { createCollection } from '@tanstack/db'
import PocketBase from 'pocketbase'
import { pocketbaseCollectionOptions } from 'pocketbase-db-collection'

type Todo = {
  id: string
  title: string
  done: boolean
}

const pb = new PocketBase('http://localhost:8090')

const todos = createCollection(
  pocketbaseCollectionOptions({
    recordService: pb.collection<Todo>('todos'),
  }),
)

// Wait for the initial sync to complete
await todos.stateWhenReady()

// Read
const all = todos.toArray
const one = todos.get('record-id')

// Mutate — propagated to PocketBase, then synced back through the subscription
todos.insert({ id: '', title: 'Buy milk', done: false })
todos.update('record-id', (draft) => {
  draft.done = true
})
todos.delete('record-id')
```

### Passing PocketBase options

The `options` field is forwarded to both `getFullList()` and `subscribe()`. Use it for filters, expand, fields, etc.

```typescript
const todos = createCollection(
  pocketbaseCollectionOptions({
    recordService: pb.collection<Todo>('todos'),
    options: {
      filter: 'done = false',
      expand: 'author',
      sort: '-created',
    },
  }),
)
```

### With a Standard Schema

Any [Standard Schema](https://standardschema.dev) compatible validator (Zod, Valibot, ArkType, …) can be passed via the `schema` field for typed records and validated mutations.

```typescript
import { z } from 'zod'

const todoSchema = z.object({
  id: z.string(),
  title: z.string(),
  done: z.boolean(),
})

const todos = createCollection(
  pocketbaseCollectionOptions({
    recordService: pb.collection('todos'),
    schema: todoSchema,
  }),
)
```

## API

### `pocketbaseCollectionOptions(config)`

Returns a `CollectionConfig` that can be passed to TanStack DB's `createCollection()`.

| Field | Type | Description |
| --- | --- | --- |
| `recordService` | `RecordService<TItem>` | A PocketBase record service (`pb.collection('...')`). Required. |
| `options` | `RecordFullListOptions` | Optional. Forwarded to `getFullList()` and `subscribe()`. |
| `schema` | `StandardSchemaV1` | Optional. Provides typed records. |
| Other | — | Any other `BaseCollectionConfig` field from TanStack DB (e.g. `id`, `gcTime`, `startSync`, `autoIndex`, `compare`, `utils`, …) is forwarded as-is. |

The returned config:

- sets `getKey` to `(item) => item.id` (PocketBase's record id),
- registers a `sync` function that subscribes first, then performs an initial `getFullList()` fetch,
- registers `onInsert` / `onUpdate` / `onDelete` mutation handlers that call PocketBase's `create` / `update` / `delete`,
- unsubscribes when the collection is cleaned up.

## Development

### Prerequisites

- [Bun](https://bun.sh) `>= 1.2`

### Install dependencies

```bash
bun install
```

### Build

```bash
bun run build
```

Outputs ESM (`dist/esm/`), CJS (`dist/cjs/`), and TypeScript declarations.

### Watch mode

```bash
bun run dev
```

### Tests

```bash
bun test
```

### Lint / format

```bash
bun run lint
bun run format
```

## License

MIT
