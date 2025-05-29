# Installation

EscoDB does not have a central server but rather operates as a library that you
embed in your own applications. It is distributed as a JavaScript package
installable via `npm`:

```sh
$ npm install @escodb/core
```

Once installed, it can be loaded via `require()` or `import`:

```js
// CommonJS
const { Store, FileAdapter } = require('@escodb/core')

// ESM
import { Store, FileAdapter } from '@escodb/core'
```
