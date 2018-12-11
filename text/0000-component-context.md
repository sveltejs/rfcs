- Start Date: 2018-12-11
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Component Context

## Summary

Component Context is used to pass values down to deeply nested components, providing a simple dependency injection (DI) approach for shared values (e.g. `store` in v2). It builds on svelte primitives of special elements and property binding to provide a familiar and idiomatic API.

## Motivation

There are many cases where access to "global" values is desired without having to pass values through all intermediate components. These can include both top-level concerns such as locale and caching and local concerns like passing scales to chart children. Here is an example of setting a top-level store value similar to svelte v2 that is available in all child components and setting local theme and x values that are available to children of each `Chart` instance.

```js
import App from './App.html';
import store from './store';

const app = new App({
  target: document.querySelector('body'),
  props: {
    store
  }
});
```

__Before__:

```html
<!-- App.html -->

<Chart {store} bind:theme={light} bind:x={chart_1_x}>
  <Bars {store} theme={light} x={chart_1_x} />
</Chart>
<Chart {store} theme={dark} bind:x={chart_2_x}>
  <Bars {store} theme={dark} x={chart_2_x} />
</Chart>

<script>
  import Chart from './Chart.html';
  import Bars from './Bars.html';

  export let store;

  let light, chart_1_x, chart_2_x;
  const dark = { bg: 'black', color: 'red' };
</script>
```

```html
<!-- Chart.html -->

<div style="background: {theme.bg}">
  <slot />
</div>

<script>
  import { calculateX } from './helpers';

  export let store;
  export let theme = { bg: 'white', color: 'blue' };
  export let x;

  $: x = calculateX(store.data);
</script>
```

```html
<!-- Bars.html -->

Bars: {store.data} {theme.color} {x}
```

__After, with Component Context__:

```html
<!-- App.html -->

<svelte:context {store} />
<!--            ^ Pass store to child components -->

<Chart>
  <Bars />
</Chart>
<Chart theme={dark}>
  <Bars />
</Chart>

<script>
  import Chart from './Chart.html';
  import Bars from './Bars.html';

  export let store;

  const dark = { bg: 'black', color: 'red' };
</script>
```

```html
<!-- Chart.html -->

<svelte:context bind:store {theme} {x} />
<!--            ^ Load store from parent context -->
<!--                       ^ Pass theme and x to child components -->

<div style="background: {theme.bg}">
  <slot />
</div>

<script>
  import { calculateX } from './helpers';

  export let theme = { bg: 'white', color: 'blue' };

  let store;
  let x;
  $: x = calculateX(store.data);
</script>
```

```html
<!-- Bars.html -->

<svelte:context bind:store bind:theme bind:x />
<!--            ^ Load store, theme, and x from parent context -->

Bars: {store.data} {theme.color} {x}
```

Using Component Context allows for `store` to be passed down to deeply nested components and `theme` and `x` to be passed down to child components of `Chart`, with a separate `theme` and `x` for each `Chart`. 

## Detailed design 

Component Context values pass in one direction only, from parent-to-child. Generally, they behave similarly to props, with a few subtle differences (that should be intuitive):

- `<svelte:context name={value} />` creates a Component Context value with `name` in all child components. It merges `name` into the current context (overriding any existing `name`) and passes this new context to all child components.
- `<svelte:context bind:name={value} />` creates a two-way binding, with new values coming in from the __parent__'s Component Context being bound to the local `value` and changes in `value` going out to the __children__'s Component Context.
- If `<svelte:context ... />` isn't defined for a component, the existing context is passed through unchanged.

```html
<svelte:context {theme} />

<ThemeToggle>
  <CurrentTheme /> <!-- Displays "Theme is dark." when toggled -->
</ThemeToggle>

<CurrentTheme /> <!-- Displays "Theme is light." even when toggled -->

<script>
  import ThemeToggle from './Child.html';
  import CurrentTheme from './CurrentTheme.html';
  
  const theme = 'light';
</script>
```

```html
<!-- ThemeToggle.html -->
<svelte:context bind:theme />

<label><input type="checkbox" checked="{theme === 'dark'}" on:change="{onToggle}"> Dark Theme</label>
<slot />

<script>
  const onToggle = e => {
    theme = e.target.checked ? 'dark' : 'light';
  }
</script>
```

```html
<!-- CurrentTheme.html -->
<svelte:context bind:theme />

<p>Theme is {theme}.</p>
```

It may be unexpected that `<CurrentTheme>` displays different values in the above example, but it's an important distinction that Component Context values are not global to the application, but instead local to the component and apply in one direction only, from parent-to-child. When `<ThemeToggle>` changes the Component Context value, it applies to all children of that component, but it does _not_ propogate back up the component tree.

## How we teach this

Component Context is an advanced feature available in some other front-end frameworks, although they differ in global vs. local values. React creates local context values with its [Context API](https://reactjs.org/docs/context.html), while Vue and Ember have global context with plugins (e.g. `this.$store`) and [Services](https://guides.emberjs.com/release/applications/services/), respectively.

Generally, Component Context represents a great way to manage global and shared values in applications. If a top-level component controls a value that is passed to many child components, this is a great case for applying Component Context. Throughout this RFC, the concept has been referred to as "Component Context" for clarity, but it should be taught as simply "Context" or `svelte:context` and can appear in the "Special elements" section of the guide.

## Drawbacks

The primary drawback with Component Context stems from something that context APIs struggle with in general: it can be unclear what context values a component depends on and where those values are coming from. Props come strictly from the parent component and are marked with the `export ...` API, making their requirements and source clear. Context values can come from anywhere up the tree and are found in the `<svelte:context ...>` component, making their requirements and source less clear. For these reasons, props should be preferred by svelte users and Component Context treated as a more advanced feature.

## Alternatives

Everything proposed in the RFC can be accomplished with props (albeit in a more verbose and explicit fashion), so the primary alternative is to do nothing. Additionally, Component Context could use a global rather than local approach, with each component using a shared, top-level context. The global approach is much less flexible and can't be used for some of the examples presented in this RFC, but it would likely be a simpler implementation.

## Unresolved questions

Is this feasible with svelte's underlying architecture changes for v3?
