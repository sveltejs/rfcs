- Start Date: 2021-10-12
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Remove `{#await}`, add a `promisable` store

## Summary

Bear with me, this is not click-bait. I present a (in my opinion) better alternative to `{#await}` that already works _right now_ without any breaking changes to Svelte. I present a `promisable` and a `preservedPromisable` store that are more powerful than `{#await}` in every single way.

## Motivation

I picked up Svelte roughly when v3 was released. So consider this RFC from the point of view of someone that never used v1 or v2 and is not aware of how all these things grew. Well, not entirely, I did some research:

* `{#await}` was discussed and added at the end of 2017 https://github.com/sveltejs/svelte/issues/654#issuecomment-345471246
* Reactive stores with the `$prefix` syntax roughly one year later, at the end of 2018 https://github.com/sveltejs/rfcs/pull/5

When I picked up Svelte both things already existed. Keep that in mind.

My own comment on await expressions (https://github.com/sveltejs/svelte/issues/1857#issuecomment-878058564) inspired me to write this RFC. As someone who has written a fair amount of Svelte code and a rather large and complex application I can say: **I never use `{#await}`**. I also don't know if and how others are productive with it. To be fair, I haven't seen any real Svelte code bases apart from my own, so maybe I'll embarrass myself with this RFC. I did check out the first ten or so web search results to find some articles about `{#await}` to see how others are using it. But none of them went farther than showing the syntax (like the official tutorial), which is far from an actual application. It looks like "here's `{#await}`, now go figure out how to draw the rest of the owl".

Anyway, originally this motivation section contained a lot of text and paragraphs, but I've boiled it down to this list:

1. **Await blocks are not surgical**. They are all-or-nothing. You can't share elements between the different branches. You also can't do things like add a `pending` class to an element outside of your `{#await}` structure. I assume people either use HoC (blame React) or use multiple `{#await}` blocks for a single Promise. But then you also duplicate your `<slot>` for each branch? And if you nest your HoC now your Promises run in series, not parallel? Even with await-expressions this does not feel ergonomic to me. `{#await}` is modeled after how Promises are used in imperative code, this doesn't fit the reactive templates very well. It's not executed top down once, it's alive. The solution I propose is more flexible than `{#await}` while being easier to learn at the same time.
2. **You cannot preserve the original value while loading**. Basically https://github.com/sveltejs/svelte/issues/955. The solution I propose solves this as well.
3. **It's an anti-pattern to ignore Promises**. This was my original comment on the await-expression issue. `AbortController` exists. And while you can of course use it with `{#await}`, I assume most people don't. And I don't blame them, as you have to do it manually. The solution I propose does it automagically for you.

## Detailed design

Stores. I love stores. Have you heard about stores? I abstract absolutely everything into stores. Stores solve all (my) problems.

The following is an generalization of a pattern I already use, but I tried to make it as general purpose as possible. In my applications I have more specialized implementations, e.g. one specifically for `fetch`. Now that I took the time to extract the things they all have in common I can see myself migrating to this as well.

### Add a `promisable` helper to `svelte/store`

In it's basic version a `promisable` store is a `readable` store that wraps a Promise. It allows you to do exactly what an `{#await}` block does but with a regular `{#if}`. This also means you don't have to remember the different `{#await}` variants / syntax.

Example from the docs:

Before:

```svelte
{#await promise}
  <!-- promise is pending -->
  <p>Waiting for the promise to resolve...</p>
{:then value}
  <!-- promise was fulfilled -->
  <p>The value is {value}</p>
{:catch error}
  <!-- promise was rejected -->
  <p>Something went wrong: {error.message}</p>
{/await}
```

After (more on `aborted` below):

```svelte
{#if $promise.pending}
  <!-- promise is pending -->
  <p>Waiting for the promise to resolve...</p>
{:else if $promise.aborted}
  <!-- promise was rejected via abort(), not considered an actual error -->
  <p>The promise was aborted, congrats on saving I/O and CPU cycles 🎉</p>
{:else if $promise.error}
  <!-- promise was rejected with an unexpected error -->
  <p>Something went wrong: {$promise.error.message}</p>
{:else}
  <!-- promise was fulfilled -->
  <p>The value is {$promise.value}</p>
{/if}
```

For the sake of the examples assume we have a `getUserDetails(id)` function that returns a Promise with the details for the given user. We use it in the `<UserDetails id>` component. You use `promisable` like this:

```svelte
<script>
export let id;

import { promisable } from 'svelte/store';
import { getUserDetails } from './api.js';

$: details = promisable(getUserDetails(id));
</script>

<div class:pending={$details.pending}>
  {#if $details.pending}
    <p>Loading...</p>
  {:else if $details.aborted}
    <p>Aborted...</p>
  {:else if $details.error}
    <p>Something went wrong: {$details.error.message}</p>
  {:else}
    <h1>{$details.value.name}</h1>
    <img src={$details.value.profileImage} />
  {/if}
</div>
```

Alright, this is just the basic version and it already has the benefit over `{#await}` that I can use `pending` _anywhere_ outside the `{#await}` block. Peak surgical. The semantics are the same as with `{#await}`. Once `id` changes, a new store is created for the new Promise and the subscription count of the old one drops to `0`. Thus the original Promise result is simply ignored, because nobody listens to it.

But it gets better.

### Initial value

The `promisable` store accepts a second argument, which is the initial value of the `value` property. When defined this allows you to safely access `$promise.value` in _any_ Promise state, e.g. while it's still `pending`. Example:

```svelte
<script>
export let id;

import { promisable } from 'svelte/store';
import { getUserDetails } from './api.js';

$: details = promisable(getUserDetails(id), {
  name: '■'.repeat(20),
  profileImage: 'data:image/png,....'
});
</script>

<div class:pending={$details.pending}>
  <h1>{$details.value.name}</h1>
  <img src={$details.value.profileImage} />
</div>
```

The initial value will become relevant again later when we solve the "preserve the previous value when creating a new Promise" issue. But first let's improve the carbon footprint of `promisable` by aborting operations no longer relevant.

### (Automatic) abortion

`AbortController`s are sick. The first argument of `promisable` can either be a Promise or a function that returns a Promise. This factory function will receive exactly one argument, an `AbortSignal`.

Let's take the example from above and assume `getUserDetails` can be aborted via an `AbortSignal`.

```svelte
<script>
export let id;

import { promisable } from 'svelte/store';
import { getUserDetails } from './api.js';

$: details = promisable(signal => getUserDetails(id, { signal }));
</script>
```

That's it. Now every time `id` changes and the subscription count of the underlying store drops to `0` it will `abort()` the `getUserDetails` operation.

Since `async` functions are implicitly Promises, you can also do things like this inside the factory:

```svelte
<script>
export let id;

import { promisable } from 'svelte/store';

$: details = promisable(async (signal) => {
  const response = await fetch(`/users/{id}`, { signal });
  return await response.json();
});
</script>
```

Neat. We can also expose `abort()` on the store if you want to imperatively do it without creating a new `promisable` that aborts the old one.

```svelte
<script>
  function abort() {
    // Three ways to abort the operation:

    // A. This will implicitly abort the old request and put your UI in a resolved state.
    promise = promisable(Promise.resolve('le data'));

    // B. You could also do this instead if you want the UI to settle in the aborted state.
    //    A different button to (re)start the operation would then assign a new promisable.
    promise.abort()

    // C. You could also do this instead because in A `Promise.resolve` would jump into pending state for one tick.
    //    More on preservedPromisable below.
    promise = preservedPromisable('le data');
  }
</script>

<button on:click={abort}>Cancel</button>
```

Now on to the last problem: we want to preserve the value of the last resolved Promise (or the initial value) when creating a new `promisable`.

### Add a `preservedPromisable` helper to `svelte/store`

This is a thin helper that glues the two `promisable`s together to preserve the last value. The common example seems to be autocomplete suggestions, but this can be useful in a lot of cases. For example in one of my applications I use a Monaco Editor to show the value of a selected item. And when switching the selected items I don't want Monaco to briefly show an empty content, because loading the entry is pretty fast. This would look bad (flickering editor) and wastefully re-renders Monaco.

Before (resets the suggestions to empty when `query` changes):

```svelte
<script>
export let query;

import { promisable } from 'svelte/store';
import { suggestUserNames } from './api.js';

$: details = promisable(suggestUserNames(query), []);
</script>

<ul class:spinner={$details.pending}>
  {#each $details.value as suggestion}
    <li>{suggestion}</li>
  {:else}
    {#if $details.pending}
      <li><em>Loading...</em></li>
    {:else}
      <li><em>No results for "{query}"</em></li>
    {/if}
  {/each}
</ul>
```

After (keeps showing the old suggestions while the new ones are loading):


```svelte
<script>
export let query;

import { preservedPromisable } from 'svelte/store';
import { suggestUserNames } from './api.js';

// This on its own behaves like a constant `promisable` store holding a resolved Promise with the given value.
const details = preservedPromisable([]);

// This keeps using the previous `value` until the new Promise has resolved.
$: details.await(signal => getUserDetails(id, { signal }));
</script>

<ul class:spinner={$details.pending}>
  {#each $details.value as suggestion}
    <li>{suggestion}</li>
  {:else}
    {#if $details.pending}
      <li><em>Loading...</em></li>
    {:else}
      <li><em>No results for "{query}"</em></li>
    {/if}
  {/each}
</ul>
```

# Implementation

```js
import { readable, writable } from 'svelte/store';

export const promisable = function (promiseOrFactory, initialValue) {
  let controller;
  let promise;

  if (typeof promiseOrFactory === 'function') {
    // Only create an AbortController when the consumer expects the signal argument.
    if (promiseOrFactory.length === 1) {
      controller = new AbortController();
      promise = promiseOrFactory(controller.signal);
    } else {
      promise = promiseOrFactory();
    }
  } else {
    promise = promiseOrFactory;
  }

  const { subscribe } = readable(
    {
      pending: true,
      aborted: false,
      error: null,
      value: initialValue,
    },
    (set) => {
      // _maybe_ the whole factory thing belongs here?
      // I haven't considered a use-case where the subscriber count can drop to 0 and later increase again (not a pattern I use).
      // But should this really cause the operation to start again? It might have been aborted earlier, so yes?

      promise
        .then((value) => {
          set({
            pending: false,
            aborted: false,
            error: null,
            value,
          });
        })
        .catch((error) => {
          set({
            pending: false,
            aborted: error.name === 'AbortError',
            error,
            value: initialValue,
          });
        });

      return () => {
        controller?.abort();
      };
    }
  );

  return {
    subscribe,
    abort() {
      controller?.abort();
    },
  };
};

export const preservedPromisable = function (initialValue) {
  let currentValue = initialValue;
  let innerUnsub = () => {};

  const { set, subscribe } = writable({
    pending: false,
    aborted: false,
    error: null,
    value: initialValue,
  });

  return {
    subscribe(fn) {
      const unsub = subscribe(fn);

      return () => {
        unsub();
        innerUnsub;
      };
    },
    await(promiseOrFactory) {
      const pstore = promisable(promiseOrFactory, currentValue);

      innerUnsub();

      innerUnsub = pstore.subscribe((pvalue) => {
        // Re-expose the promisable value when it changes.
        set(pvalue);

        // But also remember the value of the promise.
        currentValue = pvalue.value;
      });
    },
  };
};
```

### Demo

Showcasing basically all the things.

https://svelte.dev/repl/54a1377577ed450ca112c9b4b0535084?version=3.43.1

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

It's just a store, so the only thing new to learn is a small API. To me (someone that uses stores for everything) this is not a new pattern, it's a nice generalized abstraction. I personally like the name `promisable`.

> Would the acceptance of this proposal mean the Svelte guides must be
re-organized or altered? Does it change how Svelte is taught to new users
at any level?

If this leads to `{#await}` being removed in v4, then yes. But there will be a lot more breaking changes anyway. If `{#await}` stays around, then this is just an additional way to do things.

> How should this feature be introduced and taught to existing Svelte
users?

Everyone knows about stores already. Migrating from `{#await}` to this is almost search & replace. It might even simplify existing code if aborting happens imperatively. In addition it might give some ah-ha moments of things you can do that is impossible with `{#await}` (e.g. the REPL showcases binding of the `pending` property to the parent).

## Drawbacks

I think my mental model of Svelte and stores might be different from that of others. People might be familiar with `{#await}` and feel comfortable using it, while the store variant might look funny to them.

It also might not feel as "native" or "official" like having a dedicated `{#await}` block. But what makes Promises so special that they need their own block?

## Alternatives

This could of course be a user land package. And I will continue doing this regardless if anyone else does it too or if it becomes the "official" way.

## Unresolved questions

### Am I a fool?

Is `{#await}` actually incredible? What am I missing? I'd like to see some actual usage from real applications. Do you actually use it as-is? Do your design decisions often follow the limitations of `{#await}` or are you able to solve everything with it? Do you resort to imperatively spaghettiing around it, e.g. by passing functions around?

### What about `AbortController` polyfill / support?

https://caniuse.com/abortcontroller

Three options:

1. Add one to Svelte? It's just a very simple event emitter. But there's not really a point because if the browser does not support it, none of the APIs do (except maybe your own).
2. Add a _mocked_ one to Svelte (`abort` and `addEventListener` noops). That just makes sure it doesn't crash when using it in an unsupported browser.
3. Document it and put it in the hands of the developer. If they use the `signal` argument they need to ensure their target browsers supports it. We just make sure to not crash in others browsers and only create an `AbortController` when `factoryFn.length === 1` (I went that route for the implementation).

### `aborted` state?

I prefer to have it explicitly, but consumers could also check `{#if $promise.error.name === 'AbortError'}` on their own. Different cases to consider:

1. You are not using `AbortController`, then `aborted` is always `false` and you can just ignore that branch. Same for all other branches, e.g. if you know the promise will always resolve and never reject. Similar to the different `{#await}` syntax.
2. You are using `AbortController`, but you only care about actual errors. That saves you a branch and you could do things like `if $promise.pending || $promise.aborted` or whatever you need your UI state to be (you could have an outer branch and then render an aborted-icon via an additional `$promise.aborted` branch, things that `{#await}` would make super complicated).

So I don't see `aborted` getting in your way and it might help popularize `AbortController`.

### What about SSR / SvelteKit?

As far as I understand SSR waits for `{#await}` blocks and does some async/concurrent rendering?

We can hook into `promisable` (by having a different implementation on the server) to track all Promises. There is already prior art like `$$.on_destroy.push` we could do the same when `promisable` is used in the script block. E.g. track them via `$$.promises.push` and then `Promise.all` them on the server. Do this recursively until all components are rendered and no more Promises are pending.

This would be entirely transparent to the component and developer. And it works _without_ `{#await}`, which I don't plan on using ever.