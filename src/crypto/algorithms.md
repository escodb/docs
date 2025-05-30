# Algorithms used

EscoDB makes use of the following cryptographic primitives and constructions:

- AES-256-GCM
- SHA-256
- HMAC
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
