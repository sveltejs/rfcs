- Start Date: 2021-05-29
- RFC PR: 
- Svelte Issue: 

# Abstract pre-render

## Summary

Adding support for prerendered HTML files which look for their page store data at client-side runtime (what I'd call "abstract"), instead of using the snapshot of such data as it was at build time (what I'd call "concrete").

## Motivation

This will allow us to pre-render every page, even the ones that do something with the request-specific page store data in `onMount`. It is possible to do this today with no changes to SvelteKit itself, if you have a web server that can read and transform (interpolating values) into the html artifacts for each request. But doing that is messy.

There's a lot more detail about motivation in the initial comment of https://github.com/sveltejs/kit/issues/1533.

## Detailed design

### Technical Background

Right now, a pre-rendered page has code like this to hydrate (this is for a parameterized route):

    page: {
      host: location.host, // TODO this is redundant
      path: "/my/page/123",
      query: new URLSearchParams("foo=bar"),
      params: {"id": "123"}
    }

Aside from the "host", those are all the concrete, specific values as they were at build time, baked into the page as literals. But one might want to use the prerendered HTML artifact in an abstract way, for variable and wildcard request parameters, in which case you'd want something more like this:

    page: {
      host: location.host,
      path: location.pathname,
      query: new URLSearchParams(location.search),
      params: parsePathParams(location.pathname, /^\/my\/page\/(\w)+$/, ["id"]),
    }

Where `parsePathParams` is a simple function that gives you an object of path params.

### Implementation

The current state of the code is here:

https://github.com/sveltejs/kit/blob/a66affec4168fae053fb6349805b60028306abcf/packages/kit/src/runtime/server/page/render.js#L136

The changes would be:

* For host, remove the TODO comment and the ternary, and just always use `location.host`
* For path, use `location.pathname`
* For query, use `location.search`
* Params is the hardest one, we'd need to insert the route regex and the positional list of parameter names into a simple `parsePathParams` function, possibly just inlined as an arrow function
* Update the docs where appropriate

Note that there is an in progress PR for the query part in https://github.com/sveltejs/kit/pull/1511 but in my view it would be better to address these all together in one PR.

If we implemented this RFC, it fixes https://github.com/sveltejs/kit/issues/669 and https://github.com/sveltejs/kit/issues/1397, and partially fixes https://github.com/sveltejs/kit/issues/1561 and https://github.com/sveltejs/kit/issues/1533. This RFC is only about the page store hydration aspect but "Abstract Pre-rendering" has some other facets to it, discussed in https://github.com/sveltejs/kit/issues/1533, including the question of what values you provide for parameterized route params when adding paths to `kit.prerender.pages`.

The other big open question is -- do we need a switch between "concrete" (current behavior) and "abstract" (proposed behavior) mode, perhaps in `kit.prerender`, or would be fine to just always do abstract.

## How we teach this

I don't know if "abstract" and "concrete" are the best terminology, could also be called "deferred evaluation", or "dynamic page store vs fixed page store," etc.

Also there was some confusion about the term "parameterized route" before, just to clarify, I'm using "parameterized" to mean freely variable and unknown at build time.

Acceptance would entail some update to the guides, where it suggests what kind of pages are and are not suitable for pre-rendering. If we had abstract pre-rendering, _any_ page would be suitable for pre-rendering as long as it puts all its request-specific logic in `onMount`. See https://github.com/sveltejs/kit/issues/1397

## Drawbacks

I don't see any drawbacks.

## Alternatives

One alterative, or maybe you'd call it a workaround, is to add a layer of server-side runtime interpolation that fills in the page store for each request. This involves some ugly regex slice and dice though. I have an example of this approach, in Python, in https://github.com/johnnysprinkles/static-all-the-things/blob/22e3e98149876364d8a4317e9557f727e2287021/web/manifest_load.py#L26

Another alternative would be to just not use the page store and directly inspect the `window.location` object. This would work for the query parameters but for route parameters it would need a duplicate of the route parsing logic, including the names of the parameters, and seems like a pretty untidy solution.

## Unresolved questions

Just the two questions mentioned above:

* It's unclear what you provide for parameterized path params when pre-rendering an abstract page. It could be anything, an "x", and underscore, the only reason it matters is it determines where the html artifact lives in the build folder, and the web server will have to find the file. This question is probably out of scope for this RFC, but wanted to mention it as a related question.
* Do we need a switch between abstract and concrete mode, or can we always do abstract?
