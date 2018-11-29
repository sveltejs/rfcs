- Start Date: 2018-11-10
- RFC PR: [#5](https://github.com/sveltejs/rfcs/pull/5)
- Svelte Issue: (leave this empty)

# Reactive stores

## Summary

This RFC proposes a replacement for the existing [Store](https://svelte.technology/guide#state-management) class that satisfies the following goals:

* Works with the Svelte 3 component design outlined in [RFC 1](https://github.com/sveltejs/rfcs/pull/4)
* Allows any given component to subscribe to multiple sources of data from outside the component tree, rather than favouring a single app-level megastore
* Typechecker friendliness
* Adapts to existing state management systems like [Redux](https://redux.js.org/) or [TC39 Observables](https://github.com/tc39/proposal-observable), but does not require them
* Concise syntax that eliminates lifecycle boilerplate


## Motivation

In Svelte 1, we introduced the Store class, which provides a simple app-level state management solution with an API that mirrors that of individual components — `get`, `set`, `on`, `fire` and so on. The store can be attached to a component tree (or subtree), at which point it becomes available within lifecycle hooks and methods as `this.store`, and in templates by prefixing identifiers with `$`. Svelte is then able to subscribe to changes to app-level data with reasonable granularity and zero boilerplate.

In Svelte 2, Store support was turned on by default.

This approach works reasonably well but has some drawbacks:

* It doesn't permit namespacing or nesting; the store becomes a grab-bag of values
* Because of the convenience of the store being auto-attached to components, it becomes a magnet for things that aren't purely related to application state, such as effectful `login` and `logout` methods
* Setting deep properties (`foo.bar.baz` etc) is cumbersome
* It doesn't play well with TypeScript
* Creating derived ('computed') values involves a slightly weird API
* Working with existing state management systems isn't as natural as it should be

A large part of the reason for the design of the original store — the fact that it auto-attaches to components, for example — is that the cost of using imported objects in Svelte 1 and 2 is unreasonably high. In Svelte 3, per [RFC 1](https://github.com/sveltejs/rfcs/pull/4), that cost will be significantly reduced. We can therefore pursue alternatives without the aforementioned drawbacks.


## Detailed design

Essentially, the shift is from a single observable store of values to multiple observable values. Instead of this...

```js
export default new Store({
  user: {
    firstname: 'Fozzie',
    lastname: 'Bear'
  },
  volume: 0.5
});
```

...we create a new `writable` value store:

```js
export const user = writable({
  firstname: 'Fozzie',
  lastname: 'Bear'
});

export const volume = writable(0.5);
```

Interested parties can read (and write) `user` without caring about `volume`, and vice versa:

```js
import { volume } from './stores.js';

const audio = document.querySelector('audio');

const unsubscribe = volume.subscribe(value => {
  audio.volume = value;
});
```

Inside a component's markup, a convenient shorthand sets up the necessary subscriptions (and unsubscribes when the component is destroyed) — the `$` prefix:

```js
<script>
  import { user } from './stores.js';
</script>

<h1>Hello {$user.firstname}!</h1>
```

This would compile to something like the following:

```js
import { onDestroy } from 'svelte';
import { user } from './stores.js';

function init($$self, $$make_dirty) {
  let $user;
  onDestroy(user.subscribe(value => {
    $user = value;
    $$makeDirty('$user');
  }));

  $$self.get = () => ({ $user });
}
```


### Store API

A store *must* have a `subscribe` method, and it may also have additional methods like `set` and `update` if it isn't read-only:

```js
const number = writable(1);

const unsubscribe = number.subscribe(value => {
  console.log(`value is ${value}`); // logs 1 immediately
});

number.set(2); // logs 2

const incr = n => n + 1;
number.update(incr); // logs 3

unsubscribe();
number.set(4); // does nothing — unsubscribed
```

An example implementation of this API:

```js
function writable(value) {
  const subscribers = [];

  function set(newValue) {
    if (newValue === value) return;
    value = newValue;
    subscribers.forEach(fn => fn(value));
  }

  function update(fn) {
    set(fn(value));
  }

  function subscribe(fn) {
    subscribers.push(fn);
    fn(value);

    return function() {
      const index = subscribers.indexOf(fn);
      if (index !== -1) subscribers.splice(index, 1);
    }
  }

  return { set, update, subscribe };
}
```


### Read-only stores

Some stores are read-only, created with `readable`:

```js
const unsubscribe = mousePosition.subscribe(pos => {
  if (pos) console.log(pos.x, pos.y);
});

mousePosition.set({ x: 100, y: 100 }); // Error: mousePosition.set is not a function
```

An example implementation:

```js
function readable(start, value) {
  const subscribers = [];
  let stop;

  function set(newValue) {
    if (newValue === value) return;
    value = newValue;
    subscribers.forEach(fn => fn(value));
  }

  return {
    subscribe(fn) {
      if (subscribers.length === 0) {
        stop = start(set);
      }

      subscribers.push(fn);
      fn(value);

      return function() {
        const index = subscribers.indexOf(fn);
        if (index !== -1) subscribers.splice(index, 1);

        if (subscribers.length === 0) {
          stop();
          stop = null;
        }
      };
    }
  }
}

const mousePosition = readable(function start(set) {
  function handler(event) {
    set({
      x: event.clientX,
      y: event.clientY
    });
  }

  document.addEventListener('mousemove', handler);
  return function stop() {
    document.removeEventListener('mousemove', handler);
  }
});
```

### Derived stores

A store can be derived from other stores with `derive` (🐃):

```js
const a = writable(1);
const b = writable(2);
const c = writable(3);

const total = derive([a, b, c], ([a, b, c]) => a + b + c);

total.subscribe(value => {
  console.log(`total is ${value}`); // logs 'total is 6'
});

c.set(4); // logs 'total is 7'
```

Example implementation:

```js
function derive(stores, fn) {
  return readOnlyObservable(set => {
    let inited = false;
    const values = [];

    const unsubscribers = stores.map((store, i) => store.subscribe(value => {
      values[i] = value;
      if (inited) set(fn(...values));
    }));

    inited = true;
    set(fn(...values));

    return function stop() {
      unsubscribers.forEach(fn => fn());
    };
  });
}
```

> In the example above, `total` is recalculated immediately whenever the values of `a`, `b` or `c` are set. In some situations that's undesirable; you want to be able to set `a`, `b` *and* `c` without `total` being recalculated until you've finished. That could be done by putting the `set(fn(...values))` in a microtask, but that has drawbacks too. (Of course, that could be a decision left to the user.) Is this a fatal flaw in the design — should we strive for pull-based rather than push-based derived values? Or is it fine in reality?

Derived stores are, by nature, also read-only. They could be used, for example, to filter the items in a todo list:

```html
<script>
  import { writable, derive } from 'svelte/store.js';
  import { todos } from './stores.js';

  const hideDone = writable(false);

  const filtered = derive([todos, hideDone], (todos, hideDone) => todos.filter(todo => {
    return hideDone ? !todo.done : true;
  }));
</script>

<label>
  <input type=checkbox checked={$hideDone} on:change="{e => hideDone.set(e.target.checked)}">
  hide done
</label>

{#each $filtered as todo}
  <p class="{todo.done ? 'faded' : ''}">{todo.description}</p>
{/each}
```


### Relationship with TC39 Observables

There is a [stage 1 proposal for an Observable object](https://github.com/tc39/proposal-observable) in JavaScript itself.

Cards on the table: I'm not personally a fan of Observables. I've found them to be confusing and awkward to work with. But there are particular reasons why I don't think they're a good general solution for representing reactive values in a component:

* They don't represent a single value changing over time, but rather a stream of distinct values. This is a subtle but important distinction
* Two different subscribers to the same Observable could receive different values (!), where as in a UI you want two references to the same value to be guaranteed to be consistent
* Observables can 'complete', but declarative components (in Svelte and other frameworks) deliberately do not have a concept of time. The two things are incompatible
* They have error-handling semantics that are very often redundant (what error could occur when observing the mouse position, for example?). When they're not redundant (e.g. in the case of data coming over the network), errors are perhaps best handled out-of-band, since the goal is to concisely represent the value in a component template

Of course, some Observables *are* suitable for representing reactive values in a template, and they could easily be adapted to work with this design:

```js
function adaptor(observable) {
  return {
    subscribe(fn) {
      const subscriber = observable.subscribe({
        next: fn
      });

      return subscriber.unsubscribe;
    }
  }
}

const observable = Observable.of('red', 'green', 'blue');
const store = adaptor(observable);

const unsubscribe = store.subscribe(color => {
  console.log(color); // logs red, then green, then blue
});
```


### Examples of use with existing state management libraries

More broadly, the same technique will work with existing state management libraries, as long as they expose the necessary hooks for observing changes. (I've found this to be difficult with MobX, but perhaps I'm just not sufficiently familiar with that library — would welcome submissions.)


#### Redux

```js
// src/redux.js
import { createStore } from 'redux';

export const reduxStore = createStore((state = 0, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
});

function adaptor(reduxStore) {
  return {
    subscribe(fn) {
      return reduxStore.subscribe(() => {
        fn(reduxStore.getState());
      });
    }
  };
}

export const store = adaptor(reduxStore);
```

```html
<!-- src/Counter.html -->
<script>
  import { reduxStore, store } from './redux.js';
</script>

<button on:click="{() => reduxStore.dispatch({ type: 'INCREMENT' })}">
  Clicks: {$store}
</button>
```


#### Immer

```js
import { writable } from 'svelte/store.js';
import { produce } from 'immer';

function immerObservable(data) {
  const store = writable(data);

  function update(fn) {
    store.update(state => produce(state, fn));
  }

  return {
    update,
    subscribe: store.subscribe
  };
}

const todos = immerObservable([
  { done: false, description: 'walk the dog' },
  { done: false, description: 'mow the lawn' },
  { done: false, description: 'do the laundry' }
]);

todos.update(draft => {
  draft[0].done = true;
});
```


#### Shiz

```js
import { readable } from 'svelte/store.js';
import { value, computed } from 'shiz';

const a = value(1);
const b = computed([a], ([a]) => a * 2);

function shizObservable(shiz) {
  return readable(function start(set) {
    return shiz.on('change', () => {
      set(shiz.get());
    });
  }, shiz.get());
}

const store = shizObservable(b);

const unsubscribe = store.subscribe(value => {
  console.log(value); // logs 2
});

a.set(2); // logs 4
```


### Using with Sapper

At present, `Store` gets privileged treatment in [Sapper](https://sapper.svelte.technology) apps. A store instance can be created per-request, for example to contain user data. This store is attached to the component tree at render time, allowing `<span>{$user.name}</span>` to be server-rendered; its data is then passed to a client-side store.

It's essential that this functionality be preserved. I'm not yet sure of the best way to achieve that. The most promising suggestion is that we use regular props instead, passed into the top-level component. (In some ways this would be more ergonomic, since the user would no longer be responsible for setting up the store client-side.)

> Potential corner-cases to discuss:
> * What happens if a subscriber causes another subscriber to be removed (e.g. it results in a component subtree being destroyed)?
> * Is `$user.name` ambiguous (i.e. is `user` the store, or `user.name`?) and if so how do we resolve the ambiguity
> * What happens if `$user` is declared in a scope that has a store `user`? Do we just not subscribe?


## How we teach this

As with RFC 1, it's crucial that this be introduced with ample demos of how the `$` prefix works, in terms of the generated code.

It's arguably simpler to teach than the existing store, since it's purely concerned with data, and avoids the 'magic' of auto-attaching.


## Drawbacks

Like RFC 1, this is a breaking change, though RFC 1 will break existing stores anyway. The main reason not to pursue this option would be that the `$` prefix is overly magical, though I believe the convenience outweighs the modest learning curve.

Another potential drawback is that anything that uses a store (except the markup) must *itself* become a reactive store; they are [red functions](http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). But this problem is presumably fundamental, rather than an avoidable consequence of the approach we've happened to choose.

> [RFC 3](https://github.com/sveltejs/rfcs/blob/reactive-declarations/text/0003-reactive-declarations.md) presents an escape hatch to this problem


## Alternatives

* The existing store (but only at the top level; there is no opportunity to add stores at a lower level in the new design)
* Baking in an existing state management library (and its opinions)
* Using TC39 Observables
* Not having any kind of first-class treatment of reactive values, and relying on the reactive assignments mechanism exclusively


## Unresolved questions

* The Sapper question
* The exact mechanics of how typechecking would work
