- Start Date: 2019-11-12
- RFC PR: [#13](https://github.com/sveltejs/rfcs/pull/13)
- Svelte Issue: (leave this empty)

# Passing CSS custom properties to components


## Summary

This RFC proposes an idiomatic way to pass styles to components for the purposes of theming, using CSS custom properties:

```html
<Slider
  bind:value
  min={0}
  max={100}
  --rail-color="black"
  --track-color="red"
/>
```


## Motivation

Theming components is currently too difficult. Suppose you were using the `<Slider>` component from the (fictional) Potato Design System. Perhaps it has markup like this:

```html
<span class="potato-slider">
  <input type="hidden" {value}>
  <span class="potato-slider-rail"></span>
  <span class="potato-slider-track" style="width: {100*value/max}%">
    <span class="potato-slider-thumb"></span>
  </span>
</span>
```

(For brevity I'm omitting `tabindex`, `role` and various `aria-*` attributes that a real slider component would include for a11y reasons.)

Suppose further that we want to control (say) the colour of the rail and the track, or the size of the thumb. At present, there are only bad options:


### Global CSS

Because Svelte deliberately leaves class names intact, it's possible to select `.potato-slider-rail` etc from any stylesheet. But while this can be a useful escape hatch, it's not something to encourage as a method for theming, since it breaks encapsulation (the component has no control over which aspects of its styles can be externally controlled, even while it has complete control over which values are props and which are internal state) and is likely to lead to broken or buggy styles.

It's also brittle, since it treats internal (potentially changeable) implementation details as a public API. A good theming solution is explicit about which style properties can be changed externally, just as components are explicit about which props are accessible to consumers.

Furthermore, it becomes more difficult to style components differently based on where they appear in an app.


### The `:global` modifier

A consuming component can target the same internal class names (or any other selector) using `:global(...)`, optionally inside a non-global selector to restrict the styles to children of that component.

This has all of the downsides of global CSS (except being able to style different instances of a component differently) plus more: it may result in the need for additional DOM, and... it's kinda ugly. It feels like a hack.


### Props

An alternative approach is for the component to expose props that are then used for styling:

```html
<script>
  export let railColor = "#999";
  export let trackColor = "#333";
  // ...
</script>

<span class="potato-slider">
  <input type="hidden" {value}>
  <span class="potato-slider-rail" style="color: {railColor}"></span>
  <span class="potato-slider-track" style="color: {trackColor}; width: {100*value/max}%">
    <span class="potato-slider-thumb"></span>
  </span>
</span>
```

There's a few things not to like about this. Firstly, we're conflating state and styles. Secondly, we have to overuse inline styles to apply those values, and the compiler has to generate extra code to make that possible.

Thirdly, this provides no good way to control theming at an app level. Every time `<Slider>` is used, its style properties must be repeated. In most cases, that's probably not what we want.


## Detailed design


**Note: a previous implementation proposal worked by passing styles down as `ctx.$$styles`. This would have worked, but introduced some modest but unfortunate overhead. [It's preserved here](https://gist.github.com/Rich-Harris/6ee465dca7e86e5743cb367ba0ae3bee).**

Style properties — handily distinguishable from regular properties by the leading `--` used by CSS custom properties — are essentially syntactic sugar for a wrapper element. The example at the top of this document...

```html
<Slider
  bind:value
  min={0}
  max={100}
  --rail-color="black"
  --track-color="red"
/>
```

...desugars to something like this:

```html
<div style="display: contents; --rail-color: black; --track-color: red">
  <Slider
    bind:value
    min={0}
    max={100}
  />
</div>
```

`display: contents` essentially removes the wrapper element from the DOM, but allows it to set inheritable styles including custom properties. [It's easier to show than tell](https://svelte.dev/repl/ea454b5d951141ce989bf9ce46767c71?version=3.14.0). It's supported in [all modern browsers](https://caniuse.com/#feat=css-display-contents) (ignore notes 2 and 3, they don't apply in this situation), including Edge when the Chromium version ships.

In unsupported browsers, there is a chance it would break some layouts (though setting `width: 100%; height: 100%` would fix many of those). Those browsers generally don't support custom properties anyway, so this should probably be considered a modern-browser-only feature.

🐃 It *would* be possible for someone to accidentally target the `<div>`, so it may be preferable to use a made-up element name instead (`<styles>`?). In SVG, there might not be any choice but to use a `<g>`. In 99.9% of cases it wouldn't matter in the least, but it *would* need to be documented.


### Global theming

Because this approach uses custom properties, it becomes straightforward to set theme styles globally, or for a large subtree of the app:

```css
/* global.css */
html {
  --rail-color: black;
  --track-color: red;
}
```

```html
<!-- App.svelte -->
<div>
  <OtherStuff/>
</div>

<style>
  div {
    --rail-color: black;
    --track-color: red;
  }
</style>
```

A consumer of the component can override them...

```html
<Slider --track-color="goldenrod"/>
```

...but ultimately the component itself has control over what is exposed, and can specify its own fallback values using normal CSS custom property syntax:

```html
<!-- Slider.svelte -->
<style>
  .potato-slider-rail {
    background-color: var(--rail-color, var(--potato-theme-color, 'purple'));
  }
</style>
```


## How we teach this

This ought to be straightforward to teach, at least to people already familiar with custom properties. Terminology-wise, it probably makes sense to refer to 'style props' to distinguish them from regular component props.

It would require a new tutorial chapter and updated documentation.


## Drawbacks

*Technically* this would be a breaking change, since `--foo` is currently passed down as a regular prop (albeit only accessible via `$$props['--foo']`). I feel pretty confident saying this isn't a real world concern.

There are some more salient drawbacks:

* It relies on something that is inherently global. Different components might 'claim' a given property name. While it's possible to differentiate them at the subtree level, it's not possible to do so globally.
* IE11 doesn't support custom properties or `display: contents`, so we'd be pushing the ecosystem towards incompatibility with that browser. Should we care? Probably not. Component authors who wanted to support IE11 would have to provide fallback values, and consumers of those components would have to be okay with those fallbacks.
* Regular component properties are statically analysable, which could one day allow us to have typechecking and autocompletion when using those components. The same isn't true for style properties. We could imagine some way of changing that (some syntax that lives inside, or on, the `<style>` attribute) but it's extra work that isn't considered here.


## Alternatives


### Do nothing

Expect people to continue using `:global` and friends for this purpose. It's got us this far.


### Do nothing, but encourage the use of custom properties

We could also encourage the use of CSS custom properties for theming without making any changes to Svelte itself, in which case people could get the same end result by manually wrapping their components in elements that provide custom properties:

```html
<div style="--rail-color: black; --track-color: red">
  <Slider bind:value min={0} max={100}/>
</div>
```

This has the merit of simplicity and obviousness, but it's not particularly ergonomic: it signals that we don't consider component themeability to be a problem worth solving properly.


### Do nothing, but encourage `style` forwarding

If component authors were in the habit of expecting a `style` prop, and applying it to top-level elements, we could do the same thing like so:

```html
<Slider style="--rail-color: black; --track-color: red" />
```

This arguably breaks encapsulation (it's not encouraging you to only pass down custom properties, but *any* styles to an element you don't control), and is contingent on component authors handling it in a consistent way.


### Special-case `class`

Another suggestion is to special-case the `class` property, per [#2888](https://github.com/sveltejs/svelte/pull/2888). This is arguably more in line with popular CSS-in-JS solutions. Personally, I think `class` is too blunt an instrument — it breaks encapsulation, allowing component consumers to change styles that they probably shouldn't, while also denying them a predictable interface for targeting individual styles, or setting theme properties globally.


### props-in-style

Something else that comes up from time to time is the idea of supporting `{props}` directly in the `<style>`:

```html
<script>
  export let bg = 'red';
</script>

<style>
  div {
    background: {bg};
  }
</style>
```

Aside from being an implementation nightmare, I think the proposal in this RFC is *strictly better* than props-in-style — it gives you the same expressive power in a neater, more idiomatic way, along with the global theming ability.


## Unresolved questions

I'm not a big design system user, so I would very much like to get feedback from people who are. Would this solve your problems? What have we missed?

In particular, this takes a different approach from [CSS Shadow Parts](https://drafts.csswg.org/css-shadow-parts-1/), which allows a component consumer to target selected elements, but to then apply arbitrary styles to those elements. I'm personally surprised about that, given the degree to which web component advocates prioritise encapsulation — it seems like a footgun, honestly — but I'd be eager to learn from people with relevant experience. (Note that we'd still be able to emulate that capability with `:global` — really the question is whether it needs to be first-class.)