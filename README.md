# Import Reflection

## Status

Champion(s): Guy Bedford, Luca Casonato

Author(s): Luca Casonato

Stage: 1

## Motivation

For both JavaScript and WebAssembly, there is a need to be able to more closely
customize the loading, linking and execution of modules beyond the standard host
execution model.

For WebAssembly, imports and exports for WebAssembly models often require custom
inspection and wrapping in order to behave correctly, which can be better
provided via manual instantiation than relying directly on the base
[ESM integration][wasm-esm] proposal.

For JavaScript, creating userland loaders would require a module reflection type
in order to share the host parsing, execution, security and caching semantics.

## Proposal

This proposal enables reflection capabilities for both JavaScript and
WebAssembly modules by enabling a new type of import, an import reflection:

```js
import x from "<specifier>" as "<reflection-type>";
```

The type of the reflection object for WebAssembly would be a
`WebAssembly.Module`, as defined in the
[WebAssembly JS integration API][wasm-js-api].

The reflection would represent an unlinked and uninstantiated module, while
still being able to support the same CSP policy as the native
[ESM integration][wasm-esm], avoiding the need for `unsafe-wasm-eval` for
custom Wasm execution.

Type type of the reflection object for a JavaScript module would be defined
explicitly in this specification as a `SourceTextModule` class with the
following interface:

```js
class SourceTextModule {
  // create a new instance of this module, with the given global environment record
  instantiate (globalEnvironmentRecord: Record<string, Binding>): ModuleInstance;
  
  // static function to retrieved the named exports, where "module" corresponds
  // to the module from which this export is reexported (if any) and "name"
  // corresponds to the exported name (or * for the case of a star re-export).
  static exports(): { module?: string, name: string | '*' }[];

  // static function to retrieve imported binding list, where "module"
  // corresponds to the module specifier being imported and "name" corresponds
  // to the exported binding from the imported module.
  static imports(): { module: string, name: string }[];
}
```

`SourceTextModule.exports(module)` and `SourceTextModule.imports(module)` can
be used to inspect the module imports and exports, analogously to
`WebAssembly.exports` and `WebAssembly.imports`.

The `instantiate` method of the `SourceTextModule` class takes a global
environment record and always returns an _unlinked_ `ModuleInstance` class
instance with the following interface:

```js
class ModuleInstance {
  // per current spec record state
  state: 'unlinked' | 'linked' | 'evaluating' | 'evaluating-async' | 'evaluated';
  // link the dependency specifiers to their instances
  link (importObj: {
    [specifier: string]: ModuleInstance | WebAssembly.Instance
  });
  // evaluate an instance
  evaluate (): void | Promise<void>;
  // access the namespace once evaluated
  exports: null | ModuleNamespaceExoticObject;
}
```

The `SourceTextModule` represents a cached module compilation, while the
`ModuleInstance` represents a particular module linkage and execution process.

* `state` trackes the `SourceTextModuleRecord` state per the current specification.
* `link` links the `ModuleInstance` dependency specifiers to `ModuleInstance`
  or `WebAssembly.Instance` modules.
* `evaluate` triggers the top-level `Evaluate()` method for the entire linked
  execution graph.

[wasm-js-api]: https://webassembly.github.io/spec/js-api/#modules
[wasm-esm]: https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration

## Custom Loader Construction

To demonstrate creating a complete custom loader using the `SourceTextModule`
JS reflection API:

```js
class Loader {
  constructor () {
    this.globalEnv = globalThis;
    this.resolve = (url, parentUrl = baseUrl) => new URL(url, parentUrl);
    this.registry = new Map();
  }

  async import (specifier) {
    const id = this.resolve(specifier);
    const entry = await this.getOrCreateEntry(id);
    await this.link(entry);
    await entry.instance.evaluate();
    return entry.instance.exports;
  }

  async getOrCreateEntry (id, url = id) {
    if (this.registry.has(id)) return this.registry.get(id);

    // Get the host module reflection
    const module = await import(url, { as: 'source-text-module' });

    // Promise for the resolution and reflection load of the dependencies (not recursive)
    const depsPromise = Promise.all(module.constructor.imports(module).map(async ({ module: specifier }) => {
      const id = this.resolve(specifier, parentUrl);
      await this.getOrCreateEntry(id);
      return [specifier, id];
    }));

    // Instantiate the host reflection
    const instance = module.instantiate(this.globalEnv);
    const entry = { instance, depsPromise };
    this.registry.set(id, entry);
    return entry;
  }

  async link (entry) {
    if (entry.instance.state !== 'unlinked') return;
    const deps = await entry.depsPromise;
    const importObj = {};
    for (const [specifier, id] of deps) {
      const entry = this.registry.get(id);
      await this.link(entry);
      importObj[specifier] = entry.instance;
    }
    entry.instance.link(importObj);
  }
}
```

The reflection API has the following properties:

* The spec invariants of the ES Module execution are fully guaranteed and cannot
  be broken. Host idempotence is maintained for instance resolution, and
  execution semantics remain fully controlled.
* The order of `link` calls does not matter at all, just as long as all modules
  are linked by the time top-level evaluate is called. Loader authors can run
  this as post-order or pre-order it does not affect the final execution as this
  is only a link-time construct. Calling link without all dependency specifiers
  being mapped to `ModuleInstance` or `WebAssembly.Instance` objects throws an
  error.
* The `SourceTextModule` instance is cached by the host in its own module loader
  so that compilation for a given resource is only ever performed once, even
  between separately constructed loaders.

## Syntax

The import reflection type is added to the end of the ImportStatement,
prefixed by the "as" keyword. If combined with an import assertion, the
assertion must always come _before_ the reflection type:

```js
import x from "<specifier>" assert {} as "<reflection-type>";
```

It is also possible to specify an import reflection if the import is not
bound:

```js
import "<specifier>" as "<reflection-type>";
```

### Re-export statements

```js
export { default as x } from "<specifier>" as "<reflection-type>";
```

For re-export statements, the syntax is essentially identical to import
statements.

### Dynamic import()

```js
const x = await import("<specifier>", { as: "<reflection-type>" });
```

For dynamic imports, import reflection is specified in the same second attribute
options bag that import assertions are specified in. The ordering of
`asserts` vs `as` does not matter here.

## Integration with other specs and environments

### WebAssembly ESM Integration

Import a WebAssembly binary as a compiled module:

##### `WebAssembly.Module` imports

As explained in the motivation, supporting a `"wasm-module"` import reflection is a driving use case for this specification in order to change the behaviour of
importing a direct compiled but unlinked [Wasm module object](https://webassembly.github.io/spec/js-api/index.html#modules):

```js
import FooModule from "./foo.wasm" as "wasm-module";
FooModule instanceof WebAssembly.Module; // true

// For example, to run a WASI execution with an API like Node.js WASI:
import { WASI } from 'wasi';
const wasi = new WASI({ args, env, preopens });

const fooInstance = await WebAssembly.instantiate(FooModule, {
  wasi_snapshot_preview1: wasi.wasiImport
});

wasi.start(fooInstance);
```

#### Imports from WebAssembly

Web Assembly would have the ability to expose import reflection in its
imports if desired, as the [Module Linking proposal currently aims to specify](https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Binary.md#import-section-updates).


#### Security improvements

[CSP][] is a web platform feature that allows you to limit the capabilities the
platform grants to a given site.

A common use is to disallow dynamic code generation means like `eval`. Wasm
compilation is unfortunatly completly dynamic right now (manual network fetch +
compile), so Wasm unconditionally requires a `script-src: unsafe-wasm-eval` CSP
attribute.

With this proposal, the Wasm module would be known statically, so would not have
to be considered as dynamic code generation. This would allow the web platform
to lift this restriction for statically imported Wasm modules, and instead just
require `script-src: self` (or possibly `wasm-src: self`) like for JS. Also see
https://github.com/WebAssembly/esm-integration/issues/56.

This property does not just impact platforms using CSP, but also other platforms
with systems to restrict permissions, such as Deno.

[CSP]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy

### HTML spec

On the web platform, there are other APIs that have the ability to load ES
modules. These also need to be able to specify import reflection. Here are
some examples of how these might look.

## Cache Key Semantics

Semantically this proposal involves a relaxation of the `HostResolveImportedModule` idempotency requirement.

The proposed approach would be a _clone_ behaviour, where imports to the same module of different
reflection types result in separate keys. These semantics do run counter to the intuition
that there is just one copy of a module.

The specification would then split the `HostResolveImportedModule` hook into two components -
module asset resolution, and module asset interpretation. The module asset resolution component
would retain the exact same idempotency requirement, while the module asset interpretation component
would have idempotency down to keying on the module asset and reflection type pair.

Effectively, this splits the module cache into two separate caches - an asset cache retaining the
current idempotency of the `HostResolveImportedModule` host hook, pointing to an opaque cached asset reference,
and a module instance cache, keyed by this opaque asset reference and the reflection type.

Alternative proposals include:

- **Race** and use the attribute that was requested by the first import. This
  seems broken in that the second usage is ignored.
- **Reject** the module graph and don't load if attributes differ. This seems
  bad for composition--using two unrelated packages together could break, if
  they load the same module with disagreeing attributes.

Both of these alternatives seem less versatile than the proposed _clone_ behaviour above.

## Q&A

**Q**: How does this relate to import assertions?

**A**: Import assertions do not influence how an imported asset is evaluated, and they
do not influence the HostResolveImportedModule idempotency requirements. This
proposal does. Also see
https://github.com/tc39/proposal-import-assertions#follow-up-proposal-evaluator-attributes.

**Q**: Are there use cases for this feature other than JS & Web Assembly imports?

**A**: There may be scope to handle other types of module reflections or asset reflections,
but the scope would be restricted only to reflections of the existing known type, as opposed
to new type interpretations. E.g. hosts enabling custom source language support would not be a
possibility of this proposal.

**Q**: Why not just use `const module = await WebAssembly.compileStreaming(fetch(new URL("./module.wasm", import.meta.url)));`?

**A**: There are multiple benefits: firstly if the module is statically referenced in the module
graph, it is easier to statically analyze (by bundlers for example). Secondly when using CSP,
`script-src: unsafe-eval` would not be needed. See the security improvements section for more
details.
