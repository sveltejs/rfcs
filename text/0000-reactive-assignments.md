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

> Ordinarily in JavaScript, `export` means that other consumers of this module can read the value. In a sense, that's true (as we'll learn in the [Component API](#component-api) section), but here it also allows the consumer of this module to *write* the value. That's why we say that we're exporting a *contract* rather than a *value*.

To summarise, there are **3 simple rules** for understanding the code in a Svelte component's `<script>` block:

* The code runs once for each instantiated component
* Variable assignments trigger view updates
* `export`ed values form the component's contract with the outside world


### Lifecycle

Many components need to respond to *lifecycle events*. These are currently expressed in Svelte via four [lifecycle hooks](https://svelte.technology/guide#lifecycle-hooks) — `onstate`, `oncreate`, `onupdate` and `ondestroy`.

The `oncreate` hook runs *after* the initial render, meaning there is no way (in Svelte 2) to run setup code immediately upon instantiation — something this proposal solves. There is still a use for it, as we'll see, but in many cases `oncreate` will be unnecessary.

That setup code could include work that needs to be undone when the component is removed. For that, we import the (🐃) `ondestroy` lifecycle function (no longer called 'hooks', to avoid confusion with React Hooks which have a fundamentally different mechanism):

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

As previously mentioned, `oncreate` is used in Svelte 2 to run code after the initial render has taken place. Strictly speaking it is redundant...

```js
import { onstate, onupdate } from 'svelte';
import { once } from 'lodash-es';

export let externalProp;
let internalProp = 'defined';

onstate(once(() => {
  // externalProp is now defined
}));

onupdate(once(() => {
  // initial render has taken place
}));
```

...but we could decide (🐃) to include `oncreate` as a convenience anyway.


---

The remainder of this section will address how existing Svelte concepts are affected by this RFC. Some things (actions, transitions etc) are unmentioned because they are unaffected, though if items are missing please raise your voice.


### Refs

Refs are references to DOM nodes:

```html
<canvas ref:canvas {width} {height}></canvas>
```

In Svelte 2, that element could be accessed (from `oncreate` onwards) as `this.refs.canvas`. Under this proposal, refs are simple variables:

```html
<script>
  import { onupdate } from 'svelte';

  let canvas;
  let ctx;

  onupdate(() => {
    if (!ctx) ctx = canvas.getContext('2d');
    draw_some_shapes(ctx);
  });
</script>
```

This could compile to something like the following:

```js
import { onupdate } from 'svelte';

const Component = defineComponent((__update, __props, __refs) => {
  let canvas;
  let ctx;

  onupdate(() => {
    if (!ctx) ctx = canvas.getContext('2d');
    draw_some_shapes(ctx);
  });

  __refs(refs => {
    canvas = refs.canvas;
  });

  return () => {};
}, create_main_fragment);
```

The same would apply to component refs.


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

The high-level proposal is to introduce a function called `createEventDispatcher` (🐃) that would return a function for dispatching events:

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

As with lifecycle functions, the `dispatch` is bound to the component because of when `createEventDispatcher` is called.

Listening to events is currently done like so:

```html
<button on:click="doSomething()">click me</button>
```

This effectively becomes `{() => this.doSomething()}`. The shorthand is nice, but it is slightly confusing (the context is implicit, and the right-hand side must be a CallExpression which limits flexibility, although there's an [issue for that](https://github.com/sveltejs/svelte/issues/1766)). In the new model, there are no longer any problems around `this`, so it probably makes sense to allow arbitrary expressions instead.


### Bindings

Element bindings are a convenient shorthand, which I believe we should preserve:

```html
<input bind:value=name>

<!-- equivalent (Svelte 2) -->
<input value={name} on:input="set({ name: this.value })">

<!-- equivalent (this proposal) -->
<input value={name} on:input="{e => name = e.target.value}">
```

Component bindings are less clear-cut. In the past they've been a source of confusion for the same reason that the lifecycle hook dance causes confusion upon app initialisation. It's possible that the new lifecycle concepts will eliminate those headaches, but this requires further exploration.

The reason we can *consider* removing component bindings is that it's now trivial to pass state-altering functions down to components:

```html
<script>
  import Mousecatcher from './Mousecatcher.html';

  let mouse;

  function setPosition(x, y) {
    mouse = { x, y };
  }
</script>

<Mousecatcher {setPosition}>
  the mouse is at {mouse.x},{mouse.y}
</Mousecatcher>
```


### Component API

Under this proposal, there is no longer a `this` with state and methods *inside* a component. But there needs to be a way to interact with the component from the outside world.

Instantiating a top-level component probably needn't change (except 🐃 maybe changing `data` to `props`):

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
app.name = 'everybody'; triggers a (sync?) update
```

This creates consistent behaviour between Svelte components that are compiled to custom elements, and those that are not, while also making it easy to understand the component's contract.

Of the five **built-in methods** that currently comprise the [component API](https://svelte.technology/guide#component-api) — `get`, `set`, `fire`, `on` and `destroy` — we no longer need the first three. `on` and `destroy` are still necessary.

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

An `export const` would be a signal to the compiler that a property is read-only — in other words, attempting to assign to it would cause an error.


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

Since Sapper requires that components have some way of declaring their data dependencies prior to rendering, and since `setup` is so cumbersome, there is a special case made for `preload`. The `preload` function is attached to components on both client and server, and has no well-defined behaviour; it is purely convention.

We can do better, **but it requires something potentially controversial** — a second `<script>` block that, unlike the one we've been using so far, runs a single time rather than upon every instantiation. Let's call it (🐃) `scope="shared"`:

```html
<script scope="shared">
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


### Server-side rendering

The compiler running with the `generate: 'ssr'` option produces completely different code from `generate: 'dom'`. This is because server-rendered components do not have a lifespan — they are rendered, then immediately discarded, so rather than having a concept of 'instances' an SSR component exports a function that given some data creates HTML and CSS.

This does create some subtle behaviour differences however: there is no real place to do any kind of setup work, and `oncreate` (and the other lifecycle hooks) never run. Component bindings are also brittle.

Under this proposal, the code inside the `<script>` block *would* run for server-rendered components, which also means that `ondestroy` would need to run. `onstate` and `onupdate` would be no-ops.

I don't think the API needs to change.


### Standalone components

It's not clear how this proposal affects standalone components (as opposed to those that use shared helpers). Perhaps it doesn't?


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

I'm not yet sure how the store fits into this proposal — this is perhaps the biggest unknown at this point.


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

That opportunity no longer exists in this RFC. Instead we need some other way to express the concept of 'all the properties that were passed into this component'. The best I can come up with is an injected variable with a name like `__props__`:

```html
<script>
  import Foo from './Foo.html';
  import Bar from './Bar.html';

  const subset = () => {
    const { thingIDontWant, ...everythingElse } = __props__;
    return everythingElse;
  };
</script>

<Foo {...__props__}/>
<Bar {...subset()}/>
```

This is inelegant, but I believe the inelegance would affect a small minority of components.

> Since a component's public API ordinarily uses accessors to define the component's contract, it is likely that *inline* components would need to use a privileged non-public API, so that unanticipated props could be passed in. (This is something we should do anyway)


### Sync vs async rendering

One of the things that differentiates Svelte from other frameworks is that updates are synchronous. In other words:

```js
app.set({ size: 'supersize' });
console.log(app.refs.box.offsetWidth); // already updated
```

This is easily understood, and if there is an error during rendering it is easy to isolate the source of the problem since there is usually a short stack trace to the offending `set` call.

But it also has major drawbacks. It can result in the cyclical behaviour observed above, and it provides the framework no opportunity to optimise updates by (for example) batching operations that are likely to result in DOM reads (such as transition or `onupdate` callbacks) separately from the framework-initiated cycle of DOM writes. It also prevents us from implementing ideas like [time slicing](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html).

In addition, while it's possible to batch changes *within a component* (`this.set({ a, b, c })` results in a single update), changes *across* components (or those that affect stores and components together) are not batched. Now that `set` is no longer available, it is important that variable reassignments don't result in a sync update, but rather schedule an update (for the next animation frame, for example).

Exceptions to this rule could be made where appropriate. For example if you're tracking the height of a list...

```html
<script>
  import { onupdate } from 'svelte';

  export let items = [];
  let list;
  let listHeight = 0;

  onupdate(() => {
    listHeight = list.offsetHeight;
  });
</script>

<p>the list is {listHeight}px tall:</p>

<ul>
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
<script type="typescript">
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

In the example above, there is no need to calculate `bar` since it is not rendered.

Under this proposal, there is no longer a separate concept of computed properties. Instead, we can just use functions:

```html
<script>
  export let foo = 1;
  export let visible = false;

  const bar = () => foo;
</script>

{#if visible}
  <p>{bar()}</p>
{/if}
```

Note that we now *invoke* `bar` in the markup. If `foo` were to change while `visible` were true, we would need to update the contents of `<p>`. That means we need to track the values that could affect the return value, resulting in compiled code similar to this:

```js
function update(changed, ctx) {
  if (changed.foo) p.textContent = ctx.bar();
}
```

In some situations dependency tracking may be impossible — I'm not sure. In those cases, we can simply bail out:

```js
function update(changed, ctx) {
  p.textContent = ctx.bar();
}
```

Note that this system is now pull-based rather than push-based. One drawback over the current system of computed properties is that values that are referenced multiple times will also be calculated multiple times:

```html
{#if visible}
  <p>{bar()} — {bar()}</p>
{/if}
```

Potentially, this could be alleviated with compiler magic (i.e. detecting multiple invocations of pure functions with the same arguments, and computing once before rendering) or userland memoization, but it warrants further exploration.


### svelte-extras

Most of the methods in [svelte-extras](https://github.com/sveltejs/svelte-extras) no longer make sense. The two that we do want to reimplement are `tween` and `spring`.

Happily, these will no longer involve monkey-patching components:

```html
<script>
  import { tween } from 'svelte/animation'; // 🐃
  import * as eases from 'eases-jsnext';

  export let progress = 0;
  let tweenedProgress = 0;

  let t;

  onstate(() => {
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

The loss of computed properties as a distinct primitive will be felt by some Svelte users. It's no longer as convenient to use a value that is derived from other derived values, particularly if that value is a function.

Overall though, this solves so many inter-related problems with Svelte that I believe the benefits to be overwhelming.


## Alternatives

We toyed with a few alternative concepts:

* Not doing anything, and attempting to fix the niggly edge cases within the current paradigm
* Adopting something akin to React Hooks. This was met with a negative reaction from the community. Despite some minor ergonomic advantages in certain cases, the downsides of Hooks (the 'rules', the reliance on repeatedly calling user code, etc) were considered greater than the advantages
* Pursuing a purer vision of reactive programming, akin to [that described by Paul Stovell](http://paulstovell.com/blog/reactive-programming). This would arguably make code more difficult to reason about, not less, and would likely introduce difficult syntactical requirements

## Unresolved questions

The major TODOs centre around `Store` and spread properties.
