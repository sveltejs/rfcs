- Start Date: 2020-09-01
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Splitting out Sapper's CSS handling

## Summary

Remove Sapper's [`sapper-internal` CSS code splitting plugin](https://github.com/sveltejs/sapper/blob/b6faa5a83301b3a7b34316248542f8d132f11f00/src/core/create_compilers/RollupCompiler.ts#L79) and add [rollup-plugin-postcss](https://github.com/egoist/rollup-plugin-postcss) to the template instead.

 `rollup-plugin-postcss` would not be strictly required, but just suggested in the template. There are probably other Rollup plugins you could use as well.

## Motivation

* Moves us closer to the goal of splitting up Sapper into smaller pieces
* Gives the user more control over their CSS
	* Allows post-processing of CSS
    * Would allow people to use other CSS plugins, which isn't possible today because Sapper owns the CSS processing step ([#699](https://github.com/sveltejs/sapper/issues/699))
* Reduces the amount of code we're responsible for maintaining. Allows us to rely on a battle tested and simpler implementation.
* Improves performance by loading CSS for dynamically imported components only when needed instead of putting that CSS in the main CSS loaded on every page
* Replaces a big source of non-obvious behavior. Users think their Rollup config is entirely in the Rollup config. They don't know that we add our own Rollup plugin to tweak the CSS. This can cause a lot of confusion and a number of bug reports in both the Sapper and Rollup repos have been difficult to understand because there's this sort of hidden behavior that's happening

## Detailed design

### Technical background

`rollup-plugin-svelte` creates JS chunks so that they do something like `import 'mycomponent.css'`. This is nice because it associates css files with their corresponding JS files. However, Rollup only knows how to handle JS files, so if you leave it like this then it will fail. You need another plugin to rewrite it in some way. Today Sapper does that with the `sapper-internal` plugin which deletes the contents of the CSS file so that Rollup doesn't encounter difficulty processing them. It then spits them out to disk, keeps track of which JS files it saw it in, and loads it when those JS files are loaded. However, this process is a bit complicated. `sapper-internal` does the roughly following:
* Deletes the contents of CSS files and stores them in a map
* Creates a CSS chunk for every JS chunk
* Puts all CSS chunks corresponding to JS chunks that are dynamically loaded into a main CSS chunk
* Writes the chunks to disk
* Stores info about the chunks, their dependencies, and which routes they correspond to into a data structure which is used to generate build.json

However, another alternative would be to rewrite the JS chunk to include the CSS chunk directly. Then we just need to load the JS chunk and don't need a complicated accounting process to keep track of which CSS chunks correspond to which JS chunks and routes. This is how the [inject functionality from `rollup-plugin-postcss`](https://github.com/egoist/rollup-plugin-postcss#inject) works. This is also how [Webpack's asset management](https://webpack.js.org/guides/asset-management/) and [style-loader](https://webpack.js.org/loaders/style-loader/) work. This would simplify a number of edge cases in Sapper's CSS handling, which is fairly complex.

You could likely use a number of plugins for this step. However, `rollup-plugin-postcss` is one that's widely used and works here well. If you use it then it will handle CSS imports by updating your code to look something like:

```
function add_css() {
	var style = element("style");
	style.id = "svelte-7jmz2r-style";
	style.textContent = "nav.svelte-7jmz2r{border-bottom:1px solid #ff6600;color:var(--fg-light);font-weight:300;padding:0 1em}.icon.svelte-7jmz2r{display:block;width:1em;height:1em;float:left;font-size:2em;position:relative;top:0.4em;box-sizing:border-box;margin:0 0.5em 0 0}ul.svelte-7jmz2r{margin:0;padding:0}ul.svelte-7jmz2r::after{content:'';display:block;clear:both}li.svelte-7jmz2r{display:block;float:left}";
	append_dev(document.head, style);
}
```

There would be some difference in behavior here for dynamic imports. Right now, we include the CSS for dynamically imported components on every page. Then when the component is imported its CSS is already available. However, this results in a larger payload for most pages. With the proposed implementation, the CSS would only be fetched when the dynamic import occurs which would result in smaller transfers and better latency for most pages.

Another plugin I had looked at was [rollup-plugin-css-chunks](https://github.com/domingues/rollup-plugin-css-chunks), which behaves somewhat similarly to Sapper's current implementation. It outputs a bundle manifest that would need to be read by Sapper to locate the appropriate CSS files to load. I was trying to avoid Sapper having any specific knowledge of CSS handling and to instead keep it all contained within a Rollup plugin, so I chose `rollup-plugin-postcss` instead because it's a much better fit at the current time from that perspective. I did file a [feature request for rollup-plugin-css-chunks](https://github.com/domingues/rollup-plugin-css-chunks/issues/4) though in the hopes that it may become a good or better alternative in the future for some users.

### Implementation

I actually went ahead and implemented this since it was pretty easy

See https://github.com/sveltejs/sapper/pull/1482 and https://github.com/sveltejs/sapper-template/pull/260

## How we teach this

Update the migration guide and Rollup template. This would be a major version bump

## Drawbacks

We have less overall control over how users setup their CSS. Additional user control means they might change settings in ways we haven't tested or anticipated.

It's probably slightly worse performance for hydrated apps. The CSS would be included in the server rendered page and JS chunk. This is different than today where both load an external stylesheet. It's possible that the user could find another Rollup CSS plugin that would use a `link` to an external stylesheet instead or request that as an option to `rollup-plugin-postcss`. That would remove the duplicate bandwidth for CSS in hydrated apps, but would result in a request chain unless the appropriate preload headers are set (i.e. request and execute the JS and then it requests and executes the CSS). However, hydrated apps already have a lot of duplicated transfers. E.g. the data the user is displaying on the page would be shown on the server rendered page and then requested again by the client JS, so this is not introducing a new issue.

## Alternatives

Two main alternatives:
* Status quo
* Split Sapper's CSS plugin out into a new plugin. This would be far less powerful and something extra we need to maintain. It's also very difficult to do with the current architecture and would need to be changed to work much more like `rollup-plugin-postcss` to be able to split it out.

## Unresolved questions

Mostly I would want some help testing this. Cases to test:
* SSR
* Check that CSS code splitting happens appropriately
* Dynamic imports in pages and layouts
* CSS on error pages
* Dev mode and production mode
* Check that we did not impact Webpack projects
* Check that JS preloading was not impacted
* Various values of `preserveEntryChunk`
