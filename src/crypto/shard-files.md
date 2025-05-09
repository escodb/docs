# Shard files

Items stored in EscoDB live in _shard files_. Each item is assigned to a shard
by signing its path using HMAC-SHA-256 with the Routing Key. The first two bytes
of the hash are used to determine which shard the item resides in. If the config
file has `shards.n` set to `4`, then there will be four shard files with names
`shard-0000-3fff`, `shard-4000-7fff`, and so on.

## The shard header

The first line of each shard file contains a JSON header that looks something
like this:

```json
{
  "version": 1,
  "tag": "XmpfJjKen/k=",
  "cipher": {
    "keys": [
      "AAAAASwT71X33d2HO8kgkb0eGq6qWzi0LZ+Pap7BavPafsL9oKIvD0/xXnkmcDKixEKdVQVSHoZQ4JwOLF7Up1UU",
      "AAAAAiFQ2Ua4RI97avOsaBfu6zSOdB7/Xk+PokDULYtQ/Tnmm9iaHIdeDYYqPai8Fa1IdBUYREbHSfDIYTvOclij",
      "AAAAAwF0vhvn5BaVM85sf+3QteUwP6hQc+JY9frsXO3XiVVMy0x8gdUEg0ujUlW1orLDD1KPteFHHx6U8ExkbaTG"
    ],
    "state": "AAAAAIAAAAAAAAABRxcqXwAAAACAAAAAAAAAAUcW8sMAAAAAFXlWnwAAAAA239Ma",
    "mac": "kvAtihYSiF0uD/kaTvI+feZQcfkWXRZ6J4pzjY1gtB4="
  }
}
```

`version` specifies the schema version of the file, including this header. `tag`
is a random 64-bit value that is updated each time the file is written. This
ensures that storage backends that use a content hash as the file's "revision"
will report the file as changed even if none of its items were actually
modified. This is important for enforcing EscoDB's consistency properties.

The `cipher` section contains the following elements:

- `keys`: A list of base64-encoded Shard Keys, each containing the following
  elements:

  - A 32-bit sequential integer key ID. This uniquely identifies the key and is
    unencrypted.
  - A 16-bit integer specifying the encryption algorithm the key belongs to.
    Currently AES-256-GCM is the only supported algorithm.
  - A 256-bit encryption key.

  The algorithm ID and the key itself are encrypted under the Root Key with the
  binding context `{ file, scope: 'keys', key }` where `file` is the name of the
  shard file (e.g. `shard-8000-bfff`) and `key` is the key ID. This makes sure
  that key material genuinely corresponds to the unencrypted key ID, and the ID
  has not been tampered.

- `state`: A base64-encoded buffer containing several 64-bit counters. Each
  Shard Key has two counters in this buffer, one for the number of plaintexts it
  has encrypted, and one for the total number of blocks. These are updated
  whenever a value inside the shard is encrypted just before being written to
  storage. When either counter reaches a configured usage limit, a new Shard Key
  is generated with the next sequential key ID, and it is added to the `keys`
  list.

- `mac`: The HMAC-SHA-256 signature of the key IDs and usage counters. This is
  computed as follows:

  - Let `keys` be the concatenation of all the Shard Key IDs in the order they
    appear in the header. For example if there are three keys in the header,
    this would be the buffer `<00 00 00 01 00 00 00 02 00 00 00 03>`.
  - Let `state` be the base64-decoded `state` buffer.
  - Let `context` be `{ file, scope: 'keys', keys, state }` where `file` is the
    name of the shard file.
  - Then `mac` is the HMAC-SHA-256 signature of the canonical encoding of
    `context` using the Auth Key.

The signature ensures the integrity of those parts of the header that are stored
in plaintext. It makes sure the key IDs have not been modified and that no keys
have been added or removed, and it makes sure the counter state is genuine. This
is critical for the safe usage of the AES-GCM cipher.

### Key rotation rationale

Although we do not expect that in normal usage any shard would ever receive more
than 2<sup>32</sup> item updates, benchmarks indicate it is physically possible
to hit this limit in a matter of hours by spamming a store with document
updates. This could potentially be exploited by an attacker to force the
victim's store to reuse an IV and thereby break its encryption, and so we
implement key rotation to defend against this.

The Root Key is only used to encrypt newly generated Shard Keys, and so would
need to be rotated if more than 2<sup>32</sup> Shard Keys were created. If we
make a generous assumption about performance and assume that one can use up a
Shard Key in one hour, it would take around 490,000 years to run through
2<sup>32</sup> of them. Therefore usage-based key rotation is currently not
implemented on the Root Key, as this also might complicate the future
implementation of multi-user store access.

HMAC signatures are secure as long as there is negligible chance of a collision
on the hash output. With a 256-bit hash, keeping the collision probability to
less than 2<sup>-32</sup> allows a maximum of 2<sup>112</sup> signatures under
the same key. This is equivalent to updating a shard file ten million times per
nanosecond for the lifetime of the known universe, so we also do not implement
usage-based rotation of the Auth Key.

The key IDs and counter state are not encrypted because it would be impossible
or impractical to do so. The key IDs are used to identify which Shard Key was
used to encrypt each stored item, which would not work if the IDs were
themselves encrypted under that Shard Key. We could encrypt them under the Root
Key, but that would mean that the Root Key would be used every time a new item
was encrypted, and would therefore be used up at the same rate as Shard Keys.
Therefore we would have to implement rotation of the Root Key which simply moves
the problem to somewhere else.

Encrypting the counter state would add complexity. We cannot encrypt them under
the Root Key for the reason just given, and encrypting them under a Shard Key
would mean that the act of encrypting them would also change their state and
possibly require minting a fresh key. This is not impossible to work around but
would add unwarranted complexity. We don't regard the information about how many
items have been encrypted as secret, as it can be estimated by anyone with read
access to the file, even if they cannot decrypt any of its content.

### Concurrency control

EscoDB's consistency model is based around multiple clients interacting
concurrently with an [object
store](https://en.wikipedia.org/wiki/Object_storage) implementing
[compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap). Reading a
file from storage returns a `{ data, revision }` pair, where `revision` may be a
sequential version number, a content hash, or anything else that uniquely
identifies the current version of the file. Writing to storage only succeeds if
the client supplies a revision value matching the current stored version of the
file.

Under this model, if two clients want to update the same item, they will both
read the same shard file from storage and receive the file's current content and
revision. They will each update their copy of the shard in memory, encrypting
the item's new content under the latest Shard Key. But when writing their
updated shard back to storage, only one of these writes will be accepted, and
the other will be rejected, as the first accepted write updated the stored
file's revision.

    Client: Alice                      Storage                       Client: Bob
          |                               |                               |
          |            read()             |                               |
          | ----------------------------> |                               |
          | <---------------------------- |                               |
          |      { data: D, rev: X }      |                               |
          |                               |            read()             |
          |                               | <---------------------------- |
          |                               | ----------------------------> |
          |                               |      { data: D, rev: X }      |
          |     write(data: A, rev: X)    |                               |
          | ----------------------------> |                               |
          | <---------------------------- |                               |
          |         { ok, rev: Y }        |                               |
          |                               |     write(data: B, rev: X)    |
          |                               | <---------------------------- |
          |                               | ----------------------------> |
          |                               |             error             |

When this happens, the client whose write was rejected will reload the shard
from storage and attempt to re-apply its desired changes atop the updated state.

    Client: Alice                      Storage                       Client: Bob
          |                               |                               |
          |                               |            read()             |
          |                               | <---------------------------- |
          |                               | ----------------------------> |
          |                               |      { data: A, rev: Y }      |
          |                               |                               |
          |                               |                               |
          |                               |     write(data: C, rev: Y)    |
          |                               | <---------------------------- |
          |                               | ----------------------------> |
          |                               |         { ok: rev: Z }        |

Although Bob had his write rejected, we still count the encryptions that were
done in that write even though they were never persisted. If a ciphertext is
sent over the channel between the client and storage, it may be observed by an
attacker and should still count towards the key's usage limit. Therefore, when
Bob reloads the shard from storage and re-applies his changes on top of the new
state, he also merges the state of his key counters with the ones retrieved from
storage.

For example, imagine the shard in storage currently has a key that has been used
to encrypt 10 items. When Alice and Bob participate in this race to update the
shard, the following happens:

- Alice reads the shard from storage at revision `X` and observes its key usage
  counter has value 10.
- Bob reads the shard from storage, also at revision `X`, and also sees the
  counter value of 10.
- Alice updates two items and encrypts them both before the shard is written
  back to storage. She updates the counter to 12 to reflect this.
- Bob updates three items and attempts to write the shard back. But since his
  revision is now stale, the write is rejected.
- Bob reloads the shard from storage at revision `Y` and sees the key counter
  set to 12. He re-applies his increment of 3 from the failed write, bringing
  the value up to 15.
- Bob re-applies his changes to the three items and increases the counter value
  to 18.
- Bob writes his copy of the shard back to storage, this time successfully.

These steps are illustrated in the following sequence diagram.

    Client: Alice                      Storage                       Client: Bob
          |                               |                               |
          |            read()             |                               |
          | ----------------------------> |                               |
          | <---------------------------- |                               |
          |     { rev: X, count: 10 }     |                               |
          |                               |            read()             |
          |                               | <---------------------------- |
          |                               | ----------------------------> |
          |                               |     { rev: X, count: 10 }     |
          |    write(rev: X, count: 12)   |                               |
          | ----------------------------> |                               |
          | <---------------------------- |                               |
          |         { ok, rev: Y }        |                               |
          |                               |    write(rev: X, count: 13)   |
          |                               | <---------------------------- |
          |                               | ----------------------------> |
          |                               |             error             |
          |                               |                               |
          |                               |                               |
          |                               |            read()             |
          |                               | <---------------------------- |
          |                               | ----------------------------> |
          |                               |     { rev: Y, count: 12 }     |
          |                               |                               |
          |                               |                               |
          |                               |    write(rev: Y, count: 18)   |
          |                               | <---------------------------- |
          |                               | ----------------------------> |
          |                               |         { ok: rev: Z }        |

This mechanism makes sure that encryptions done when clients are concurrently
updating the same shard are recorded in the usage count. If a client's write is
rejected by the storage for any other reason, for example due to an
authorisation error, their changes to the counter cannot be persisted.

## Index and items

Following the header, the rest of the lines in a shard file are the _index_ and
the _items_. This data will typically look something like this:

```
AAAAAUYncFOOgsluuk0fI2JJWmLY2GASZCrHNTNJXPAk2/fv2NhGBUl6LS6+Yj60LdwNgpHlWAYkI2HfPo0=
AAAAAXPs+6XpqmWnCUHsgVFm49ZYrC5zdMBxQ6YXdORhpDVZi8RRIIUzdJC5uFx0aG5xi22skw==
AAAAAWaeds4hrRQ36HvOxu0ZvtnIkUcrmkZRnp7r6PDXTs09woIu1tZXNzulusggks7pYY+giXwBrft7o1ycg9UJb7Q=
AAAAAYsnsxumiH44cNLL6/I98cQS5+yzpNU0EFEN7ZoQ6a8luwBFHiN2K8lQlygqZWZGVtQViTTcbuuCwlVV
AAAAAfFchyrM1DHQ+0YFR2qA2X4CK88tiClaTM+Uk7TPcTf6Ui/VhEo+RBkUM43xdxr9TF1gK4hJBbN/u8I6SQ==
```

Each line consists of a base64-encoded buffer containing an unencrypted 32-bit
key ID and the encrypted content of the item. The key ID identifies which of the
keys in the header `cipher.keys` list the item is encrypted with.

The index is a sorted list of the paths of all the items in the shard. It exists
so that EscoDB can locate an item by its path without having to decrypt all the
items themselves. It is encrypted with the binding context `{ file, scope:
'index', key }` where `file` is the name of the shard file and `key` is the key
ID at the start of the line.

The items are the documents added to the store via calls to `store.update()`,
and the directory indexes that list the names of all items that appear in each
directory. Each one is encrypted with the binding context `{ file, scope:
'items', path, key }` where `path` is the item's path and `key` is the ID of the
encryption key.

The binding of items prevents _confused deputy_ attacks wherein an attacker
cannot decrypt any content, but can swap the order of ciphertexts in the file or
move ciphertexts between files. For example, if EscoDB is used as the backend
for a password manager, this would allow an attacker to swap the victim's
passwords for two different services, and thereby trick them into disclosing
their email password to a different website. Including the item's path in the
binding context ensures that the item only decrypts if it is genuinely the
document for the requested path.

The index is decrypted every time the shard is loaded from storage, while
specific items are only decrypted when a client accesses them via a `get()`,
`list()` or `find()` call. The index and items are freshly encrypted just before
the shard is written back to storage, if their value has been changed.
