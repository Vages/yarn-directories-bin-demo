# yarn-directories-bin-demo

- [Bug description](#bug-description)
- [Workarounds](#workarounds)
  - [Use Yarn \<3.6.2](#use-yarn-362)
  - [Remove directories `bin` from `yarn.lock`](#remove-directories-bin-from-yarnlock)
  - [Change shebang in `script.cjs`](#change-shebang-in-scriptcjs)
  - [Uninstalling the `jsbarcode` dependency](#uninstalling-the-jsbarcode-dependency)
- [Root cause](#root-cause)

## Bug description

Ideally, you should see the output `foofoo` when running both of the following
commands in the root directory. Only the first command works.

```shell
./script.cjs # output: foofoo
yarn bar # fails
```

The second commands fails with output that looks like this (with the exception
of paths).

```text
$ yarn bar
Internal Error: EISDIR: illegal operation on a directory, read
    at makeError$1 (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:538:24)
    at EISDIR (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:559:10)
    at ZipFS.readFileBuffer (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:2559:13)
    at ZipFS.readFileSync (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:2547:23)
    at ZipFS.readSync (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:1822:25)
    at ZipOpenFS.readSync (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:3136:18)
    at VirtualFS.readSync (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:2696:24)
    at PosixFS.readSync (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:2696:24)
    at NodePathFS.readSync (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:2696:24)
    at Object.readSync (/redacted-path/yarn-directories-bin-demo/.pnp.cjs:4287:21)
```

## Workarounds

You can get `yarn bar` to run in one of the following ways.

### Use Yarn <3.6.2

Both scripts work if you use Yarn `v3.6.1`.

```shell
yarn set version 3.6.1
./script.cjs # output: foofoo
yarn bar # output: foofoo
```

We have also tested `v3.6.0` and `v3.5.1` and found that they fix the problem.

### Remove directories `bin` from `yarn.lock`

The patch `remove-directories-from-yarn-lock.patch` removes all entries that
reference directories from `bin` keys in `yarn.lock`. (Editing a `yarn.lock`
file is not a permanent solution, as Yarn may overwrite your changes later.)

```shell
git apply remove-directories-from-yarn-lock.patch
```

### Change shebang in `script.cjs`

If you replace the shebang in `script.cjs` with `node` (instead of `yarn node`),
running `yarn bar` will work.

```shell
sed -i '' 's#yarn node#node#g' script.cjs
yarn bar # output: foofoo
```

Running the script directly leads to the following error, as `node` cannot
resolve PnP modules without Yarn.

```text
$ ./script.cjs
node:internal/modules/cjs/loader:1080
  throw err;
  ^

Error: Cannot find module 'lodash'
Require stack:
- /redacted-path/yarn-directories-bin-demo/script.cjs
    at Module._resolveFilename (node:internal/modules/cjs/loader:1077:15)
    at Module._load (node:internal/modules/cjs/loader:922:27)
    at Module.require (node:internal/modules/cjs/loader:1143:19)
    at require (node:internal/modules/cjs/helpers:119:18)
    at Object.<anonymous> (/redacted-path/yarn-directories-bin-demo/script.cjs:2:11)
    at Module._compile (node:internal/modules/cjs/loader:1256:14)
    at Module._extensions..js (node:internal/modules/cjs/loader:1310:10)
    at Module.load (node:internal/modules/cjs/loader:1119:32)
    at Module._load (node:internal/modules/cjs/loader:960:12)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:86:12) {
  code: 'MODULE_NOT_FOUND',
  requireStack: [
    '/redacted-path/yarn-directories-bin-demo/script.cjs'
  ]
}

Node.js v18.18.0
```

### Uninstalling the `jsbarcode` dependency

Removing the `jsbarcode` dependency will fix the problem entirely. In projects
that need this dependency, this is undesirable.

## Root cause

For a given dependency, Yarn adds every file and directory listed under
`directories.bin` in its `package.json` to the dependency's `bin` entry in
`yarn.lock`. This is probably against the field's intention, as NPMs
documentation on `directories.bin` only references _files_:

> If you specify a bin directory in directories.bin, all the files in that
> folder will be added. –
> https://docs.npmjs.com/cli/v10/configuring-npm/package-json?v=true#directoriesbin

In the case of `jsbarcode`, the relevant part of its `directories` entry is as
follows:

```json
// jsbarcode/package.json
{
  // (other content omitted for brevity)
  "directories": {
    "bin": "bin"
  }
}
```

The folder's tree looks as follows:

```
bin
├── JsBarcode.js
├── barcodes
│   ├── Barcode.js
│   ├── CODE128
│   │   ├── CODE128.js
│   │   ├── CODE128A.js
│   │   ├── CODE128B.js
│   │   ├── CODE128C.js
│   │   ├── CODE128_AUTO.js
│   │   ├── auto.js
│   │   ├── constants.js
│   │   └── index.js
│   ├── CODE39
│   │   └── index.js
│   ├── EAN_UPC
│   │   ├── EAN.js
│   │   ├── EAN13.js
│   │   ├── EAN2.js
│   │   ├── EAN5.js
│   │   ├── EAN8.js
│   │   ├── UPC.js
│   │   ├── UPCE.js
│   │   ├── constants.js
│   │   ├── encoder.js
│   │   └── index.js
│   ├── GenericBarcode
│   │   └── index.js
│   ├── ITF
│   │   ├── ITF.js
│   │   ├── ITF14.js
│   │   ├── constants.js
│   │   └── index.js
│   ├── MSI
│   │   ├── MSI.js
│   │   ├── MSI10.js
│   │   ├── MSI1010.js
│   │   ├── MSI11.js
│   │   ├── MSI1110.js
│   │   ├── checksums.js
│   │   └── index.js
│   ├── codabar
│   │   └── index.js
│   ├── index.js
│   ├── index.tmp.js
│   └── pharmacode
│       └── index.js
├── exceptions
│   ├── ErrorHandler.js
│   └── exceptions.js
├── help
│   ├── fixOptions.js
│   ├── getOptionsFromElement.js
│   ├── getRenderProperties.js
│   ├── linearizeEncodings.js
│   ├── merge.js
│   └── optionsFromStrings.js
├── options
│   └── defaults.js
└── renderers
    ├── canvas.js
    ├── index.js
    ├── object.js
    ├── shared.js
    └── svg.js

14 directories, 51 files
```
