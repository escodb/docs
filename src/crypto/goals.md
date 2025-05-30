# Security goals

As an end-to-end encryption system, EscoDB performs all cryptographic operations
in the client, and trusts the storage backend only to be able to return data
that was previously written to it. It assumes that an attacker may have access
to the storage server or the communication channel between the client and
server, and aims to enforce these security goals under that assumption:

- The contents of a store may only be read and modified by somebody in
  possession of the correct store password. It is assumed that the password is
  only distributed to intended authorised users; knowledge of the password
  conveys access to the store. In particular the password is never given to the
  storage adapter and so cannot be transmitted to the storage server.

- An attacker is any person attempting to interact with the store without
  knowing the store password. We assume an attacker may be able to access the
  files on the storage server or in transit.

- An attacker must not be able to recover any of the store's encrypted content,
  which includes the root encryption and authentication keys in the config file,
  the shard keys in each shard file, or individual data items.

- An attacker must not be able to learn the ID or _path_ of any data item, nor
  reveal the directory structure of such paths.

- An attacker _may_ be able to tell roughly how many items are stored, along
  with the approximate byte length of each item.

- Any attempt by an attacker to deface or tamper with the store's content must
  be detectable by a legitimate user. Such tampering may include:
  - alteration of a specific ciphertext
  - substitution of an encrypted item for any subset of it
  - moving of a ciphertext to a different logical location
  - addition or removal of an encrypted item such as a shard key or document
  - reordering of keys or documents
  - changing any unencrypted stored value in a manner that undermines these
    goals

- An attacker must not be able to cause the client to encrypt and store any
  material, nor to authenticate any material, under a key selected by the
  attacker.
