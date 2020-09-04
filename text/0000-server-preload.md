
- Start Date: 2020-09-03
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Server preload

## Summary

Allow Sapper page components to contain the server-side logic for fetching the data the page needs.

## Motivation

In many cases — perhaps the significant majority — a Sapper page's `preload` function is doing nothing more than passing the page parameters to a `this.fetch` call to a sibling server route. In other words, a blog post page like the `blog/[slug].svelte` one in the [Sapper template](https://github.com/sveltejs/sapper-template/blob/master/src/routes/blog/%5Bslug%5D.svelte) might have some code like this...

```svelte
<script context="module">
	export async function preload(page) {
		const res = await this.fetch(`blog/${page.params.slug}.json`);

		if (res.ok) {
			return {
				article: await res.json()
			};
		} else {
			const { message } = await res.json();
			this.error(res.statusCode, message);
		}
	}
</script>
```

...which necessitates an additional `blog/[slug].json.js` file like this:

```js
export async function get(req, res, next) {
	const post = await get_post_somehow(req.params.slug);
	if (post) {
		res.writeHead(200, {
			'Content-Type': 'application/json'
		});

		res.end(JSON.stringify(post));
	} else {
		res.writeHead(404, {
			'Content-Type': 'application/json'
		});

		res.end(JSON.stringify({
			message: 'Not found'
		}));
	}
}
```

Essentially, the `preload` function is nothing but boilerplate, which is anathema to the Svelte philosophy.

Furthermore, when server-rendering this route, we have to do something rather odd: we must make an HTTP request *to ourselves*, while jumping through a number of hoops to ensure that we have the correct cookies for the request. This feels complex and wasteful.

## Detailed design

Building on @lukeed's work in [#27](https://github.com/sveltejs/rfcs/pull/27), this RFC proposes that we support server-only `preload` functions.

Our blog post component could be rewritten thusly:

```svelte
<script context="module server">
	export async function preload(page) {
		const article = await get_post_somehow(page.params.slug);

		if (article) {
			return { article };
		} else {
			this.error(404, {
				message: 'Not found'
			});
		}
	}
</script>
```

> 🐃 I leave the question of whether it's `module:ssr`, `module server` or something else entirely to [#27](https://github.com/sveltejs/rfcs/pull/27) — though I'll just throw out the suggestion that having a space-separated list of contexts has a nice future-proof feeling to it

When server-rendering this page, Sapper could generate preloaded props trivially:

```js
const props = await Component.preload.call({
	redirect: (statusCode, location) => {...},
	error: (statusCode, error) => {...}
}, page, session);
```

Upon client-side navigation, Sapper would behave as though an implicit preload function had been generated. It would need to make a request to some known path; one logical choice would be to use the same path as for the page itself, but using an `Accept` header to determine whether to return HTML or JSON:

```svelte
<script context="module client">
	export async function preload(page) {
		// This function is implicit, you wouldn't have to write it
		const res = await this.fetch(`blog/${page.params.slug}`, {
			credentials: 'include',
			headers: {
				Accept: 'application/json'
			}
		});

		if (res.ok) {
			return await res.json();
		}

		if (res.statusCode >= 300 && res.statusCode < 400) {
			return this.redirect(res.statusCode, res.headers.location);
		}

		this.error(res.statusCode, (await res.json()).message);
	}
</script>
```

> 🐃 A possible alternative to the `preload` signature would be to use the existing `get` signature of server routes. This could potentially allow a page to define `post`, `patch` and `delete` handlers as well. It does however reintroduce boilerplate around `Content-Type` etc.

One minor benefit of this approach: client-side components get a little bit smaller.

### Custom client-side preload logic

In some cases, you _do_ need different logic on the client as on the server:

* You might have client-side storage that contains multiple pages of items, loaded via infinite scroll, rather than the single page you get when server-rendering
* On a couple of occasions I've added code to `preload` that delays returning a value in client-side navigation until some images have been preloaded, for the sake of a layout that is immediately stable

It should be possible to take advantage of a server preload function while retaining that flexibility. One idea: a new `this.load` function for client preloads:

```svelte
<script context="module client">
	export async function preload(page) {
		try {
			const data = await this.load();

			if (data.article.image) {
				await load_image(data.article.image);
			}

			return data;
		} catch (err) {
			this.error(err.statusCode, err.message);
		}
	}

	function load_image(src) {
		return new Promise(fulfil => {
			const img = new Image();
			img.onload = img.onerror = () => fulfil();
			img.src = src;
		});
	}
</script>
```

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

I would argue that this pattern is generally applicable enough that it could completely replace the current way of doing things. We would still need a concept of server routes (some routes, like `/auth/logout`, don't belong in a component), but I suspect we could get rid of `this.fetch` entirely.

> 🐃 Given that, it's possible that we should take the opportunity to have a broader rethink of `preload`, such as whether it should continue to only return data corresponding to props, or should instead return an object with metadata (obviating the need for `this.redirect` and `this.error`, but also adding cache headers, differentiating between pages that can be generated at build time vs server-rendered at runtime, etc):

```svelte
<script context="module server">
	export async function load(page) {
		const article = await get_post_somehow(page.params.slug);

		return {
			'cache-control': 'max-age=60000',
			status: article ? 200 : 404,
			static: true, // makes this the equivalent of Next's getStaticProps instead of getServerSideProps
			props: {
				article
			}
		};
	}
</script>
```

> Would the acceptance of this proposal mean the Svelte guides must be
re-organized or altered? Does it change how Svelte is taught to new users
at any level?

Paradoxically, it makes Sapper both easier and harder to learn — it makes it easier to start building an app, but ultimately increases the API surface area. I *think* it would be a net positive.

> How should this feature be introduced and taught to existing Svelte
users?

Using the existing migration guide.

## Drawbacks

* At present, if you follow the `blog/[slug].svelte`/`blog/[slug].json.js` convention, it's very easy to treat your web app as an API — just slap a `.json` on the end of a URL and you're done. We'd lose that.
* A shared `preload` function allows you to immediately redirect if `session.user === undefined`, for example. Under this RFC's model, the client-side Sapper runtime would need to make an HTTP request to determine that, if we were relying on implicit client preloads.
* `Accept: application/json` implies our preload functions can only return JSON — at present, `preload` functions can return anything that [devalue](https://github.com/Rich-Harris/devalue) supports, though I suspect this benefit is more academic than practical
* While this is an opportunity to revisit our design decisions around the `preload` signature, including whether to provide things like `this.fetch`, we're also talking about breaking changes. No-one likes breaking changes if they can be avoided.


## Alternatives

The yaks in this RFC (🐃) are somewhat hairy, so I've leaving space open for discussion around those rather than explicitly considering alternative designs.

If we were to rethink aspects of `preload`, a couple of things I strongly recommend reading are the [documentation on pages and data fetching](https://nextjs.org/docs/basic-features/pages) from Next, and the section on location-based cache in [this Remix blog post](https://blog.remix.run/p/remix-preview), not because they directly pertain to the topic of this RFC, but because they would likely inform whatever updated design we adopted.

## Unresolved questions

* How to identify SSR/client script blocks (i.e. [#27](https://github.com/sveltejs/rfcs/pull/27))
* The exact signature of server/client `preload` functions (but also whether we continue to call them `preload`, or adopt new functions that permit non-GET requests)
* Also, something I haven't thought too deeply about, but which I assume is solvable: how the Sapper client runtime can 'know' whether a page has a server preload function, so that it knows to use the implicit client preload