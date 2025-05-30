# Performance optimisation

EscoDB has been designed to let applications keep their data in an arbitrary
object store, including those available through cloud hosting platforms. Access
to a remote service can involve significant latency, for example it is not
uncommon to see response times of 1-2 seconds when using the Dropbox API even on
a fast and stable internet connection. This can make it very expensive to access
multiple items in a store, and so we provide a way to automatically batch
operations together to massively reduce interaction with the storage layer.


## Storage model

To make the best use of the API, it helps to understand its storage model. An
EscoDB instance stores documents and directory information in a fixed number of
_shard files_. Each document is assigned to a shard by hashing its path,
producing a uniform and random distribution of documents across shards.

To execute a `get()` or `list()` call, the store needs to load the shard
containing the requested path into memory, parse it, and then locate the
requested path within it.

To execute an `update()` or `remove()` call, it needs to load the shards
containing the affected document and all its parent directories, update all of
them in memory, and then write the updated shards back. These writes need to be
performed in a particular order to preserve the internal consistency of the
store's data structures, so they cannot all be performed in parallel.

The storage backend implements
[compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap) (CAS),
wherein it will only accept a write if the client proves it has seen the latest
version of the file. When clients try to apply concurrent updates to the same
shard, some of these writes will be rejected and the whole `update()` or
`remove()` workflow needs to be retried, all of which adds to the latency for
successfully completing the operation.

EscoDB provides a way to reduce the cost of both reads and writes, by caching
shards in memory and transparently batching multiple document updates into the
same write to a shard file.


## The Task interface

EscoDB provides an abstraction called a _task_ for grouping multiple document
accesses and reducing the communication overhead with the storage backend. To
use it, perform the usual steps to open a store:

```js
let adapter = new FileAdapter({ path: 'path/to/store' })
let store = new Store(adapter, { key: { password: 'open sesame' } })
let handle = await store.open()
```

Once you have a handle from `Store.open()`, calling `task()` on it will create
and return a new _task_.

```js
let task = handle.task()
```

Tasks have the same interface as the main store handle: `get()`, `list()`,
`find()`, `update()`, `remove()` and `prune()` all work as they do normally. The
difference is in how the task changes the interaction with the storage adapter:

- All shard files are cached indefinitely. This means calls to `get()`, `list()`
  and `find()` will always return the same result for each document/directory,
  as updates made by other clients are not visible.

- Concurrent calls to `update()`, `remove()` and `prune()` are optimised into a
  small number of writes to the backend, while preserving any ordering needed
  for internal consistency.

Document updates still obey CAS; if a write to a shard file is rejected due to
concurrent writes by other clients/tasks, all updates targeting that shard will
be retried. Since this causes the shard to be reloaded, this is the only way
that changes from other clients become visible to a task.

### Tasks vs transactions

Note that a task is _not_ a transaction, it is a data access optimisation.
Document updates performed through a task are still independent of one another,
and it is possible for some to succeed while others fail.

The shard files are also independent objects and storage adapters do not offer
transaction semantics across them. This means that if a task updates multiple
documents, those updates are not atomic. Another client may observe some of
these updates before the task has completed all its planned updates.

In terms of consistency phenomena, EscoDB prevents lost updates to a single
document, but does not prevent write skew across documents. It also does not
provide snapshot isolation; it caches each shard on first access but does not
provide a consistent view across all shards at a single moment.
 

## Optimisation patterns

For many workloads, simply using a `handle.task()` to access the store rather
than the `handle` itself will make things faster. Any number of tasks can be
created from the same handle, and so a task also functions as a way to scope
cache lifetimes.

```js
let task1 = handle.task()

await task1.get('/alice')
await task1.get('/bob')

let task2 = handle.task()

await task2.get('/carol')
```

In this example, the reads of `/alice` and `/bob` share the same cache, while
the read to `/carol` uses its own cache. This lets you control when you'd like
data accesses to take advantage of caching, versus when you'd prefer to see
up-to-date content.

### Concurrent updates

For sequential updates, the use of a task provides _some_ benefit. In the
following example, every `update()` has to read the affected shards from the
backend and write them back again:

```js
let handle = await store.open()

await handle.update('/alice', () => ({ email: 'alice@example.com' }))
await handle.update('/bob', () => ({ email: 'bob@example.com' }))
await handle.update('/carol', () => ({ email: 'carol@example.com' }))
```

If a task is used here, then each shard is only read into memory once, when it
is first needed by any of the `update()` calls. But, each `update()` still makes
exactly the same writes as it did before.

```js
let handle = await store.open()
let task = handle.task()

await task.update('/alice', () => ({ email: 'alice@example.com' }))
await task.update('/bob', () => ({ email: 'bob@example.com' }))
await task.update('/carol', () => ({ email: 'carol@example.com' }))
```

A much more significant performance gain can be had by executing the updates
concurrently using the task:

```js
let handle = await store.open()
let task = handle.task()

await Promise.all([
  task.update('/alice', () => ({ email: 'alice@example.com' })),
  task.update('/bob', () => ({ email: 'bob@example.com' })),
  task.update('/carol', () => ({ email: 'carol@example.com' }))
])
```

When this is done, the updates also merge their changes into a small number of
shard file writes. A task can quite comfortably merge 100,000 document updates
together into a workload that reads each shard once and writes to each shard
just a couple of times.

Note that if the set of concurrent `update()` calls in the above example were
invoked on `handle`, rather than on `task`, they would actually be executed
sequentially. When invoked through `handle`, each `update()` call acts
independently with its own copy of the shard content. If they were allowed to
execute concurrently, their writes to the backend would likely conflict with
each other and cause a lot of retries before all of them eventually succeeded.
To produce a more optimal outcome, each `update()` is instead executed one at a
time.

### Execution planning

In general, any `update()` or `remove()` executed by a task will be merged into
whatever uncompleted work the task is already executing, to dynamically reduce
the total number of writes made. If a shard file write fails due to a CAS
violation, the retried operations are optimised in the same way.

One caveat to be aware of is that this optimisation involves building an
execution plan that is stored in memory. This plan increases in size with every
pending operation, and with the number of shards in the store. At a certain
number of pending updates, the cost of storing and updating this plan may become
prohibitive, but this can be avoided by putting a limit on the number of updates
put into a single batch.

```js
let handle = store.open()

let docs = {
  '/alice': { email: 'alice@example.com' },
  '/bob': { email: 'alice@example.com' },
  // ...
}

let task = handle.task()
let updates = []
let limit = 1e5

for (let [path, doc] of Object.entries(docs)) {
  updates.push(task.update(path, () => doc))

  if (updates.length >= limit) {
    await Promise.all(updates)
    updates = []
  }
}
```

Note that you do not need to create a fresh task when the `updates` list is
flushed. A task's execution plan only retains _pending_ operations in memory, so
a single task can be used indefinitely as long as its set of pending operations
does not get too large, and all pending operations eventually complete.
