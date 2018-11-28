- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Reactive declarations

## Summary

This RFC proposes an implementation of the 'destiny operator' inside Svelte components, using a little-known and rarely-used JavaScript feature called [labels](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label):

```html
<script>
  let count = 1;
  let doubled;
  let quadrupled;

  compute:doubled = count * 2;
  compute:quadrupled = doubled * 2;
</script>

<p>Twice {count} is {doubled}; twice that is {quadrupled}</p>

<button on:click="{() => count += 1}">+1</button>
```


## Motivation

In [RFC 1](https://github.com/sveltejs/rfcs/blob/reactive-assignments/text/0001-reactive-assignments.md), we introduced the idea of *reactive assignments*, in which assigning to a component's local variables...

```js
count += 1;
```

...causes the component to update.

As an indirect consequence of that change, we're able to get rid of all the boilerplate associated with Svelte 2 components, greatly simplifying both the compiler and the user's application code. But it also means getting rid of [computed properties](https://svelte.technology/guide#computed-properties), which are a powerful and convenient mechanism for push-based reactivity:

```html
<p>Twice {count} is {doubled}; twice that is {quadrupled}</p>

<button on:click="{() => count += 1}">+1</button>

<script>
  export default {
    data() {
      return { count: 1 };
    },

    computed: {
      doubled: ({ count }) => count * 2,
      quadrupled: ({ doubled }) => count * 2
    }
  };
</script>
```

The useful thing about computed properties is that they are only recalculated when their inputs change, side-stepping a common performance problem that affects some frameworks in which derived values must be recalculated on every render. Unlike some other frameworks that implement computed properties, the compiler is able to build a dependency graph of computed properties at *compile time*, enabling it to sort them topologically and generate highly efficient code that doesn't depend on expensive *run time* dependency tracking.

RFC 1 glosses over the loss of computed properties, suggesting that we could simply replace them with functions:

```html
<script>
  let count = 1;
  const doubled = () => count * 2;
  const quadrupled = () => doubled() * 2;
</script>

<p>Twice {count} is {doubled()}; twice that is {quadrupled()}</p>

<button on:click="{() => count += 1}">+1</button>
```

But in this scenario, `doubled` must be called twice (once directly, once via `quadrupled`). Not only that, but we have to trace which values are read when `doubled` and `quadrupled` are called so that we know when we need to update them; in some cases it would be necessary to bail out of that optimisation and call those functions whenever *anything* changed. In more realistic examples, this results in a lot of extra work relative to Svelte 2.

The only way to avoid that work is with a [Sufficiently Smart Compiler](http://wiki.c2.com/?SufficientlySmartCompiler). We need an alternative.


## Detailed design

In [What is Reactive Programming?](http://paulstovell.com/blog/reactive-programming), Paul Stovell introduces the 'destiny operator':

```
var a = 10;
var b <= a + 1;
a = 20;
Assert.AreEqual(21, b);
```

If we could use the destiny operator in components, the Svelte 3 compiler would be able to do exactly what it does with Svelte 2 computed properties — update `b` whenever `a` changes.

Unfortunately we can't, because that would be invalid JavaScript, and it's important for many reasons that everything inside a component's `<script>` block be valid JS. Is there a piece of syntax we could (ab)use in its place? Happily, there is — the *label*:

```js
let a = 10;
let b;

compute:b = a + 1;
```

This tells the compiler 'run the `b = a + 1` statement whenever `a` changes'.

It's only fair to acknowledge that this is *weird*. Aside from the unfamiliarity of labels (for most of us), we're used to statements running in order, top-to-bottom.

But it's not quite as weird as it might first seem. *Declarations* don't run in order; a class can extend a function constructor that is defined later, and you can freely put your exports at the top of your module and your imports at the bottom if it makes you happy. Seen in this light, `compute:b = a + 1` is a **declaration of equivalence** between `b` and the expression `a + 1`, rather than a statement.

And the concept isn't new — framework designers have invented all sorts of ways to approximate the destiny operator. MobX's `computed` function and decorator, RxJS Observables, and the computed properties in Svelte and Vue are all related ideas. The main difference with this approach is that it's syntactically much lighter, and depends on compile-time dependency tracking rather than (for example) wrapping everything in proxies and accessors.

In fact, it's similar to [Observable](https://beta.observablehq.com/), a platform for reactive programming notebooks. In both cases, expressions run repeatedly (but conservatively) in [topological order](https://beta.observablehq.com/@mbostock/how-observable-runs). The most commonly used analogy is that of a spreadsheet, where cells with formulas automatically stay consistent with the cells that they reference, without needing to update the whole dang worksheet.


### The mechanics of reactive declarations

For one thing, they're not *actually* declarations — we're simply marking statements that should be re-run periodically. The compiler output for the example at the top of this document might resemble the following:

```js
function init($$self, $$make_dirty) {
  let count = 1;
  let doubled;
  let quadrupled;

  function handle_click() {
    count += 1;
    $$make_dirty('count');
  }

  $$self.get = () => ({ count, doubled, quadrupled, handle_click });

  $$self.synchronize = $$dirty => {
    if ($$dirty.count) doubled = count * 2; $$dirty.doubled = true;
    if ($$dirty.doubled) quadrupled = doubled * 2; $$dirty.quadrupled = true;
  };
}
```

(This code is illustrative; it isn't necessarily optimal.)

A consequence of this design is that we can update multiple computed values in a single go. For example, one way to compute values for an SVG scatterplot involves iterating over data multiple times...

```js
compute:x_scale = get_scale([min_x, max_x], [0, width]);
compute:y_scale = get_scale([min_y, max_y], [height, 0]);

compute:min_x = Math.min(...points.map(p => p.x));
compute:max_x = Math.max(...points.map(p => p.x));
compute:min_y = Math.min(...points.map(p => p.y));
compute:max_y = Math.max(...points.map(p => p.y));
```

...but we could do it more efficiently (if more verbosely) in a single pass:

```js
compute:x_scale = get_scale([min_x, max_x], [0, width]);
compute:y_scale = get_scale([min_y, max_y], [height, 0]);

compute: {
  min_x = Infinity; max_x = -Infinity; min_y = Infinity; max_y = -Infinity; // reset

  points.forEach(point => {
    if (point.x < min_x) min_x = point.x;
    if (point.x > max_x) max_x = point.x;
    if (point.y < min_y) min_y = point.y;
    if (point.y > max_y) max_y = point.y;
  });
}
```

Another consequence is that it's straightforward to include side-effects (`compute: console.log(foo)`). Reasonable people can disagree about whether that is to be encouraged or not. It does suggest that a more 'honest' name (🐃) would be `autorun:` rather than `compute:`, or something to that effect. Other possibilities include `update:`, `reactive:`, `repeat:` and so on.


### Timing

Since reactive declarations are likely to depend on props passed into the component from outside, we probably don't want them to run immediately upon instantiation (when props aren't yet available).

> 🐃 There has been some discussion about whether we *should* make props available immediately, rather than waiting until the first update cycle. In other words,
>
>     export let foo = 'fallback value';
>
> would be transformed by the compiler to something like
>
>     let foo = $$prop('foo', 'fallback value');
>
> where `$$prop = (name, fallback) => name in props ? props[name] : fallback`

We also don't want to run them immediately upon every change. Recalculating `foo` after `bar` as updated...

```js
compute:foo = expensivelyRecompute(bar, baz);

function handleClick() {
  bar += 1;
  baz += 1;
}
```

...would be wasteful. Instead, reactive declarations should all be updated in one go, at the beginning of the component's update cycle.

**This does highlight a limitation** — we're not talking about a 'true' destiny operator, in which the intermediate value of `foo` *would* be available if you were to access it immediately after setting `bar`. Reactive declarations are *eventually* consistent with their inputs. This would be an important thing to communicate clearly.

It also raises the question of what should happen if reactive declaration inputs are updated inside a `beforeRender` handler, immediately after synchronization has happened.


### Read-only values

In Svelte 2, computed properties are read-only — attempting to write to them throws an error, but only at run time and only in dev mode. With this proposal we can do better: having identified computed values, we can treat any assignments to them (e.g. in an event handler) as illegal and raise a compile-time warning or (🐃) error.


### Reactive stores

[RFC 2](https://github.com/sveltejs/rfcs/blob/svelte-observables/text/0002-observables.md) introduced a proposal for reactive stores (the naming is TBD; the title of that RFC is 'Svelte observables', but at one point the favoured name was 'sources'. For now I'll refer to them as reactive stores, for alignment with reactive assignments and declarations).

Briefly, the idea is that a reactive store exposes a `subscribe` method that can be used to track a value over time; *writable* stores would also expose methods like `set` and `update`. This allows application state to be stored outside the component tree, where appropriate. Inside templates, stores can be referenced with a `$` prefix that exposes their value:

```html
<script>
  import { todos, user } from './stores.js';
</script>

<h1>Hello {$user.name}!</h1>

{#each $todos as todo}
  <p>{todo.description}</p>
{/each}
```

One limitation of reactive stores is that it's difficult to mix them with a component's local state. For example, if we wanted a filtered view of those todos, we can't simply derive a new store that uses a local filter variable —

```html
<script>
  import { derive } from 'svelte/store';
  import { todos, user } from './stores.js';

  let hideDone = false;
  const filtered = derive(todos, t => hideDone ? !t.done : true);
</script>

<h1>Hello {$user.name}!</h1>

<label>
  <input type=checkbox bind:checked={hideDone}>
  hide done
</label>

{#each $filtered as todo}
  <p>{todo.description}</p>
{/each}
```

— because `filtered` doesn't have a way to know when `hideDone` changes. Instead, we'd need to create a new store:

```diff
<script>
-  import { derive } from 'svelte/store';
+  import { writable, derive } from 'svelte/store';
  import { todos, user } from './stores.js';

-  let hideDone = false;
-  const filtered = derive(todos, t => hideDone ? !t.done : true);
+  const hideDone = writable(false);
+  const filtered = derive([todos, hideDone], (t, hideDone) => hideDone ? !t.done : true);
</script>

<h1>Hello {$user.name}!</h1>

<label>
-  <input type=checkbox bind:checked={hideDone}>
+  <input type=checkbox checked={$hideDone} on:change="{e => hideDone.set(e.target.checked)}">
  hide done
</label>

{#each $filtered as todo}
  <p>{todo.description}</p>
{/each}
```

Reactive declarations offer an alternative, if we allow the same treatment of values with the `$` prefix:

```diff
<script>
  import { derive } from 'svelte/store';
  import { todos, user } from './stores.js';

  let hideDone = false;
-  const filtered = derive(todos, t => hideDone ? !t.done : true);
+  let filtered;
+  compute:filtered = $todos.filter(t => hideDone ? !t.done : true);
</script>

<h1>Hello {$user.name}!</h1>

<label>
  <input type=checkbox bind:checked={hideDone}>
  hide done
</label>

-{#each $filtered as todo}
+{#each filtered as todo}
  <p>{todo.description}</p>
{/each}
```

The obvious problem with this is that `$todos` isn't defined anywhere in the `<script>`, which is potentially confusing to humans and computers alike. This is possible solvable with a combination of documentation and linting rules.


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

(TODO setting values ourselves inside `beforeRender` etc — and why that's unfortunate)

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?