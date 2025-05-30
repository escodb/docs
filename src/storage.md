# Storage adapters

EscoDB provides a number of storage adapters that can be passed to the `Store`
class to handle persistence. Two such adapters are included in the
`@escodb/core` package:

- `FileAdapter`: Stores content on the local filesystem. Most current
  applications will use this adapter.

- `MemoryAdapter`: Stores content in memory and does not write to any persistent
  store. This adapter is mainly used for testing; if your application uses a
  remote storage backend, it can be swapped out for this adapter and it will
  work identically.

Several other adapters are available via other packages:

- [`@escodb/couchdb-adapter`](https://github.com/escodb/couchdb-adapter):
  Adapter for storing content in [CouchDB](https://couchdb.apache.org/).

- [`@escodb/s3-adapter`](https://github.com/escodb/s3-adapter): Adapter for
  storing content in [Amazon S3](https://aws.amazon.com/s3/).


## Custom adapters

Support for any other storage system can be accomplished by implementing the
Storage Adapter API, which consists of the following methods:

- `read(id)`: Returns `Promise<{ value, rev }>` where `value` is a string
  representing the current content of the file or object named `id`. `rev` is a
  _revision token_ that identifies the current version of the file, and can be
  any type of value that the `write()` method knows how to handle. If the file
  does not currently exist, returns `Promise<null>`.

- `write(id, value, rev)`: If `rev` matches the current revision of the file
  named `id`, then it writes the string `value` to the file and returns
  `Promise<{ rev }>` where `rev` is the new revision token. Otherwise, it
  returns a `Promise` rejected with an error with `code = 'ERR_CONFLICT'`. If
  the input `rev` is `null` then the write must only succeed if the file does
  not currently exist.

Any object implementing these two methods can be used as a storage adapter. To
make sure that EscoDB's concurrency control works correctly, these methods must
obey the following constraints:

- The first `write(id, value, rev)` call to succeed for a given `id` must have
  `rev = null`. That is, the client must believe it is creating the file for the
  first time. If the file does not exist then any non-null `rev` input should be
  rejected.

- Any subsequent `write(id, value, rev)` call for that `id` must succeed only if
  `rev` is a revision token that has previously been returned by `read(id)` or
  `write(id, ...)`. That is, some client must previously have been told that
  `rev` was the current revision token.

- `write(id, value, rev)` must succeed at most once for each unique combination
  of `id` and `rev`, i.e. only one `value` is accepted for each version of a
  file.

- The `rev` returned by a successful call to `write(id, value, rev)` must be
  different from any previously returned `rev`. EscoDB includes a small amount
  of random content each time it updates a file, so this requirement can be met
  by using a collision-resistant content hash.

- After a `write(id, value, rev)` call succeeds, all subsequent calls to
  `read(id)` must return `value` until the next successful call to `write(id,
  ...)`.

- `write(id, value, rev)` must appear to be atomic to any external observer.
  That is, calls to `read(id)` must only ever return a `value` that was passed
  in some prior successful `write(id, value, rev)` call and must never return
  any intermediate or incomplete value.


## Security considerations

The storage layer in EscoDB is deliberately minimal. It does not have any
awareness of the data model built on top of it, the data access execution
planner, or any of the cryptographic techniques used to secure the data. It is
simply a place where blobs of data can be stored and retrieved.

Being an end-to-end encryption system, EscoDB places minimal trust in the
storage layer and tries to guard against an attacker who has access to the
storage server or to the communication channel between the client and the
server.

That being said, it is still a good idea to secure the storage backend as much
as you are able. The storage system should be protected by strong credentials
that grant access only to authorised users, it should be logically or physically
isolated from systems that do not require access to it, and all communication
with it should use an encrypted and authenticated transport mechanism such as
TLS or SSH.
