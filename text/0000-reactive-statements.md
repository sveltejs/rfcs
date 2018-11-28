- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Reactive statements

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

In [RFC 1](https://github.com/sveltejs/rfcs/blob/reactive-assignments/text/0001-reactive-assignments.md), we introduced the idea of *reactive assignments*, in which assigning to a component's local variables (`count += 1`) causes the component to update.

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


### The destiny operator

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
var a = 10;
var b;

compute:b = a + 1;
```

This tells the compiler 'run the `b = a + 1` statement whenever `a` changes'.

It's only fair to acknowledge that this is *weird*. Aside from the unfamiliarity of labels (for most of us), we're used to statements running in order, top-to-bottom.

But it's not quite as weird as it might first seem. *Declarations* don't run in order; a class can extend a function constructor that is defined later, and you can freely put your exports at the top of your module and your imports at the bottom if it makes you happy. Seen in this light, `compute:b = a + 1` is a declaration of equivalence between `b` and the expression `a + 1`, rather than a statement.

And the concept isn't new — framework designers have invented all sorts of ways to approximate the destiny operator. MobX's `computed` function and decorator, RxJS Observables, and the computed properties in Svelte and Vue are all related ideas. The main difference with this approach is that it's syntactically much lighter, and depends on compile-time dependency tracking rather than (for example) wrapping everything in proxies and accessors.

TODO

* when exactly these statements would run (immediately before `beforeRender`? what happens if you assign to a dependency inside `beforeRender`?)
* caveats and limitations (statements wouldn't run inside an event handler, for example — you have to wait for the update cycle the event handler causes)
* blocks (multiline statements that could assign to multiple values)
* whether to call it `compute:` or `autorun:` or `update:` or something else
* how to handle assignments outside the computation. compile error?
* how these interact with sources/stores
* the alternative — setting values ourselves inside `beforeRender` etc — and why that's unfortunate


> Why are we doing this? What use cases does it support? What is the expected
outcome?

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

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