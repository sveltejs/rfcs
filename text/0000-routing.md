- Start Date: 2020-09-16
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Refactoring out SvelteKit's Routing

## Summary

Split SvelteKit's routing functionality into a standalone library and make it possible to use SvelteKit's routing outside of SvelteKit.

## Motivation

Benefits include:

* Making it possible to use SvelteKit's routing outside of SvelteKit. The routing component should be able to be used in a client-side only application
* Making SvelteKit's routing more configurable
    * User requests for configuring the router include [routes aliases](https://github.com/sveltejs/sapper/issues/1450) and [configuring trailing slashes](https://github.com/sveltejs/sapper/issues/519), which we could expose APIs for
* Improved testability. Right now to test routing functionality you need a sample application and all tests are integration tests
* I believe it would become possible to make it such that SvelteKit's routing can be used with or without generating it from the file system though this is perhaps not a primary goal.
* Finally, SvelteKit's routing and Routify are quite similar. It may be possible to combine these two systems if everyone involved were open to such a possibility. I think it'd be good for the Svelte community to have a single solution because it means that solution would get more developer support than two individual solutions. Right now [there are many Svelte routing solutions](https://twitter.com/lihautan/status/1315482668440580096?s=19).

## Detailed design

### Implementation

This could be split up into multiple PRs that are smaller and easier to review.

#### Step 1

Remove need to specify CSS files in routing components.

Update: this was completed in [#1508](https://github.com/sveltejs/sapper/pull/1508).


#### Step 2

Put all routing information in a single routing component. Historically, in Sapper, it has been split between `start` and `app`.

Update: this has basically been completed with the router now living `packages/kit/src/runtime/client`


#### Step 3

Create a routing API.

Parse routes at runtime to simplify API. Update: this is now done and lives in `packages/kit/src/runtime/client/parse.js`

Right now, the generated `./svelte-kit/generated/client-manifest.js` contains something like:

```
export { matchers } from './client-matchers.js';

export const components = [
	() => import("../../src/routes/__layout.svelte"),
	() => import("../runtime/components/error.svelte"),
	() => import("../../src/routes/[slug].svelte"),
	() => import("../../src/routes/example/[param].svelte"),
	() => import("../../src/routes/index.svelte")
];

export const dictionary = {
	"": [[0, 4], [1]],
	"example/[param]": [[0, 3], [1]],
	"[slug]": [[0, 2], [1]]
};
```

This API is pretty compact, but not too human readable. It also requires the layouts and error components to be specified repeatedly. We could create a friendly API for this:
```
const routes = new Route('/')
	.layout(() => import("../../src/routes/__layout.svelte"))
	.error(() => import("../runtime/components/error.svelte"))
	.routes({
		'': () => import('../../src/routes/index.svelte'),
		'[slug]': () => import('../../src/routes/[slug].svelte'),
		'example/[param]': () => import(../../src/routes/example/[param].svelte);
	});
```

#### Step 4

Refactor out routes generation into separate Vite plugin like `vite-plugin-svelte-kit:router`. Publish components separately.

Review the API and make sure we like it. There's probably some minor cleanup to do before exposing the routing API more broadly

We could potentially have a library for the core routing functionality. And then a second for the file-system-based routes generator.


## How we teach this

Update the documentation, migration guide, and templates.

## Drawbacks

Some user migration may be necessary. E.g. if the router is a separate component, then it will need to be included in `package.json` as a dependency.

The bundle size may increase slightly if the routes parsing is included in the bundle. This probably would result in about 50 lines of additional code in the router runtime. However, it would reduce the number of lines required to register a route so it may end up being a savings for larger applications. If we are concerned about this it could be made an optional pluggable component of the router. Most users will not register components at runtime and so using generated regexes will continue to be just fine for the existing use cases. Though where it is likely to be very helpful is if we want to allow programmatic access to configuring the routes (e.g. to add a header)

## Alternatives

Probably the main alternative would be the status quo. If people want to use Sapper's routing without the rest of Sapper then we point them to Routify, page.js, or some other solution.

## Unresolved questions

The last step seems the one that we would need to put the most time and thought into. E.g. should it live in a Rollup plugin, would we continue to support webpack, are there reason we don't want to make everything into plugins?

When publishing new npm packages should we put them in a `@sapper` namespace? Can/should we publish multiple packages from the existing Saper repo or should we make a new repo for each package?
