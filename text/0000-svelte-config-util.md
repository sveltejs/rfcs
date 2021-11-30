- Start Date: (fill me in with today's date, 2021-09-28)
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Svelte config utility

## Summary

Introduce a new utility package `@sveltejs/config` to offer tools in the svelte ecosystem a common way
to read and validate svelte config files and provide config features to users in a consistent manner.

## Motivation

Currently, tools implement config reading separately (or not at all),
leading to extra maintenance and more work for new tools who need to read it.

Reducing this overhead and offering a place to enforce consistency
and implement new features is the main motivation of this RFC,
resulting in a better dx for both users writing svelte config files
and tool authors consuming them via a simple common api.

### Features

#### for tools
* async `loadConfig` function
* defined namespacing (top-level keys owned by a tool)
* validation utilities

#### for users
* `defineConfig` helper for intellisense like in [vite config](https://vitejs.dev/config/#config-intellisense)
* `async` config support like in [vite config](https://vitejs.dev/config/#async-config)
* allow `svelte.config.ts`


## Detailed design

### Technical Background

### Prior Art
* [postcss-load-config](https://github.com/postcss/postcss-load-config)
* [sveltekit](https://github.com/sveltejs/kit/tree/master/packages/kit/src/core/config)
* [vite-plugin-svelte](https://github.com/sveltejs/vite-plugin-svelte/blob/main/packages/vite-plugin-svelte/src/utils/load-svelte-config.ts)

### Implementation

The implementation is going to be fully compatible to existing usage of svelte.config.js (cjs/mjs) files.

Tools can opt in by adopting `@sveltejs/config`

New config features for users could potentially introduce changes that are not backwards compatible
to tools reading svelte config without `@sveltejs/config`.
To reduce friction, v1.0 of @sveltejs/config could be limited to features that are explicitly backwards compatible
and features that would break will only be released after our tools adopted it

#### New Features for tool authors

##### async `loadConfig` function
```js
import { loadConfig } from '@sveltejs/config';
//...
const config = await load_config();
```

loadConfig accepts an optional `options` argument with the following options:
```js
options = {
    fromDir: '/some/path', // process.cwd by default, tools can pass what they need, eg vite root
    inlineConfig: {/*...*/}, // svelte config object built from inline options eg cli args passed to the tool. merged in
    namespace: 'kit', // namespace for the tool loading the config, useful for advanced features
    //...
}
```

##### defined namespacing (top-level keys owned by a tool)

Currently, namespaces in svelte.config.js are loosely agreed upon. 
`compilerOptions`, `preprocess`, `extensions` and `kit` are used by SvelteKit.

Each tool will get its own namespace, e.g. `vitePlugin`, `rollupPlugin`, `webpackLoader`, `languageTools`.

Community provided tools will be allowed to use their own namespaces (e.g. `elder` or `routify`),
but these are not supported beyond being tolerated by `@sveltejs/config`

##### validation utilities

SvelteKit contains [code](https://github.com/sveltejs/kit/blob/master/packages/kit/src/core/config/options.js) to validate config options and output helpful messages. 
Refactor this code to be of general use and provide an api

```js
import { loadConfig ,validateConfig } from '@sveltejs/config';
//...
const config = await load_config();
const rules = {
    //TBD, use sveltekit options as a base
}
// throws on error
validateConfig(config, rules);
```

#### New Features for users

##### `defineConfig` helper for intellisense like in [vite config](https://vitejs.dev/config/#config-intellisense)
```js
import { defineConfig } from '@sveltejs/config'

export default defineConfig({
// ...
})
```

##### `async` config support like in [vite config](https://vitejs.dev/config/#async-config)
```js
export default defineConfig(async () => {
    const data = await asyncFunction()
    return {
        // build specific config
    }
})
```

##### allow `svelte.config.ts` and transpile automatically

Transpilation is done via lazy dynamic import of `typescript` or `esbuild`. If neither is available, this fails.
It allows users of typescript projects to fully embrace ts without burdening js projects with a typescript dependency.


## How we teach this

### To tool authors

README.md and/or docs/ in the new GitHub repo, example usage in kit and vite-plugin-svelte

### To users

Currently, svelte.config.js isn't really documented on its own. The tools show how they use it.
Add user documentation to the GitHub repo too and link to it from svelte.dev docs.
Alternatively: Add a new section to the svelte.dev docs directly, this would incur extra work for new features though


## Drawbacks

Once a tool adopts this, it is bound to the features offered. 
If new requirements come up it would have to be added in `@sveltejs/config`
and create additional overhead for coordinating releases

## Alternatives

### use existing 3rd party tools
[lilconfig](https://github.com/antonk52/lilconfig) is a smaller alternative to cosmiconfig for loading,
object validation libraries exist too.

### keep doing it separately in each tool

It works as it is now, maybe the pain isn't worth the hassle.
Just add some documentation and raise awareness of potential issues with different implementations

## Unresolved questions

### exact api for validation

How to refactor kit validation to be of general use

### caching

How could we cache the config so that multiple tools being used together can share the same load