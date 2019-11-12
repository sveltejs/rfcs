- Start Date: 2019-11-12
- RFC PR: (leave this empty)
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

It's also brittle, since it treats internal (potentially changeable) implementation details as a public API.

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

Style properties — handily distinguishable from regular properties by the leading `--` used by CSS custom properties — are passed down to components through a separate channel. The example at the top of this RFC might be converted to the following JavaScript:

```js
const slider = new Slider({
  props: {
    value: ctx.value,
    min: 0,
    max: 100
  },
  styles: {
    '--rail-color': 'black',
    '--track-color': 'red',
  }
});
```

These properties — `--rail-color` and `--track-color` etc — are essentially part of the interface of `<Slider>`, just like the `value`, `max` and `min` properties are.

Inside the Slider component, these styles would need to be applied to the top-level element — the equivalent of doing this:

```html
<span class="potato-slider" style="--rail-color: black; --track-color: red">
```

In practice that might look something like this:

```js
// internal helper
function apply_styles(node, styles) {
  for (const k in styles) {
    node.style.setProperty(k, styles[k]);
  }
}

function create_main_fragment(ctx) {
  // ...

  return {
    c() {
      span = element("span");
      apply_styles(span, ctx.$$styles);
    },
    // ...
    p(changed, ctx) {
      if (changed.$$styles) apply_styles(span, ctx.$$styles);
    },
    // ...
  };
}
```

(It would get slightly more complex for cases where there was an existing `style` attribute, particularly if it included a `--rail-color` property or was an opaque `style={styles}` type attribute. But the principle is the same.)

In the SSR case:

```js
return `<span class="${"potato-slider"}" style="${apply_ssr_styles($$styles)}">...</div>`;
```


### Inheritance

In the same way that custom properties applied to an element affect all descendant elements, custom properties applied to a component would affect all DOM within that component. It could be argued that this breaks encapsulation, but a) it matches the expectations people have from using custom properties to date, b) would in general be more convenient, and c) is more or less out of our hands since that's just how custom properties work.


### Multiple top-level elements

A component might have multiple top-level elements. In such cases, it would be necessary to apply the custom properties to all of them.

We could consider omitting them for elements that appear not to use the custom properties, and don't contain any components or any child elements that use them, but this might break expectations: it would be valid to select an element using global (or `:global`) CSS and expect the custom property to be present.


### Zero top-level elements

A trickier case is when there are no top-level elements, only components without any surrounding DOM. In these situations we would need to forward styles:

```html
<TopLevelComponent --foo={bar}>
```

```js
const toplevelcomponent = new TopLevelComponent({
  props: {...},
  styles: Object.assign({}, ctx.$$styles, {
    '--foo': ctx.bar
  })
});
```


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
* The compiler would need to generate extra code for every component (for applying received style properties, and for passing them on to top-level child components), regardless of whether they were actually used, and the helper would be included in everyone's app. (This *could* be avoided with some form of whole-app optimisation.)
* IE11 doesn't support custom properties, so we'd be pushing the ecosystem towards incompatibility with that browser. Should we care? Probably not. Component authors who wanted to support IE11 would have to provide fallback values, and consumers of those components would have to be okay with those fallbacks.
* Regular component properties are statically analysable, which could one day allow us to have typechecking and autocompletion when using those components. The same isn't true for style properties. We could imagine some way of changing that (some syntax that lives inside, or on, the `<style>` attribute) but it's extra work that isn't considered here.


## Alternatives

One alternative is to do nothing: expect people to continue using `:global` and friends for this purpose. It's got us this far.

We could also encourage the use of CSS custom properties for theming without making any changes to Svelte itself, in which case people could get the same end result by wrapping their components in elements that provide custom properties:

```html
<div style="--rail-color: black; --track-color: red">
  <Slider bind:value min={0} max={100}/>
</div>
```

This has the merit of simplicity and obviousness, and doesn't involve any extra code being generated, but it's also a hack: it signals that we don't consider component themeability to be a problem worth solving properly.

Another suggestion is to special-case the `class` property, per [#2888](https://github.com/sveltejs/svelte/pull/2888). This is arguably more in line with popular CSS-in-JS solutions. Personally, I think `class` is too blunt an instrument — it breaks encapsulation, allowing component consumers to change styles that they probably shouldn't, while also denying them a predictable interface for targeting individual styles, or setting theme properties globally.

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