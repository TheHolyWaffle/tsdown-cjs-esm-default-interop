# tsdown CJS default-interop repro (ESM-only external dep)

Minimal reproduction: in a `"type": "module"` package, tsdown's CommonJS output
default-imports an external ESM-only dependency via `__toESM(require(dep), 1)`
(`isNodeMode = 1`). Under Node's `require(esm)`, that re-nests the dependency's
default export under an extra `.default`, so `dep.default.<member>` is `undefined`
at runtime.

Uses `chalk@5` (ESM-only, default export) as the external dependency.

## Steps

```sh
npm install
npm run build      # tsdown — succeeds, exit 0

npm run repro      # node -e "require('./dist/index.cjs')"
# -> TypeError: chalk.default.red is not a function
```

The ESM build is fine:

```sh
node --input-type=module -e "import('./dist/index.mjs').then(m => console.log(m.message))"
# -> hello (styled)
```

## Output

`dist/index.cjs`:

```js
let chalk = require("chalk");
chalk = __toESM(chalk, 1);              // isNodeMode = 1
const message = chalk.default.red("hello");   // chalk.default is the namespace, not the fn
exports.message = message;
```

Node's `require("chalk")` already returns a module namespace with a correct
`.default` and `__esModule: true`. The `__toESM(..., 1)` helper's `isNodeMode = 1`
argument unconditionally sets `.default` to the whole namespace, overwriting the real
default, so `chalk.default.red` is `undefined`.

## Trigger — the package's `"type"` field

The bug appears only when the built package is `"type": "module"`. Set
`"type": "commonjs"` in `package.json` and rebuild: tsdown then emits
`chalk = __toESM(chalk)` (no `isNodeMode`), which respects `__esModule` and works.

| `package.json` `"type"` | emitted interop | `require()` |
| --- | --- | --- |
| `"module"` | `__toESM(chalk, 1)` | TypeError |
| `"commonjs"` | `__toESM(chalk)` | works |

## Not a rolldown bug

Driving rolldown directly with `external: ['chalk']` and `format: 'cjs'` emits the
correct `__toESM(chalk)` (no `isNodeMode`) and loads fine — rolldown has no notion of
the consumer's package `type`. The `isNodeMode = 1` is introduced by tsdown for a
`.cjs` emitted inside a `"type": "module"` package.

## Environment

```
tsdown    0.22.3
rolldown  1.1.4
node      24.16.0
```
