- Start Date: 2020-08-30
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Multiple `context=module` Scripts

## Summary

Allow for multiple `script[context=module]` blocks to exist, targetting different output/generate modes.

In general, this will (probably) be used _primarily_ by Sapper (and its relatives) but, the ability must be implemented by Svelte. More importantly, this makes sense for Svelte in its own right, too, since a flagship feature of Svelte is that it can compile/generate its own `ssr`-vs-`dom` outputs.

## Motivation

In many – but not most – situations, it makes sense to export different module methods depending on _how_ a component is compiled. The most popular example is `preload` and so I'll use that as [an example](#example), but this RFC is most definitely not limited to nor specifically targetting `preload` alone.

## Detailed Design

By default, `context=module` will act the same as it does now. This is because there's no reason for it to be any different - which also means that this would be a non-breaking change/addition.

While syntax is still WIP and up for debate, I'm thinking something like these will work well:

#### Option 1

```html
<script context="module">
    <!-- shared with "dom" and "ssr" -->
</script>

<script context="module:dom">
    <!-- added to "dom" output only -->
</script>

<script context="module:ssr">
    <!-- added to "ssr" output only -->
</script>
```

#### Option 2

> **Note:** I am avoiding `type=` intentionally to avoid IDE/tooling issues.

```html
<script context="module">
    <!-- shared with "dom" and "ssr" -->
</script>

<script context="module" output="dom">
    <!-- added to "dom" output only -->
</script>

<script context="module" output="ssr">
    <!-- added to "ssr" output only -->
</script>
```

I lean towards Option 1.


## Example

> The thought experiment may be more familiar under the guise of Sapper, but it's not **for** Sapper specifically.

Let's assume you have an `Article` component. It's self-reliant, meaning it fetches its own data (based on a `slug` prop, perhaps) and renders the article body with the given response:

```html
<script lang="ts">
    let post: Article = {};

    export let slug: string;
</script>

<h1>{post.title}</h1>

<div class="content">
	{@html post.body}
</div>
```

Generally, something like this warrants a `preload` function inside a `context=module` block, and then a runtime (Sapper, or otherwise) will call this `preload` function so that the `Article` instance has its data handy at time of render. We'll use `fetch` for this:

```html
<script context="module" lang="ts">
    let loaded = {};

    export function preload(req: IRequest) {
        loaded.slug = req.params.slug;
        return fetch(`.../${loaded.slug}`).then(r => r.json()).then(obj => {
            loaded.data = obj;
        });
	}
</script>

<script lang="ts">
    let post: IArticle = loaded.data || {};
    export let slug: string = loaded.slug || '';
</script>

<h1>{post.title}</h1>

<div class="content">
	{@html post.body}
</div>
```

With this example, we have the same `preload` function that works in `dom` and `ssr` compilations, assuming that our server's runtime has an appropriate `fetch` polyfill.

However, what if you don't want to use `fetch` - or any network requestor - for server preloading? What if your DOM and SSR environments warrant different behaviors entirely?

For example, if you have a single Node.js cluster, which is responsible for SSR, the public API, _and_ acts as a database client, you probably want to just hook into the database directly since it's there. Well, how are you supposed to do that without killing the `dom` output? How much will you have to "work around" Svelte just to inject component data?

> **Hint:** You have to add at least one `set_post`-like function to `context=module` exports so that you can call that directly with your external preloader. Not to mention that you still have to polyfill `fetch` in this case _and_ have to remember/implement a data-layer for one thing in two different locations...

Instead, I am proposing multiple, output/environment-specific `context=module` blocks.
This allows the developer of this example to be explicit with the behavioral distinctions while keeping them in the same place. And since the entire `script` block is contained, each output can bring with it different  dependencies without ever risking that server-only dependencies make it into the browser and vice versa.

```html
<script context="module:dom" lang="ts">
    // Still using fetch() for DOM

    let loaded = {};

    export function preload(req: IRequest) {
        loaded.slug = req.params.slug;
        return fetch(`.../${loaded.slug}`).then(r => r.json()).then(obj => {
            loaded.data = obj;
        });
	}
</script>

<script context="module:ssr" lang="ts">
    // Bring database dependencies for server
    import sql from 'postgres';
    import type { IncomingMessage } from 'http';

    let loaded = {};

    // Maintain my own preload() contract
    export async function preload(req: IncomingMessage) {
        loaded.slug = req.params.slug;

        const rows = await sql<IArticle>`
            select * from articles 
            where slug = ${loaded.slug}
            and deleted_at is null
            limt 1
        `;

        if (rows.length) loaded.data = rows[0];
	}
</script>

<script lang="ts">
    let post: IArticle = loaded.data || {};
    export let slug: string = loaded.slug || '';
</script>

<h1>{post.title}</h1>

<div class="content">
	{@html post.body}
</div>
```


## How we teach this

> TBH I think our current documentation around how `context=module` exports are accessed could use more detail and/or more highlighting. I find that users who side-step Sapper are still unaware how/where to find its contents.

I think it's really as simple as adding the two variants to the `context=module` docs:

1) A bare `context=module` is shared across `dom` and `ssr` output
2) A namespaced `context=module` is ***only*** included in its target output

```js
let module = {
    default: Component,
    ...context // context=module
};

if (has_context('dom') && options.generate === 'dom') {
    module = { ...module, ...context_dom };
} else if (has_context('ssr') && options.generate === 'ssr') {
    module = { ...module, ...context_ssr };
}
```

## Drawbacks

* Additional work in `svelte.parse` is needed.
* Users may be confused if `how we teach this` fails/sub-par

## Alternatives

None. Introduced in a maintainers meeting.
@Rich-Harris is interested in building upon this and taking it further in the Sapper arena, but this is a necessary building block and first step.

## Unresolved questions

Syntax/naming – see `Option 1` vs `Option 2` above.