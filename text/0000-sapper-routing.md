- Start Date: 2020-09-16
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Refactoring out Sapper's Routing

## Summary

Split Sapper's routing functionality into a standalone library and make it possible to use Sapper's routing outside of Sapper.

## Motivation

Benefits include:

* Making it possible to use Sapper's routing outside of Sapper. The routing component should be able to be used in a client-side only application
* Making Sapper's routing more configurable. E.g. the user may wish to set a custom header on a certain route. We could expose the router component in the application and allow the user to set configuration on a per-route basis along the lines of `router.header('/[list]/[page]', {'Cache-Control': max-age=600'})`
    * Other requests users have for configuring the router include [routes aliases](https://github.com/sveltejs/sapper/issues/1450) and [configuring trailing slashes](https://github.com/sveltejs/sapper/issues/519), which we could expose APIs for as well
* Improved testability. Right now to test routing functionality you need a sample application and all tests are integration tests
* I believe it would become possible to make it such that Sapper's routing can be used with or without generating it from the file system though this is perhaps not a primary goal.
* Finally, Sapper's routing and Routify are quite similar. It may be possible to combine these two systems if everyone involved were open to such a possibility. I think it'd be good for the Svelte community to have a single solution because it means that solution would get more developer support than two individual solutions. Right now [there are many Svelte routing solutions](https://twitter.com/lihautan/status/1315482668440580096?s=19).

## Detailed design

### Implementation

This could be split up into multiple PRs that are smaller and easier to review.

#### Step 1

Remove need to specify CSS files in routing components.

Update: this has been completed in [#1508](https://github.com/sveltejs/sapper/pull/1508)


#### Step 2

Put all routing information in a single routing component. Historically it has been split between `start` and `app`.

Update: this has been largely completed in [#1434](https://github.com/sveltejs/sapper/pull/1434). We can probably still clean up the prefetching code, etc.


#### Step 3

Add an API to register routes and make the generated code call the router rather than the router calling the generated code.

Right now, the generated `manifest-client.mjs` contains something like:

```
export const components = [
	{
		js: () => import("../../../routes/index.svelte")
	},
	{
		js: () => import("../../../routes/[list]/[page].svelte")
	}
];

export const routes = (d => [
	{
		// index.svelte
		pattern: /^\/$/,
		parts: [
			{ i: 0 }
		]
	},
	{
		// [list]/[page].svelte
		pattern: /^\/([^/]+?)\/([^/]+?)\/?$/,
		parts: [
			null,
			{ i: 1, params: match => ({ list: d(match[1]), page: d(match[2]) }) }
		]
	}
])(decodeURIComponent);
```


It would be nice to invert this such that the generated code calls the router instead of the router importing the generated code:

```
import router from 'router';

router.register({
  route: /^\/$/,
  component: () => import("../../../routes/index.svelte")
});
router.register({
  route: /^\/([^/]+?)\/([^/]+?)\/?$/,
  params: match => ({ list: d(match[1]), page: d(match[2]) }),
  component: () => import("../../../routes/[list]/[page].svelte")
});
```

This API is a bit complex. However, it could be greatly simplified by making the route parsing be part of the router runtime instead of happening at compile time:

```
import router from 'router';

router.register('/', () => import("../../../routes/index.svelte"));
router.register('/[list]/[page]', () => import("../../../routes/[list]/[page].svelte"));
```

#### Step 4

Unify the client and server routing APIs.

The routes today get generated into `manifest-client.mjs` and `manifest-server.mjs` in the `src/node_modules/@sapper/internal` directory. Seperate manifests are needed because Svelte generates different components for SSR and the DOM. Additionally, the server-side routing manifest contains a list of server routes, which the client manifest does not.

The server manifest also differs in a number of unnecessary ways:
* it provide directly imported components while the client-side does dynamic imports. This could easily be changed so that both client and server use dynamic imports
* it contains a `name` field used only for webpack support. This field is a result of webpack unnecessarily using a different format in `build.json`
* it contains a `file` field to lookup `preload` header dependencies from `build.json`. We should be able key off the route instead of the file name in `build.json`

We can remove the unnecessary differences. We will still need different client-side and server-side router instantiations registering their respective components, but they can utilize the same `router.register` API. On the client-side we can call `router.navigate`. On the server-side, we can just ask the router to return the registered component for a given route and let `get_page_handler` continue to handle it as it does today.

#### Step 5

Refactor out routes generation into separate plugin. Publish components separately.

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
