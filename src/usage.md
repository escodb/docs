# Usage

The `@escodb/core` package exports a few different classes:

```js
const { Store, FileAdapter } = require('@escodb/core')
```

The full set of exported classes is:

- `Store`: the core class representing an EscoDB instance backed by a specific
  storage location. This is the main entry point for all interactions with a
  store.

- `FileAdapter`: storage adapter for storing content on the local filesystem.
  Most current applications will use this adapter.

- `MemoryAdapter`: storage adapter that stores content in memory and does not
  write to any persistent store. This adapter is mainly used for testing; if
  your application uses a remote storage backend, it can be swapped out for this
  adapter and it will work identically.


## Creating and opening a store

All interaction with a store begins by creating a storage adapter, and then
creating a `Store` instance with this adapter and the password needed to access
it.

```js
let adapter = new FileAdapter({ path: 'path/to/files' })

let store = new Store({
  key: { password: 'open sesame' }
})
```

The `FileAdapter` class takes the following options:

- `path` (required): The pathname to the directory where the store's files will
  be kept. The current process's user must have permission to read and write in
  this directory.

- `fsync`: Whether to call `fsync(2)` when writing to files. This is more robust
  but adds significant latency. For maximal safety, this is `true` by default
  and EscoDB optimises document updates into as few disk accesses as possible to
  minimise the latency impact. In some scenarios, for example when running unit
  tests, it may be desirable to disable `fsync` to reduce latency.

The `Store` class takes the following options:

- `key.password`: The password that is used to encrypt the store's root
  encryption keys. This should be a high-entropy unique password that cannot be
  guessed or brute-forced by an attacker.

EscoDB may support other types of access control in future, such as a raw
symmetric encryption key or a user's asymmetric key pair. For now, passwords are
the only supported access method.

### Creating a new store

If nothing has yet been written to the store, i.e. the target directory contains
no files, then it must be created:

```js
let handle = await store.create({
  password: { iterations: 1000000 },
  shards: { n: 2 }
})
```

This performs the initial setup for the store: generating a set of cryptographic
keys and related configuration data, and writing this config to disk. The
options are:

- `password.iterations`: The number of rounds used to convert the password to an
  encryption key via PBKDF2-SHA-256. This currently defaults to 600,000.

- `shards.n`: The number of _shard_ files that the store will spread documents
  across. Increasing the number of shards allows the store to perform more
  concurrent data access and decreases the time to read and write shards as each
  file contains a smaller slice of the whole data set. However it also increases
  the number of disk accesses (or HTTP requests for remote stores) required to
  access multiple items. Setting the number of shards is a trade-off between
  these factors. The default number is `4`.

If the store already exists, then `Store.create()` will throw an error.

### Opening an existing store

If the store has already been initialised, then it needs to be opened:

```js
let handle = await store.open()
```

This reads the config into memory and uses the password to decrypt the store's
root keys. This involves a slow hash of the store password but this only needs
to be done once when the store is first opened, not when individual items are
accessed.

If the store has not been initialised then `Store.open()` will throw an error.

As a convenience, EscoDB provides a method that will open the store if it
exists, otherwise it will initialise it with the given options and then open it.

```js
let handle = await store.openOrCreate({
  password: { iterations: 1000000 },
  shards: { n: 2 }
})
```

Note that if the store already exists, the options passed to `openOrCreate()`
will have no effect; they will not be used to update the store's settings and
will simply be ignored.


## Working with documents

Documents in EscoDB are indexed by a _path_, a string resembling an absolute
filesystem path. Paths must begin with a slash, and may contain letters, digits,
dashes, underscores and dots, and they may contain multiple _segments_ separated
by slashes. `/info.txt`, `/phone-numbers` and `/bob/avatar.png` are all valid
document paths.

A _document path_ must not end with a trailing slash. A _directory path_ does
end with a trailing slash; such a path refers to the set of documents whose
paths have it as a prefix. For example, the directory `/bob/` contains the
document `/bob/avatar`, while the directory `/` contains the document `/index`
and the directory `/bob/`.

Directories are created implicitly when documents are updated; they do not need
to be explicitly created. A directory is simply a cache of the document paths
that begin with a certain prefix, which means the store can efficiently return a
list of the directory's children without scanning the whole store.

It is legal for a directory to have the same textual name as a document;
`/users` and `/users/` are distinct paths that refer to a document and a
directory respectively. Thus it is legal to have documents named `/users` and
`/users/alice` co-exist.

The store handle returned by `Store.open()` or `Store.create()` supports the
following methods.

### `handle.get(path)`

Returns the current content of the document at `path`, or `null` if the document
does not exist.

```js
await handle.get('/users/alice')
// -> null

await handle.update('/users/alice', () => ({ name: 'Alice' }))

await handle.get('/users/alice')
// -> { name: 'Alice' }
```

### `handle.list(path)`

Returns the names of all the documents and subdirectories that are direct
children of the directory at `path`, or `null` if the directory does not exist.

```js
await handle.list('/users/')
// -> null

await handle.update('/users/alice', () => ({ name: 'Alice' }))
await handle.update('/users/bob/contacts', () => ({ email: 'bob@example.com' }))

await handle.list('/users/')
// -> ['alice', 'bob/']
```

### `handle.find(path)`

Returns an async iterator containing the names of all the documents that are
descendants of the directory at `path`.

```js
await handle.update('/users/alice', () => ({ name: 'Alice' }))
await handle.update('/users/bob/contacts', () => ({ email: 'bob@example.com' }))

let paths = []

for await (let path of handle.find('/users/')) {
  paths.push(path)
}

// -> paths = ['alice', 'bob/contacts']
```

Note that `find()` enumerates only documents that reside under the given
directory, it does not emit intermediate directory names. It returns an async
iterator rather than a single `Promise` because it requires traversing multiple
directory entries in the store, rather than reading the content of a single
directory.

### `handle.update(path, fn)`

Updates the document at `path` by applying the function `fn` to its current
state.

```js
// create a new doc, or ignore its current content

await handle.update('/users/alice', () => {
  return { emails: ['alice@example.com'] }
})

// update a doc based on its current state

let doc = await handle.update('/users/alice', (doc) => {
  let emails = [...doc.emails, 'work.address@example.org']
  return { ...doc, emails }
})

// -> doc = { emails: ['alice@example.com', 'work.address@example.org'] }
```

The `update()` method takes a function rather than a simple document value in
order to support concurrency. If two clients try to update the same file at the
same time, the storage layer will return an error to one of them and that client
will retry. It does this by reloading the file containing the other client's
update, and then re-applying `fn` to the document.

This means that `fn` may be executed multiple times, but only its final output
will be persisted. That is, once the storage backend accepts the client's write,
it will not apply the same update to the document again. For example, if three
clients concurrently execute the function `update('/counter', (doc) => ({ n:
doc.n + 1 }))`, then the `n` field in the document `/counter` will be
incremented by `3`, even though each client may execute the update function
multiple times as they all race to update the same file.

The same thing applies if the same client makes multiple concurrent updates of
its own:

```js
await handle.get('/counter')
// -> { n: 1 }

await Promise.all([
  handle.update('/counter', (doc) => ({ n: doc.n + 1 })),
  handle.update('/counter', (doc) => ({ n: doc.n + 1 })),
  handle.update('/counter', (doc) => ({ n: doc.n + 1 }))
])

await handle.get('/counter')
// -> { n: 4 }
```

`update()` returns a `Promise` that resolves with the new state of the document
when the document has been successfully written to the storage backend. If
writing to storage raises any error other than a conditional write conflict, the
`Promise` is rejected with this error.

### `handle.remove(path)`

Unconditionally removes the document from the store.

```js
await handle.get('/users/alice')
// -> { name: 'Alice Smith' }

await handle.remove('/users/alice')

await handle.get('/users/alice')
// -> null
```

Like `update()`, the `remove()` method returns a `Promise` that resolves when
its effect is successfully persisted, or rejects with an error if writing the
update to storage fails.

### `handle.prune(path)`

Recursively removes all the documents under the directory `path`.

```js
await handle.get('/users/alice')          // -> { name: 'Alice' }
await handle.get('/users/bob/contacts')   // -> { email: 'bob@example.com' }
await handle.list('/users/')              // -> ['alice', 'bob/']

await handle.prune('/users/')

await handle.get('/users/alice')          // -> null
await handle.get('/users/bob/contacts')   // -> null
await handle.list('/users/')              // -> null
```

To remove all the documents in the store, call `prune('/')`.
