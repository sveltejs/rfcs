- Start Date: 2021-05-24
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Interpolate JS in style tag

## Summary

This RFC proposes a syntactic sugar to interpolate JS inside a component's style tag:

```svelte
<script>
  let num = 10
</script>

<div>Hello</div>

<style>
  div {
    width: js(num + 'px');
    overflow: hidden;
  }
</style>
```

## Motivation

Passing variables from script to style is usually done through CSS variables.

For example, this is how we do it today:

```svelte
<script>
  let num = 10
</script>

<div style="--width: {num}px">Hello</div>

<style>
  div {
    width: var(--width);
    overflow: hidden;
  }
</style>
```

However,

1. This technique isn't as well known to beginners.

2. Leads to boilerplate for each variable.

3. Potential CSS variable naming collision.

With built-in support for interpolating JS in the style tag, we can alleviate the problems above.

## Detailed design

### Technical Background

There is currently a [feature request](https://github.com/sveltejs/svelte/issues/758) which was closed in favor of the [style properties RFC](https://github.com/sveltejs/rfcs/pull/13). Though in my honest opinion, that RFC did not really answer the goal of the feature request, which was to pass variables into the style tag within a component.

Recently, a similar style interpolation feature had also [landed in Vue 3.2](https://github.com/vuejs/rfcs/pull/231) and it was well-received by the community. As many ideas and concerns has been spurred and addressed there, we can follow its footsteps and implement our own variant.

### Implementation

I've made an [experiment](https://github.com/bluwy/svelte-js-in-css) of a Svelte preprocessor that introduces a new CSS function `js( <js-expression> )`.

For completeness, here's the input and the expected high-level output that Svelte will work with:

**Input**:

```svelte
<script>
  let num = 10
</script>

<div>Hello</div>

<style>
  div {
    width: js(num + 'px');
    overflow: hidden;
  }
</style>
```

**Output**:

```svelte
<script>
  let num = 10
</script>

<div style="display: contents; --naVbV1: {num + 'px'}; ">
<div>Hello</div>
</div>

<style>
  div {
    width: var(--naVbV1);
    overflow: hidden;
  }
</style>
```

#### Why `js()`

`js()` was chosen as it's the least web-breaking feature and plays in the same area as `calc()` and `var()`, so most preprocessors would've took into account of this function syntax. Vue 3.2 also uses a similar `v-bind()` syntax, which the decision was likely carefully made. There's also a [CSS Houdini draft](https://github.com/w3c/css-houdini-drafts/issues/857) that further expands on custom CSS functions, so there could be a future where this would be a standard syntax.

#### Experiment caveats

The preprocessor however has caveats that a full built-in Svelte integration can solve:

##### Hashing function is not the same as Svelte's `cssHash` option.

When generating a hash to scope the CSS variable, it can't be done at the preprocessing level as the hash is only computed after preprocessing is done. A built-in integration can avoid this issue.

##### A wrapping `display: contents` div is used

The preprocessor wraps the entire markup with `<div style="display: contents" />` in a similar fashion of how passing CSS custom properties to component works. A built-in integration could avoid having double `div`s if we use CSS custom properties and this feature together, since we can share them internally.

## How we teach this

I'm currently referring this feature as "JS in CSS" as it makes sense for other single file component libraries too, like Vue. I'm not aware of any existing naming convention for this feature.

For Svelte, a new tutorial chapter and documentation needs to be added.

## Drawbacks

- Browsers that don't support CSS custom properties and `display: contents` would fail. However, as this feature is opt-in, it's up to the component authors to decide. Plus, we're already one leg in with this drawback for the style properties feature.

- May be hard to debug where the JS expressions come from, for example, `width: var(--abc123)`. Though there could be ways to remedy this, for example slicing the first word of a JS expression as the CSS variable name => `width: var(--boxWidth-abc123)`

## Alternatives

### Do nothing

Encourage the way we did things before:

```svelte
<script>
  let num = 10
</script>

<div style="--width: {num}px">Hello</div>

<style>
  div {
    width: var(--width);
    overflow: hidden;
  }
</style>
```

### Use `{}` syntax instead of `js()`

Another obvious syntax would be using `{}`, but this could easily trip up CSS preprocessors, which results in more complex tooling to support this syntax. Syntax highlighting would fail too without special support. I don't think it's worth sacrificing complexity for familiarity.

### Attach CSS variables directly to element, instead of a wrapping `display: contents` div

For example, directly attaching `style="--abc123: 10px"` to the CSS-selected component. It is entirely possible to support this technique. However, the implementation may not be trivial, for example, usage of interpolating JS variables in CSS keyframes need to be linked back to the CSS-selected element.

I've only had a brief look at [Stylesheet.ts](https://github.com/sveltejs/svelte/blob/b295d68ec696a00e4d52cc7f86fed149b13062d2/src/compiler/compile/css/Stylesheet.ts), and it may require some serious work to support this behaviour.

## Unresolved questions

1. Are there any other drawbacks with a custom `js()` syntax?
2. Can the new `js()` syntax be easily supported by the Svelte Language Server?
