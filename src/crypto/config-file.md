# The config file

Every EscoDB store contains a file named `config` that is populated when the
store is first created. This file contains the keys necessary for various
operations, all of which are encrypted under the user's password. Any binary
values such as keys and salts are encoded using base64. A normal config file
looks like this:

```json
{
  "version": 1,
  "password": {
    "salt": "nCAFvP5M5bZlLRWdc9M+3QnLkX9+uKtovGA7GVk6foU=",
    "iterations": 600000
  },
  "cipher": {
    "key": "bym4sON3aF8GXpLzqZqxEmHll1G5e5r0zCN7kqdsnYCT+lHpC30sgFJ5D5FNTigb6BS9aZe5EHIDt7M1"
  },
  "auth": {
    "key": "0mB/FwRdeYF+qhHumLlhlXQVo46hCWRwTOsz2F8FZnvUu8e89rjVBSpX8bHKQjXJG54LP/9d5UlAjeOFCkOfR41ulKeGxpZyzm2PorywRy35ZI0hagU7EgEbKUo="
  },
  "shards": {
    "key": "LPDgFUUovpp7c7kJTO8DJymUq21ugDEovrQTXcibUHQ1SzuFw4PmxSqhMaUukA3PxX7wgaDYSMfyBQbMhvo3nlMGOA0kd2rjtn+QxKGEyA3VOfB9edBTd7gKIsA=",
    "n": 4
  }
}
```

The meaning of the fields is as follows:

- `version`: A value that tracks the schema version of the config file,
  currently the only valid value is `1`.

- `password`: Contains the 256-bit salt and iteration count that are used with
  PBKDF2-HMAC-SHA-256 to turn the password into the _User Key_; this key is then
  used to encrypt all the other keys in the config file.

- `cipher`: Contains the 256-bit _Root Key_ which is used to encrypt the _Shard
  Keys_ that are used inside each storage file. This key is encrypted under the
  User Key, with the binding context `{ file: 'config', scope: 'keys.cipher' }`.

- `auth`: Contains the 512-bit _Auth Key_ that is used with HMAC-SHA-256 to
  authenticate unencrypted metadata stored inside the storage files. This key is
  encrypted under the User Key, with the binding context `{ file: 'config',
  scope: 'keys.auth' }`.

- `shards`: Contains the number of shard files that the storage is divided into,
  and the 512-bit _Routing Key_ that is used with HMAC-SHA-256 to hash item
  paths to determine which shard file they are stored in. This usage of HMAC is
  not really security-critical, it serves as a convenient pseudo-random function
  for distributing items across shards in a manner that is randomised between
  different stores. The Routing Key is also encrypted under the User Key, with
  the binding context `{ file: 'config', scope: 'keys.router' }`.

All keys and salts contained in the config are independent random values. The
User Key (derived from the user's password) is only used to encrypt the keys in
this file, when the store is first created.
