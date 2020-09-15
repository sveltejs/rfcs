- Start Date: 2020-09-01
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Splitting out Sapper's CSS handling

## Summary

Split out Sapper's [`sapper-internal` CSS code splitting plugin](https://github.com/sveltejs/sapper/blob/b6faa5a83301b3a7b34316248542f8d132f11f00/src/core/create_compilers/RollupCompiler.ts#L79) into a separate plugin that's added to template instead. The plugin suggested to be used in the template could be either [rollup-plugin-css-chunks](https://github.com/domingues/rollup-plugin-css-chunks) or a new plugin of our own creation.

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

`rollup-plugin-svelte` creates JS chunks that do something like `import 'mycomponent.css'`. This is nice because it associates css files with their corresponding JS files. However, Rollup only knows how to handle JS files, so if you leave it like this then it will fail. You need another plugin to rewrite it in some way. Today Sapper does that with the `sapper-internal` plugin which deletes the contents of the CSS file so that Rollup doesn't encounter difficulty processing them. It then spits them out to disk, keeps track of which JS files it saw it in, and loads it when those JS files are loaded. However, this process is a bit complicated. `sapper-internal` does the roughly following:
* Deletes the contents of CSS files and stores them in a map
* Creates a CSS chunk for every JS chunk
* Puts all CSS chunks corresponding to JS chunks that are dynamically loaded into a main CSS chunk
* Writes the chunks to disk
* Stores info about the chunks, their dependencies, and which routes they correspond to into a data structure which is used to generate build.json

However, another alternative would be to rewrite the JS chunk to include the CSS chunk directly. Then we just need to load the JS chunk and don't need a complicated accounting process to keep track of which CSS chunks correspond to which JS chunks and routes. This is how the [inject functionality from `rollup-plugin-postcss`](https://github.com/egoist/rollup-plugin-postcss#inject) works. This is also how [Webpack's asset management](https://webpack.js.org/guides/asset-management/) and [style-loader](https://webpack.js.org/loaders/style-loader/) work. This would simplify a number of edge cases in Sapper's CSS handling, which is fairly complex.

There would be some difference in behavior here for dynamic imports. Right now, we include the CSS for dynamically imported components on every page. Then when the component is imported its CSS is already available. However, this results in a larger payload for most pages. With the proposed implementation, the CSS would only be fetched when the dynamic import occurs which would result in smaller transfers and better latency for most pages.

### Implementation

Update - 2020-09-14: The `link` injection functionality has been merged into Sapper in [#1508](https://github.com/sveltejs/sapper/pull/1508).

We could either [use `rollup-plugin-css-chunk` if the link injection functionality from #1508 is added there](https://github.com/domingues/rollup-plugin-css-chunks/issues/4) or we could create a new plugin containing this functionality.

## How we teach this

Update the migration guide and Rollup template. This would be a major version bump

## Drawbacks

We have less overall control over how users setup their CSS. Additional user control means they might change settings in ways we haven't tested or anticipated.

It creates one new JS that that's requested: `inject_styles.js`

## Alternatives

The main alternative would be the status quo

## Unresolved questions

Will `rollup-plugin-css-chunk` accept this change or should we create a new plugin?

We should do a through job of testing:
* SSR
* Check that CSS code splitting happens appropriately
* Dynamic imports in pages and layouts
* CSS on error pages
* Dev mode and production mode
* Check that we did not impact Webpack projects
* Check that JS preloading was not impacted
* Various values of `preserveEntryChunk`
