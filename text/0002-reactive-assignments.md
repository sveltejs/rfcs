- Start Date: 2018-11-01
- RFC PR: https://github.com/sveltejs/rfcs/pull/4
- Svelte Issue: https://github.com/sveltejs/svelte/issues/1826

# Reactive assignments

## Summary

This RFC proposes a radically new component design that solves the biggest problems with Svelte in its current incarnation, resulting in smaller apps and less code to write. It does so by leveraging Svelte's unique position as a compiler to solve the problem of *reactivity* at the language level.

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
  };
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
        // does not exist on type {...}"
        const { foo } = this.get();
      }
    }
  };
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
  const len = () => name.length;
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
* Opportunities to adopt ideas like [time slicing and Suspense](https://auth0.com/blog/time-slice-suspense-react16/)


## Detailed design

Consider a counter component, the workhorse of examples such as these:

```html
<script>
  let count = 0;
</script>

<button on:click="{() => count += 1}">
  clicks: {count}
</button>
```

> The quotes around the event handler are unnecessary as far as Svelte is concerned, but included for the sake of non-Svelte-aware syntax highlighting

Here, we have declared a single state variable, `count`, with an initial value of `0`. The code inside the `<script>` block runs **once per component instance, when the instance is created**.

When clicking the button, the text inside the button should update to reflect the current value. This can be achieved by scheduling an update after every assignment expression (these are trivial to discover via AST traversal), resulting in something resembling this pseudo-code:

```js
button.addEventListener('click', () => {
  count += 1;
  __update('count');
});
```

`__update` is a framework-provided function that causes the view to update, with knowledge of which values have changed. The implementation of `__update` is unimportant for now, other than to note that it schedules an update **asynchronously** in contrast to Svelte's current model, for reasons to be [discussed later](#sync-vs-async-rendering).

This code transformation also applies to assignments inside the `<script>` block. This component...

```html
<script>
  let count = 0;
  const incr = () => {
    count += 1;
  };
</script>

<button on:click={incr}>
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

  return [
    // this creates the top-level `ctx` variable for `create_main_fragment`
    () => ({ count })
  ];
}, create_main_fragment);
```

**This behaviour might seem surprising, even shocking at first.** Variable assignment does not have side-effects in JavaScript, meaning that this is arguably something else — [SvelteScript](https://mobile.twitter.com/youyuxi/status/1057291776724271104), perhaps. In practice though, developers readily embrace 'magic' that makes their day-to-day lives easier as long as the mechanisms are ultimately easy to understand — something observed with React Hooks, Vue's reactivity system, Immer's approach to immutable data, and countless other examples. This does underscore the need for this code transformation to be well-documented and explained, however.


### Mutating objects and arrays

While it's good practice to use immutable data in components, Svelte should not force developers to do so. Assigning to a property of an object or array would have the same basic behaviour — `a.b = c` would trigger an update on `a`. It's possible that changes could be tracked at a more granular level (i.e. tracking `a.b` rather than `a`, so that `<p>{a.d}</p>` is unaffected), but these are implementation details beyond the scope of this RFC.

Calling array methods would *not* trigger updates on those arrays.

> `x = x` can be used to 'force' an update on `x`, in the rare cases it is necessary


### Props

Many frameworks (and also web components) have a conceptual distinction between 'props' (values passed *to* a component) and 'state' (values that are internal to the component). Svelte does not. This is a shortcoming.

There is a solution to this problem, but brace yourself — this may feel a little weird at first:

```html
<script>
  export let name = 'world';
</script>

<h1>Hello {name}!</h1>
```

Here, we are exporting a *contract* with the outside world — we are saying that a consumer of this component can specify a value for `name`, but that it will default to `world`:

```html
<Hello name="everybody"/>
```

This would compile to something like the following:

```js
const Component = defineComponent((__update) => {
  let name = 'world';

  return [
    () => ({ name }),
    props => {
      if ('name' in props) name = props.name;
    }
  ];
}, create_main_fragment);
```

**This is an abuse of syntax**, but no worse than Svelte's existing crime of abusing `export default`. As long as it is well-communicated, it is easy to understand. Using existing syntax, rather than inventing our own, allows us to take full advantage of existing tooling and editor support.

> Ordinarily in JavaScript, `export` means that other consumers of this module can read the value. In a sense, that's true (as we'll learn in the [Component API](#component-api) section), but here it also allows the consumer of this module to *write* the value. That's why we say that we're exporting a *contract* rather than a *value*.

To summarise, there are **3 simple rules** for understanding the code in a Svelte component's `<script>` block:

* The code runs once for each instantiated component
* Variable assignments trigger view updates
* `export`ed values form the component's contract with the outside world


### Lifecycle

Many components need to respond to *lifecycle events*. These are currently expressed in Svelte via four [lifecycle hooks](https://svelte.technology/guide#lifecycle-hooks) — `onstate`, `oncreate`, `onupdate` and `ondestroy`.

The `oncreate` hook runs *after* the initial render, meaning there is no way (in Svelte 2) to run setup code immediately upon instantiation — something this proposal solves. There is still a use for it, as we'll see, but in many cases `oncreate` will be unnecessary.

That setup code could include work that needs to be undone when the component is removed. For that, we import the `onDestroy` lifecycle function (no longer called 'hooks', to avoid confusion with React Hooks which have a fundamentally different mechanism):

```html
<script>
  import { onDestroy } from 'svelte';
  import { format_time } from './helpers.js';

  let time = new Date();

  const interval = setInterval(() => {
    time = new Date();
  }, 1000);

  onDestroy(() => {
    clearInterval(interval);
  });
</script>

<p>The time is {format_time(time)}</p>
```

The `onDestroy` callback is associated with the component instance because it is called during instantiation; no compiler magic is involved. Beyond that (if it is called outside instantiation, an error is thrown), there are no rules or restrictions as to when a lifecycle function is called. Reusability is therefore straightforward:

```js
// helpers.js
import { onDestroy } from 'svelte';

export function useInterval(fn, ms) {
  const interval = setInterval(fn, ms);
  onDestroy(() => clearInterval(interval));
  fn();
}
```

```html
<script>
  import { useInterval, formatTime } from './helpers.js';

  let time;
  useInterval(() => time = new Date(), 1000);
</script>

<p>The time is {formatTime(time)}</p>
```

> This might seem less ergonomic than React Hooks, whereby you can do `const time = useCustomHook()`. The payoff is that you don't need to run that code on every single state change, and it's easier to see which values in a component are subject to change, and *when* a specific value is changing.

There are three other lifecycle functions required — `onMount` (similar to `oncreate` in Svelte v2), `beforeUpdate` (similar to `onstate`) and `afterUpdate` (similar to `onupdate`):

```html
<script>
  import { beforeUpdate, afterUpdate, onMount } from 'svelte';

  export let foo;

  beforeUpdate(() => {
    // this callback runs whenever props change
  });

  afterUpdate(() => {
    // this callback runs after the view is updated
  });

  onMount(() => {
    // this runs once, after the first `afterUpdate`

    return function() {
      // this (optional) returned function runs on destroy,
      // allowing references (to timeouts etc) to stay within
      // the function, and preventing DOM-specific cleanup
      // code running in an SSR context
    };
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

Under this proposal, `beforeUpdate` callbacks run only once per cycle. This makes infinite loops impossible:

```html
<script>
  import { beforeUpdate } from 'svelte';

  export let temperature = 32;
  let getting = null;
  let previous_temperature;

  beforeUpdate(() => {
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

Any `afterUpdate` callbacks would run after the view was updated, whether as a result of prop or state changes. Assignments in an `afterUpdate` callback would result in an immediate re-render (once all `afterUpdate` callbacks have run, instead of on the next frame) but would *not* cause the callback to run again. This would allow components to respond to layout changes, for example.


---

The remainder of this section will address how existing Svelte concepts are affected by this RFC. Some things (CSS etc) are unmentioned because they are unaffected, though if items are missing please raise your voice.


### Directives

In Svelte 2 there are six different kinds of directive:

* `on:[event]` and `on:[event]=[callExpression]`
* `use:[action]` and `use:[action]=[argument]`
* `ref:[name]`
* `in:[transition]`, `out:[transition]` and `transition:[transition]`, and `in:[transition]=[params]` etc
* `bind:[name]` and `bind:remote=[local]`
* `class:[name]` and `class:[name]=[expression]`

Currently, directive values do not have curly braces as delimiters. This is an occasional source of confusion (and compiler complexity) that can be resolved by making directives more like attributes:

```html
<!-- current -->
<div class:foo="a ? b : c">...</div>

<!-- proposed -->
<div class:foo="{a ? b : c}">...</div>
```

Even though this is two extra characters (we preserve the quotes because we care about syntax highlighting in non-Svelte-aware environments) it's arguably more readable and consistent. This pattern would be used across all directives, but there are particular changes that apply to `ref:` and `on:`, discussed below.


#### `ref:`

The `ref:` directive is anomalous in that it cannot have a value. This relates to a very common feature request — to be able to set refs inside `each` blocks. We can fix both things — and reduce API surface area — by replacing `ref:x` with `bind:this={x}`:

```html
<canvas bind:this={canvas} {width} {height}></canvas>
```

In Svelte 2, that element could be accessed (from `oncreate` onwards) as `this.refs.canvas`. Under this proposal, refs are simple variables:

```html
<script>
  import { afterUpdate } from 'svelte';

  export let width;
  export let height;
  let canvas;
  let ctx;

  afterUpdate(() => {
    if (!ctx) ctx = canvas.getContext('2d');
    draw_some_shapes(ctx);
  });
</script>
```

(Since a binding can have any [l-value](https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue), we can also do `bind:this={array[index]}` inside `each` blocks.)

The example above could compile to something like the following:

```js
import { afterUpdate } from 'svelte';

const Component = defineComponent((__update) => {
  let width;
  let height;
  let canvas;
  let ctx;

  afterUpdate(() => {
    if (!ctx) ctx = canvas.getContext('2d');
    draw_some_shapes(ctx);
  });

  return [
    () => ({ width, height }),
    props => {
      if ('width' in props) width = props.width;
      if ('height' in props) height = props.height;
    },
    refs => {
      if ('canvas' in refs) canvas = refs.canvas;
    }
  ];
}, create_main_fragment);
```

The same would apply to component refs.


#### `on:`

Currently, inline event handlers must be call expressions...

```html
<button on:click="set({ x: x + 1 })">increment</button>
```

...and the callee must be either a method of the component, the event, or some other whitelisted thing. It's weird, inconsistent and annoying to learn. Since there are no longer any problems around `this`, we can simplify things greatly by passing functions rather than their call expressions:

```html
<button on:click="{() => x += 1}">increment</button>
```

This also allows us to do fairly sophisticated things like this:

```
<input on:keydown="{e => e.which === 39 ? next(e) : e.which === 37 ? prev(e) : null}">
```

In Svelte 2, there is a shorthand:

```html
<!-- these are equivalent: -->
<div on:click>...</div>
<div on:click="fire('click', event)">...</div>
```

It's not yet clear if we should keep this, or replace it entirely with event bubbling, discussed [below](#events).

**Custom events** in Svelte 2 enable you to use user-defined events in addition to built-in DOM events:

```html
<div on:drag="handleDrag(event)">drag me</div>

<script>
  export default {
    events: {
      drag(node, callback) {
        // implementation goes here
      }
    }
  };
</script>
```

This doesn't translate well to Svelte 3. But we can do something better — we can use actions instead:

```html
<script>
  import { drag } from 'svelte/gestures'; // 🐃 ???

  function handle_drag(arg) {...}
</script>

<div use:drag={handle_drag}>drag me</div>
```

> (The 🐃 emoji used throughout this document indicates a yak that needs shaving.)

The event system itself will be discussed in more detail [below](#events).


### namespace/tag options

The default export from a Svelte 2 component can include compiler options — for example declaring a namespace (for SVG components) or a tag name (for custom elements):

```js
export default {
  namespace: 'svg',
  tag: 'my-cool-thing'
};
```

Under this proposal, these options would be expressed by a new `<svelte:meta>` tag:

```html
<svelte:meta namespace="svg" tag="my-cool-thing">
```

### Script-less components

At present it is possible to create components with no `<script>` block:

```html
<h1>Hello {name}!</h1>
```

This is convenient and simple, and should be preserved. Under this proposal the above would be equivalent to the following — any value referenced in the markup becomes an exported variable:

```html
<script>
  export let name;
</script>

<h1>Hello {name}!</h1>
```


### Events

Svelte 2 components can fire events with `this.fire(eventName, optionalData)`. These events can be listened to programmatically, with `component.on(eventName, callback)`, or declaratively in component markup:

```html
<Widget on:flubber="doSomething()">
```

Since there's no more `this`, there's no more `this.fire`, which means we need to rethink events. This gives us an opportunity to align with web components, where `CustomEvent` is used ([issue here](https://github.com/sveltejs/svelte/issues/1655)) — this also gives us event bubbling, which avoids needing to manually propagate events.

The high-level proposal is to introduce a function called `createEventDispatcher` that would return a function for dispatching events:

```js
import { createEventDispatcher } from 'svelte';

const dispatch = createEventDispatcher();

let i = 0;
const interval = setInterval(() => {
  dispatch(i++ % 2 ? 'tick' : 'tock', {
    answer: 42
  });
}, 1000);
```

This would create a `CustomEvent` with an `event.type` of `tick` (or `tock`) and an `event.detail` of `{ answer: 42 }`.

> 🐃 We could match the arguments to `new CustomEvent(name, opts)` instead — where `detail` is one of the options passed to the constructor alongside things like `bubbles` and `cancelable`

> Note that `new CustomEvent` is unsupported in IE; legacy mode would need to use `document.createEvent('whatever')` instead

As with lifecycle functions, the `dispatch` is bound to the component because of when `createEventDispatcher` is called.



### Component API

Under this proposal, there is no longer a `this` with state and methods *inside* a component. But there needs to be a way to interact with the component from the outside world.

Instantiating a top-level component remains the same, except that `data` is renamed to `props` to match the language used elsewhere:

```js
import App from './App.html';

const app = new App({
  target: document.querySelector('body'),
  props: {
    name: 'world'
  }
});
```

Exported properties can be exposed as accessors:

```js
app.name; // world
app.name = 'everybody'; // triggers an update synchronously
```

This creates consistent behaviour between Svelte components that are compiled to custom elements, and those that are not, while also making it easy to understand the component's contract.

Of the five **built-in methods** that currently comprise the [component API](https://svelte.technology/guide#component-api) — `fire`, `get`, `set`, `on` and `destroy` — we no longer need the first two. `on` and `destroy` are still necessary, and we'll keep `set` for cases where someone needs to change multiple props at the top-level with a single update.

To differentiate those built-in methods from regular properties (so that people don't need to worry about potential conflicts), `on`, `destroy` and `set` become `$on`, `$destroy` and `$set`.

In some cases, a component that is designed to be used as a standalone widget will create its own **custom methods**. In Svelte 2, these are lumped in with 'private' (except not really) methods. Under this proposal, custom methods are just exported variables that happen to be functions:

```html
<script>
  let visible;

  export function show() {
    visible = true;
  }

  export function hide() {
    visible = false;
  }
</script>

<div class="peekaboo">
  {#if visible}
    <p>now you see me</p>
  {/if}
</div>
```

```js
import Peekaboo from './Peekaboo.html';

const peekaboo = new Peekaboo(...);
peekaboo.show();
```

An `export const`, `export function` or `export class` would be a signal to the compiler that a property is read-only — in other words, attempting to assign to it would cause an error.


### `preload` and `setup`

There is a rather awkward mechanism for declaring static properties on a component constructor in Svelte 2 — the `setup` hook:

```html
<script>
  export default {
    setup(Component) {
      Component.staticMethod = () => {...};
    }
  };
</script>
```

This is deeply weird, and due to an oversight (that we can't correct without a breaking change) only runs on the client.

Since [Sapper](https://sapper.svelte.technology/) requires that components have some way of declaring their data dependencies prior to rendering, and since `setup` is so cumbersome, there is a special case made for `preload`. The `preload` function is attached to components on both client and server, and has no well-defined behaviour; it is purely convention.

We can do better, **but it requires something potentially controversial** — a second `<script>` block that, unlike the one we've been using so far, runs a single time rather than upon every instantiation — `context="module"`:

```html
<script context="module">
  export function preload({ params }) {
    return this.fetch(`data/${params.id}.json`).then(r => r.json()).then(things => {
      return { things };
    });
  }
</script>

<script>
  export let things;
</script>

{#each things as thing}
  <p>{thing}</p>
{/each}
```

`things` would be injected into the instance `<script>` block at the same time as props (i.e. between instantiation and first render).

Anything exported from `context="module"` would be available to other modules in the normal fashion:

```js
import Widget, { preload } from './Widget.html';

Promise.resolve(preload(params)).then(props => {
  widget = new Widget({
    target,
    props
  });
});
```

Default exports would be forbidden, since they would conflict with the component itself.

> Conceptually, `context="module"` is a place to put any functions that are shared between instances. This wouldn't be necessary day-to-day however, as the compiler can automatically hoist functions that don't reference internal state.


### Server-side rendering

The compiler running with the `generate: 'ssr'` option produces completely different code from `generate: 'dom'`. This is because server-rendered components do not have a lifespan — they are rendered, then immediately discarded, so rather than having a concept of 'instances' an SSR component exports a function that, given some data, creates HTML and CSS.

This does create some subtle behaviour differences however: there is no real place to do any kind of setup work, and `oncreate` (and the other lifecycle hooks) never run. Component bindings are also brittle.

Under this proposal, the code inside the `<script>` block *would* run for server-rendered components, which also means that `ondestroy` would need to run. `onMount`, `beforeUpdate` and `afterUpdate` would be no-ops.

The API would stay the same as it currently is (we experimented with a method of exporting an SSR `$render` function from a regular component, but it turns out to be impractical for various reasons).


### Store

A [store](https://svelte.technology/guide#state-management) can be attached to a component, and it will be passed down to all its children. This allows components to access (and manipulate) app-level data without needing to pass props around. Svelte 2 provides a convenient `$` syntax for referencing store properties:

```html
<h1>Hello {$name}!</h1>
```

Similarly, store methods can be invoked via event listeners:

```html
<button on:click="$set({ count: count + 1 })">increment</button>
```

The store can also be accessed programmatically:

```js
const { foo } = this.store.get();
this.store.doSomething();
```

Since there is no longer a `this`, we need to reconsider this approach. At the same time, the `get`/`set`/`on`/`fire` interface (designed to mirror the component API) feels slightly anachronistic in light of the rest of this RFC. A major problem with the store in its current incarnation is its lack of typechecker-friendliness, and the undocumented hoops necessary to jump through to integrate with popular libraries like MobX.

A proposal for a store replacement exists in the form of [RFC 2](https://github.com/sveltejs/rfcs/pull/5).


### Spread props

It's convenient to be able to pass down props from a parent to a child component without knowing what they are. In Svelte 2 we can do this — it's hacky but it works:

```html
<!-- Lazy.html -->
{#await loader() then mod}
  <svelte:component this={mod.default} {...props}/>
{/await}

<script>
  export default {
    computed: {
      props: ({ loader, ...props }) => props
    }
  };
</script>
```

That opportunity no longer exists in this RFC. Instead we need some other way to express the concept of 'all the properties that were passed into this component'. One suggestion is to use the `bind` directive on (🐃) the `<svelte:meta>` element described above:

```html
<svelte:meta bind:props/>

<script>
  import Foo from './Foo.html';
  import Bar from './Bar.html';

  let props;

  const subset = () => {
    const { thingIDontWant, ...everythingElse } = props;
    return everythingElse;
  };
</script>

<Foo {...props}/>
<Bar {...subset()}/>
```


### Sync vs async rendering

One of the things that differentiates Svelte from other frameworks is that updates are synchronous. In other words:

```js
app.set({ size: 'supersize' });
console.log(app.refs.box.offsetWidth); // already updated
```

This is easily understood, and if there is an error during rendering it is easy to isolate the source of the problem since there is usually a short stack trace to the offending `set` call.

But it also has major drawbacks. It can result in the cyclical behaviour observed above, and it provides the framework no opportunity to optimise updates by (for example) batching operations that are likely to result in DOM reads (such as transition or `onupdate` callbacks) separately from the framework-initiated cycle of DOM writes. It also prevents us from implementing ideas like [time slicing](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html).

In addition, while it's possible to batch changes *within a component* (`this.set({ a, b, c })` results in a single update), changes *across* components (or those that affect stores and components together) are not batched. Now that `set` is no longer available, it is important that variable reassignments don't result in a sync update, but rather schedule an update (in a microtask).

Exceptions to this rule could be made where appropriate. For example if you're tracking the height of a list...

```html
<script>
  import { afterUpdate } from 'svelte';

  export let items = [];
  let list;
  let listHeight = 0;

  afterUpdate(() => {
    listHeight = list.offsetHeight;
  });
</script>

<p>the list is {listHeight}px tall:</p>

<ul ref:list>
  {#each items as item}
    <li>{item}</li>
  {/each}
</ul>
```

...then you want the change to `listHeight` to be reflected in the view as soon as all the `onupdate` callbacks have fired, *not* after an animation frame (which would result in lag).

Similarly, in terms of the public component API, it might be necessary for `customElement.foo = 1` to result in a synchronous update.


### Suspense

This proposal isn't directly concerned with [implementing Suspense](https://github.com/sveltejs/svelte/issues/1736), but care should be taken to ensure that Suspense can be implemented atop this proposal without breaking changes.


### TypeScript

A major advantage of this proposal over Svelte 2 is that components become far more amenable to typechecking and other forms of static analysis. Even without TypeScript, VSCode is able to offer much richer feedback now than with Svelte 2.

An eventual goal is to be able to use TypeScript in components:

```html
<script lang="typescript">
  export let name: string;
</script>

<h1>Hello {name}!</h1>
```

Editor integrations would ideally offer autocompletion and as-you-type typechecking inside the markup. This is not my area of expertise, however, so I would welcome feedback on this proposal from people who are more familiar with the TypeScript compiler API and ecosystem.


### Custom elements

Svelte would continue to offer a custom element compile target. No real changes would be involved here, since this proposal brings 'vanilla' Svelte components in line with web components viz. property access and event handling.

The `props` compiler option is redundant now that the contract is defined via `export`ed variables. The `tag` option can be set via `<svelte:meta>` as discussed above.


### Dependency tracking

In Svelte 2, 'computed properties' use compile-time dependency tracking to derive values from state, avoiding recomputation when dependencies haven't changed. These computed properties are 'push-based' rather than 'pull-based', which can result in unnecessary work:

```html
{#if visible}
  <p>{bar}</p>
{/if}

<script>
  export default {
    data: () => ({
      foo: 1,
      visible: false
    }),

    computed: {
      bar: ({ foo }) => expensivelyCalculateBar(foo)
    }
  };
</script>
```

In the example above, there is no real need to calculate `bar` since it is not rendered, but it happens anyway.

[RFC 3](https://github.com/sveltejs/rfcs/pull/8) proposed a Svelte 3 friendly alternative to computed properties that is more flexible and involves less boilerplate.

### svelte-extras

Most of the methods in [svelte-extras](https://github.com/sveltejs/svelte-extras) no longer make sense. The two that we do want to reimplement are `tween` and `spring`.

Happily, these will no longer involve monkey-patching components:

```html
<script>
  import { beforeUpdate } from 'svelte';
  import { tween } from 'svelte/motion';
  import * as eases from 'eases-jsnext';

  export let progress = 0;
  let tweenedProgress = 0;

  let t;

  beforeUpdate(() => {
    if (t) t.stop();

    t = tween(tweenedProgress, progress, {
      easing: eases.cubicOut,
      duration: 200
    }, value => {
      tweenedProgress = value;
    });
  });
</script>

<progress value={tweenedProgress}/>
```


### Examples

The 'imagined' components are REPL links, but they do not work in the REPL (obviously) — they're just there so it's easy to see the before/after side-by-side.

* markdown editor [current](https://svelte.technology/repl?version=2.15.0&demo=binding-textarea) / [imagined](https://svelte.technology/repl?version=2.15.0&gist=a0443aa0fc68947b5fad8fae1aa63627)
* media elements [current](https://svelte.technology/repl?version=2.15.0&demo=binding-media-elements) / [imagined](https://svelte.technology/repl?version=2.15.0&gist=8562a70e1a1e708b735b7a50e80b3cfe)
* nested components [current](https://svelte.technology/repl?version=2.15.0&demo=nested-components) / [imagined](https://svelte.technology/repl?version=2.15.0&gist=cbad006c2197633ac75fc8f31c3670df)
* SVG clock [current](https://svelte.technology/repl?version=2.15.0&demo=svg-clock) / [imagined](https://svelte.technology/repl?version=2.15.0&gist=758aacc71f506518a70c0e42f0735f6c)
* Simple transition [current](https://svelte.technology/repl?version=2.15.0&demo=transitions-fade) / [imagined](https://svelte.technology/repl?version=2.15.0&gist=7bd7b4349a27342ca96b58046eb6e765)
* Custom transition [current](https://svelte.technology/repl?version=2.15.0&demo=transitions-custom) / [imagined](https://svelte.technology/repl?version=2.15.0&gist=2e9830aad338305877d226d80a9d04d4)
* Temperature converter [current](https://svelte.technology/repl?version=2.15.0&demo=7guis-temperature) / [imagined](https://svelte.technology/repl?version=2.15.0&gist=439fc5e7c89d91a31c9c9c1447434532)


## How we teach this

The success of this idea hinges on whether we can explain how **reactive assignments** work to developers who aren't compiler nerds. That means lots of examples that emphasise showing the compiled output, so that developers can understand what is happening to their code even if they don't need to care about it day-to-day.

It is also essential to have plentiful examples showing how existing Svelte patterns can be implemented in this new world.


## Drawbacks

Obviously, this is a breaking change. We're not the React team; we don't have the resources to support two separate paradigms.

It also introduces some frankly somewhat surprising behaviour. Having spent much of the week thinking about it, and toying with existing components, I do earnestly believe that this approach feels natural once you're over the initial shock, but not everyone is guaranteed to feel that way.

Overall though, this solves so many inter-related problems with Svelte that I believe the benefits to be overwhelming.


## Alternatives

We toyed with a few alternative concepts:

* Not doing anything, and attempting to fix the niggly edge cases within the current paradigm
* Adopting something akin to React Hooks. This was met with a negative reaction from the community. Despite some minor ergonomic advantages in certain cases, the downsides of Hooks (the 'rules', the reliance on repeatedly calling user code, etc) were considered greater than the advantages
* Pursuing a purer vision of reactive programming, akin to [that described by Paul Stovell](http://paulstovell.com/blog/reactive-programming). This would arguably make code more difficult to reason about, not less, and would likely introduce difficult syntactical requirements

## Unresolved questions

None, I think.
