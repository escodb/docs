# Cryptography

EscoDB is a document store designed to keep its contents private. It achieves
this via two key features: it lets the user choose where its contents are
physically stored, and it encrypts everything input by the user, including
document IDs (referred to as _paths_ in the API).

It also aims to protect stored data from certain forms of tampering. While
encryption prevents an attacker from being able to read the contents of your
files, tamper resistance means they cannot deface data in certain ways, even if
they are unable to decrypt it. They key thing that EscoDB prevents is the
ability to move encrypted records inside a file so that a document becomes
readable via a different ID than the one it was written to. This is achieved by
appropriate use of _authenticated encryption with associated data_ (AEAD).

EscoDB cannot protect you from arbitrary changes to its data files, including
partial or total deletion of their content. You must still use appropriate
measures to prevent unauthorised access to data in transit and at rest.

The following pages describe how EscoDB uses cryptography in the data it stores,
including why certain choices have been made.


## Algorithms used

EscoDB makes use of the following cryptographic algorithms:

- AES-256-GCM
- HMAC-SHA-256
- PBKDF2

The implementation of all these algorithms is provided by the host platform; on
Node.js we use the [crypto](https://nodejs.org/api/crypto.html) module and in
the browser we use the [Subtle
Crypto](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto) API.

Keys, initialisation vectors (IVs), salts, and other random values are produced
in one of two ways. If the platform provides a dedicated `generateKey()`
function for the relevant algorithm, then that function is used. Otherwise, the
`crypto.randomBytes()` or `crypto.getRandomValues()` function is used.

Where buffers need to be compared for equality, a timing-safe comparison
function provided by the host platform is used.
