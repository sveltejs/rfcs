- Start Date: 2018-11-01
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Reactive assignments

## Summary

This RFC proposes a radically new component design that solves the biggest problems with Svelte in its current incarnation, resulting in smaller apps and less code to write. It does so by leveraging Svelte's unique position as a compiler to solve the problem of *reactivity* at the language level.

**This RFC is a work-in-progress, and some pieces are missing!**

## Motivation

Many developers report being intrigued by Svelte, but unwilling to accommodate some of the quirks of its design. For example, using features like nested components or transitions requires that you *register* them, involving unfortunate boilerplate:

```html
<div transition:fade>
  <Nested/>
</div>

<script>
  import { fade } from 'svelte-transitions';
  import Nested from './Nested.html';

  export default {
    components: { Nested },
    transitions: { fade }
  }
</script>
```

Declaring methods is also somewhat cumbersome, and in many editor configurations leads to red squigglies since it violates the normal semantics of JavaScript:

```html
<button on:click="doSomething()">click me</button>

<script>
  export default {
    methods: {
      doSomething() {
        // In VSCode, this causes "Property 'get'
        // does not exist on type {...}
        const { foo } = this.get();
      }
    }
  }
</script>
```

Perhaps the biggest hangup is that the markup doesn't have access to arbitrary values inside the `<script>` tag. Instead, values must be *made available* to the markup, either as props passed into the component, or as `data`, `helpers` or `computed` — three Svelte-specific concepts with overlap that many people find highly confusing:

```html
<h1>Hello {capitalise(name)}!</h1>
<p>Your name has {len} letters in it.</p>

<script>
  import { capitalise } from './utils.js';

  export default {
    data() {
      // this function is invoked for each component instance, and
      // provides the 'base' for the instance's initial state,
      // but any property can be overridden from outside
      return {
        name: 'world'
      };
    },

    helpers: {
      // helpers are essentially identical to data, except that
      // their values can never change
      capitalise
    },

    computed: {
      // computed values are functions that are re-run when their
      // dependencies change, but as far as the markup is concerned
      // are just another type of value
      len: ({ name }) => name.length
    }
  };
</script>
```

There are legitimate technical and historical reasons for this design. And some people — particularly those with a background in Vue or Ractive — are familiar with it and enjoy it. But the reality is that for most of us, it sucks. It would be much nicer to do something like this:

```html
<script>
  import { capitalise } from './utils.js';
  let name = 'world';
  let len = () => name.length;
</script>

<h1>Hello {capitalise(name)}!</h1>
<p>Your name has {len()} letters in it.</p>
```

Why can't we? Because of *reactivity*. There's no way for the view to 'know' if `name` has changed.


## The reactivity problem

UI frameworks are all, to some degree, examples of reactive programming. The notion is that your view can be thought of as a function of your state, and so when your state changes the view must, well... react.

Reactivity is implemented in a variety of different ways (note: I am not expert in these frameworks other than Svelte, please correct any misapprehensions):

* **React**'s approach is to re-render the world on every state change. This necessitates the use of a virtual DOM, which in turn necessitates a virtual DOM reconciler. It is effective, but an inefficient approach for three reasons: first, the developer's code must run frequently, providing ample opportunity for 'death by a thousand paper cuts' particularly if the developer is unskilled; second, the virtual DOM itself is a short-lived object, thirdly, discovering 'what changed' via virtual DOM diffing is wasteful in the common case that the answer is 'not much'. Reasonable people can (and do) disagree about whether these inefficiences are impactful on real apps (the answer is presumably 'it depends'), but it is well known that the most reliable way to make code faster is for it to do less work, and React does a lot of work.

* **Vue** can track which values changed using getters and setters (accessors). In other words, your data forms the basis of a 'viewmodel', with accessors corresponding to the initial properties. Assigning to one of them — `vm.foo = 1` — schedules an update. This code is pleasant to write but is not without its downsides — it can result in unpredictable behaviour as values are passed around the app, and it encourages mutation. It also involves some minor computational and memory overhead, and some gotchas when setting as-yet-unknown properties (this will be fixed in Vue 3 with proxies, but presumably at the cost of IE support).

* **Svelte** provides each component with a `set` method (and a corresponding `get` method) allowing the developer to make explicit state changes. (Those changes are fully synchronous, which is both good and bad — good because the mental model and resulting stack traces are simple, bad because it can result in layout thrashing, buggy lifecycles, and infinite loops that need to be guarded against.) This is verbose when mutating or copying objects, and puts valuable things like first-class TypeScript integration essentially out of reach.

None of these are ideal. But [we're a compiler](https://mobile.twitter.com/Rich_Harris/status/1057290365395451905), which means we have tools in our toolbox that are unique to Svelte. What if we could solve reactivity *at the language level*?

This proposal outlines a novel (as far as we're aware) solution, while bringing a number of significant benefits:

* Easier-to-grok design
* Less application code
* Smaller bundles
* Simpler, more robust lifecycles
* A realistic path towards first-class TypeScript support
* Well-defined contracts between components and their consumers
* Opportunities to adopt ideas like Suspense


## Detailed design

Consider a counter component, the workhorse of examples such as these:

```html
<script>
  let count = 0;
</script>

<button on:click="count += 1">
  clicks: {count}
</button>
```

> 🐃 For familiarity, we use the current event listener syntax. It may be preferable to use `{() => count += 1}` — a large part of the reason we avoid that currently is because `this` makes it tricky, which is no longer an issue.

> (The 🐃 emoji used throughout this document indicates a yak that needs shaving.)

Here, we have declared a single state variable, `count`, with an initial value of `0`. The code inside the `<script>` block runs **once per component instance, when the instance is created**.

When clicking the button, the text inside the button should update to reflect the current value. This can be achieved by scheduling an update after every assignment expression (these are trivial to discover via AST traversal), resulting in something resembling this pseudo-code:

```js
button.addEventListener('click', () => {
  count += 1;
  __update({ count: true });
});
```

`__update` is a framework-provided function that causes the view to update, with knowledge of which values have changed. The implementation of `__update` is unimportant for now, other than to note that it schedules an update **asynchronously** in contrast to Svelte's current model, for reasons to be discussed later.

This code transformation also applies to assignments inside the `<script>` block. This component...

```html
<script>
  let count = 0;
  const incr = () => {
    count += 1;
  };
</script>

<button on:click="incr()">
  clicks: {count}
</button>
```

...would be transformed to something like this:

```js
function create_main_fragment(component, ctx) {
  // ... the code that creates and maintains the view —
  // this will be relatively unaffected by this proposal
}

const Component = defineComponent((__update) => {
  let count = 0;
  const incr = () => {
    count += 1;
    __update({ count: true });
  };

  // this creates the top-level `ctx` variable for `create_main_fragment`
  return () => { count };
}, create_main_fragment);
```

**This behaviour might seem surprising, even shocking at first.** Variable assignment does not have side-effects in JavaScript, meaning that this is arguably something else — [SvelteScript](https://mobile.twitter.com/youyuxi/status/1057291776724271104), perhaps. In practice though, developers readily embrace 'magic' that makes their day-to-day lives easier as long as the mechanisms are ultimately easy to understand — something observed with React Hooks, Vue's reactivity system, Immer's approach to immutable data, and countless other examples. This does underscore the need for this code transformation to be well-documented and explained, however.


### Props

Many frameworks (and also web components) have a conceptual distinction between 'props' (values passed *to* a component) and 'state' (values that are internal to the component). Svelte does not. This is a shortcoming.

The proposed solution is to use the `export` keyword to declare a value that can be set from outside as a prop:

```html
<script>
  export let name = 'world';
</script>

<h1>Hello {name}!</h1>
```

This would compile to something like the following:

```js
const Component = defineComponent((__update, __props) => {
  let name = 'world';

  __props((changed, values) => {
    name = values.name;
    __update(changed);
  });

  // this creates the top-level `ctx` variable for `create_main_fragment`
  return () => { name };
}, create_main_fragment);
```

**This is an abuse of syntax**, but no worse than Svelte's existing crime of abusing `export default`. As long as it is well-communicated, it is easy to understand. Using existing syntax, rather than inventing our own, allows us to take full advantage of existing tooling and editor support.

To summarise, there are **3 simple rules** for understanding the code in a Svelte component's `<script>` block:

* The code runs once for each instantiated component
* Variable assignments trigger view updates
* `export`ed values form the component's contract with the outside world


### Lifecycle

Many components need to respond to *lifecycle events*. These are currently expressed in Svelte via four [lifecycle hooks](https://svelte.technology/guide#lifecycle-hooks) — `onstate`, `oncreate`, `onupdate` and `ondestroy`.

The `oncreate` hook is redundant, since the `<script>` code runs once per instantiation. That code could include setup work that needs to be undone when the component is removed. For that, we import the (🐃) `ondestroy` lifecycle function (no longer called 'hooks', to avoid confusion with React Hooks which have a fundamentally different mechanism):

```html
<script>
  import { ondestroy } from 'svelte';
  import { format_time } from './helpers.js';

  let time = new Date();

  const interval = setInterval(() => {
    time = new Date();
  }, 1000);

  ondestroy(() => {
    clearInterval(interval);
  });
</script>

<p>The time is {format_time(time)}</p>
```

The `ondestroy` callback is associated with the component instance because it is called during instantiation; no compiler magic is involved. Beyond that (if it is called outside instantiation, an error is thrown), there are no rules or restrictions as to when a lifecycle function is called. Reusability is therefore straightforward:

```html
<script>
  import { use_interval, format_time } from './helpers.js';

  let time;
  use_interval(() => {
    time = new Date();
  }, 1000);
</script>

<p>The time is {format_time(time)}</p>
```

```js
// helpers.js
import { ondestroy } from 'svelte';

export function use_interval(fn, ms) {
  const interval = setInterval(fn, ms);
  ondestroy(() => clearInterval(interval));
  fn();
}
```

> This might seem less ergonomic than React Hooks, whereby you can do `const time = useCustomHook()`. The payoff is that you don't need to run that code on every single state change.

There are two other lifecycle functions required — (🐃) `onstate` and (🐃) `onupdate`:

```html
<script>
  import { onstate, onupdate } from 'svelte';

  export let foo;

  onstate(() => {
    // this callback runs whenever props change
  });

  onupdate(() => {
    // this callback runs after the view is updated
  });
</script>
```

> Note that the `changed`, `current` and `previous` arguments, present in Svelte 2, are absent from these callbacks — I believe they are now unnecessary.

Currently, `onstate` and `onupdate` run before and after every state change. Because Svelte 2 doesn't differentiate between external props and internal state, this can easily result in confusing cyclicality:

```js
<script>
  export default {
    data: () => ({
      temperature: 32,
      getting: null
    }),

    onstate({ changed, current, previous }) {
      if (changed.temperature && previous) {
        // does this call result in `onstate` immediately being
        // called again?
        this.set({
          getting: current.temperature > previous.temperature
            ? 'warmer'
            : 'colder'
        });
      }
    }
  };
</script>
```

Under this proposal, the `onstate` callback runs whenever props change but *not* when private state changes. This makes cycles impossible:

```html
<script>
  import { onstate } from 'svelte';

  export let temperature = 32;
  let getting = null;
  let previous_temperature;

  onstate(() => {
    // this assignment will *not* result in the callback
    // firing again
    getting = temperature > previous_temperature
      ? 'warmer'
      : 'colder';

    previous_temperature = temperature;
  });
</script>
```

> Note that `previous_temperature`, if unused in the view, will not get the reactive treatment.

Any `onupdate` callbacks would run after the view was updated, whether as a result of prop or state changes. Assignments in an `onupdate` callback would result in a synchronous re-render but would *not* cause the callback to run again. This would allow components to respond to layout changes, for example.

A special `onupdate` case is that you often need to do work a single time once the component is mounted. This could be done in userland...

```js
onupdate(once(() => {
  doInitialSetup();
}));
```

...but it may turn out to be better to have a dedicated function for that.


### Refs

TODO (including component refs)

### Store

TODO

### Spread props

TODO

### namespace/tag options

TODO

### Events

TODO

### Component bindings

TODO

### Component API

TODO

### Preload

TODO

### Server-side rendering

TODO

### Standalone components

TODO

### Sync vs async rendering

TODO

### Suspense

TODO

### TypeScript

TODO

### Custom elements

TODO

### Dependency tracking

TODO

### Examples

TODO (show how to do equivalent of now-missing things like computed properties)



## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Svelte guides must be
re-organized or altered? Does it change how Svelte is taught to new users
at any level?

> How should this feature be introduced and taught to existing Svelte
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Svelte,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?