<p align="center">
  <img src="/assets/1050684.jpg" />
</p>
<p align="center">
  <strong>Documentation files for EDOPro's Card Scripting API</strong>
</p>

Entries are in yaml format, found under [`/docs`](/docs/).

## Parsing pipeline
1. Read all the `.yml` files from the supplied directory recursively. Subfolders do not matter and are only for developer organization.
   
2. Feed the yaml files into [`js-yaml`](https://github.com/nodeca/js-yaml).

   2.1. Check if the file is a properly formatted yaml file.

   2.2. Check if the top-level definition is a "mapping" (which translates to a JS object).

   2.3. Assign the yaml tag supplied at the top of the file as the JS object's `doctype` field.

   2.4. Check if the JS object has a recognized doctype (function, namespace, constant, enum, type, or tag).

   2.5. Assign the file path to the result's `filepath` field.

   2.5. If any of the files fail a check, stop the pipeline here, giving an error value that describes the invalid files.

   2.6. There's now an array of plain JS objects with a `doctype` and `filepath`.

3. Group all entries according to their `doctype`.

4. Decode each group using the "codecs" implemented in [`io-ts`](https://gcanti.github.io/io-ts/).
   These are fine-grained rules that each entry must follow, and are specific to each doctype.

   4.1. `io-ts` also narrows down the type as it checks each rule, going from plain JS objects into a stricter interface.
   
   4.2. Entries may also be transformed a bit while decoding, if needed.
   For example, the `name` field of functions become `namespace + "." + name`, and a `partialName` field is added.
   Markdown strings also parsed into a markdown AST.

   4.3. If any of the entries or groups fail their respective codec, stop the pipeline here.
   Generate an error report that describes the invalid entries.

   4.4. All entries are now guaranteed to be correct and have a strict typing.

5. Finalize the entries, further transforming them to have better types and additional information.
   
   5.1. Some entries need a `SourceRecord` to get their source code link.
   This is generated by fetching the source files from the core and CardScripts repo,
   and parsing lua definitions in them.

Note that finalization did **NOT** need to be a separate step.
It could have been integrated in the codecs, like the transformations in `Step 4.2`.
However, it's pointless to finalize an entry when there's at least one other invalid file that will cause the pipeline to fail anyway.
As such, only transformations that are *necessary for checking* are included in `Step 4.2`.

### Dumping validated raws in JSON format
For quick local validation, you can run a dump script that essentially performs Steps 1 - 4 in the pipeline. 
This generates a JSON file with the resulting entries, or an error file.
It does not run Step 5, so the entries are not finalized, although guaranteed to be correct.

- Clone the repo and navigate to it.
- `npm i`
- `npm run dump`
- Check the `dump/` folder for the output (or error).

***

## As a Typescript Package
The repository is also a typescript package for the default loader module that provides typings and validates entries as strictly as possible.
This can be installed with npm.

```
npm i @that-hatter/scrapiyard
```

This package is used as a dependency in all projects under `scrapi`.

### Usage

The most relevant function is `loadYard`. It is recommended to use [`fp-ts`](https://gcanti.github.io/fp-ts/) for the least amount of friction.

```ts
import { pipe } from 'fp-ts/function';
import * as TE from 'fp-ts/TaskEither';
import * as sy from '@that-hatter/scrapiyard';

const program = pipe(
  sy.loadYard(sy.DEFAULT_OPTIONS),
  TE.map(({ api }) => {
    // ...
  })
);
```

There are also more granular functions that perform specific steps, such as `loadAPI`, `loadRawAPI`, and `loadSourceRecord`.
See [`dump.ts`](/src/dump.ts) for an example usage.

If not using `fp-ts`, you will need to call the value returned by `loadYard` to get a `Promise`, and manually check if its `_tag` is `"Right"`.

```ts
const yard = await sy.loadYard(sy.DEFAULT_OPTIONS)();
if (yard._tag === 'Right') {
  const { api } = yard.right;
  // ...
}
```

***

**Inspired, one way or another, by the following projects**
- [Factorio runtime API documentaion](https://lua-api.factorio.com/latest/index-runtime.html)
- [LÖVE wiki](https://love2d.org/wiki/Main_Page)
- [Binding of Isaac modding API documentation](https://wofsauge.github.io/IsaacDocs/rep/)
- [moonwave](https://github.com/evaera/moonwave)
- [sumneko's lua lsp server annotations](https://github.com/LuaLS/lua-language-server/wiki/Annotations)
