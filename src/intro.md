# Introduction

EscoDB is a document store designed for privacy and portability. It provides
end-to-end encryption of all data and supports storing content on any object
store with conditional writes. It principally targets read-heavy applications
requiring storage of sensitive material, such as passwords, TOTP keys, API
tokens, private keys and other credentials.

Here is a brief overview of its API:

```js
let adapter = new FileAdapter({ path: 'path/to/store' })

let store = new Store(adapter, {
  key: { password: 'open sesame' }
})

let handle = await store.openOrCreate()

await Promise.all([
  handle.update('/contacts/alice', () => ({ email: 'alice@example.com' })),
  handle.update('/contacts/bob', () => ({ email: 'bob@example.com' }))
])

let contacts = await handle.list('/contacts/')
// -> ['alice', 'bob']

let alice = await handle.get('/contacts/alice')
// -> { email: 'alice@example.com' }
```

Its key features are:

- **Flexible and convenient data model**: EscoDB's interface is that of a
  key-value store, where the values are JSON-compatible documents, and keys
  resemble filesystem paths. This allows for grouping related items into
  directories, and efficiently listing and removing all the items within a
  directory.

- **End-to-end encryption of all content**: All content -- both keys and values
  -- is encrypted at rest and is resistant to tampering. Document keys and
  directory structure are not readable to anyone who does not know the store
  password. An attacker cannot modify the stored data without being detected.

- **Bring your own storage**: EscoDB is designed to run in a variety of
  environments and supports storing its data in any object store with
  conditional writes. This includes the local filesystem, cloud platforms like
  [Amazon S3](https://aws.amazon.com/s3/), [Google Cloud
  Storage](https://cloud.google.com/storage), or
  [Dropbox](https://www.dropbox.com/), or self-hosted services like
  [remoteStorage](https://remotestorage.io/) or
  [Solid](https://solidproject.org/). Users can add support for any data store
  by implementing a small plugin API.

- **Safe concurrent access**: By using the underlying data store's conditional
  write API, EscoDB ensures that multiple clients can concurrently update the
  store without losing updates. It ensures the internal consistency of its data
  model in the presence of concurrent updates to the same files, and partial
  failure of a client workload.

- **Highly optimised batch updates**: Being designed to interact with remote
  storage services with high latency, EscoDB transparently optimises document
  updates into as few requests as possible. It does this while preserving data
  consistency when clients perform concurrent updates to the same files.
