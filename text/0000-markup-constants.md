- Start Date: 2020-09-09
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Markup constants

## Summary

Add a new `{@const ...}` tag that defines a local constant.

## Motivation

Consider a component with an `each` block:

```svelte
<script>
  export let boxes;
</script>

{#each boxes as box}
  <div
    class="box"
    style="width: {box.width}px; height: {box.height}px"
  >
    {box.width} * {box.height} = {box.width * box.height}
  </div>
{/each}
```

Suppose we'd like to add a `large` class for boxes over 10,000 square pixels. We could do it like this...

```svelte
<script>
  export let boxes;
</script>

{#each boxes as box}
  <div
    class="box"
    class:large={box.width * box.height >= 10000}
    style="width: {box.width}px; height: {box.height}px"
  >
    {box.width} * {box.height} = {box.width * box.height}
  </div>
{/each}
```

...but that duplicates the `box.width * box.height` expression. If we were to add `medium` and `jumbo` classes, this would quickly get out of hand. We could define a helper function instead...

```svelte
<script>
  export let boxes;

  const area = box => box.width * box.height;
</script>

{#each boxes as box}
  <div
    class="box"
    class:large={area(box) >= 10000}
    style="width: {box.width}px; height: {box.height}px"
  >
    {box.width} * {box.height} = {area(box)}
  </div>
{/each}
```

...but adding logic to the `<script>` is unfortunate, and we're still calculating the area twice per box.

We could create what our forebears called a 'viewmodel'...

```svelte
<script>
  export let boxes;

  const boxes_with_area = boxes.map(box => ({
    ...box,
    area: box.width * box.height
  }));
</script>

{#each boxes_with_area as box}
  <div
    class="box"
    class:large={box.area >= 10000}
    style="width: {box.width}px; height: {box.height}px"
  >
    {box.width} * {box.height} = {box.area}
  </div>
{/each}
```

...but that feels like a hack.

These situations crop up from time to time, and the various workarounds are labour-intensive, involve wasted computation, and impede readability and refactoring.

## Detailed design

Suppose we added a new `{@const ...}` tag to the Svelte template language. We could define `area` where it is used:

```svelte
<script>
  export let boxes;
</script>

{#each boxes as box}
  {@const area = box.width * box.height}

  <div
    class="box"
    class:large={area >= 10000}
    style="width: {box.width}px; height: {box.height}px"
  >
    {box.width} * {box.height} = {area}
  </div>
{/each}
```

The `@const` indicates that the value is read-only (i.e. it cannot be assigned to or mutated in an expression such as an event handler), and communicates, through its similarity to `const` in JavaScript, that it only applies to the current scope (i.e. the current block or element).

Attempting to read a constant outside its scope would be a reference error (unless the constant was already shadowing a value, of course):

```svelte
{#each boxes as box}
  {@const area = box.width * box.height}

  <!-- ... -->
{/each}

<p>this is a reference error: {area}</p>
```

### Hoisting and TDZ

With JavaScript `const` (and `let`), values cannot be read before they have been initialised. We could choose to do the same thing here, though in our case it is an unnecessary restriction — we could very simply hoist values to the top of their block, in the order in which they're declared:

```svelte
{#if n}
  <p>{n}^4 = {hypercubed}</p>

  {@const squared = n * n}
  {@const cubed = squared * n}
  {@const hypercubed = cubed * n}
{/if}
```

> 🐃 You could argue that this improves or degrades readability depending on your perspective. You could also argue in favour of topological ordering, as we have in the case of reactive statements/declarations. I don't yet have strong opinions one way or the other.

### Conflicts

Defining the same constant twice in a single block would be a compile error:

```svelte
{@const foo = a}
{@const foo = b}
```


## How we teach this

I've provisionally suggested `{@const ...}` for the reasons articulated above, namely the similarities to `const`, so this new feature could be called a 'const tag'. The keyword is open to bikeshedding though — perhaps someone could make a case for calling it `{@value ...}` or `{@local ...}` instead.

Accepting this proposal wouldn't require any changes to the existing docs, but would require some new documentation to be created.

## Drawbacks

The evergreen answer to the 'why should be *not* do this?' question is 'it increases the amount of stuff to learn, and is another thing that we have to implement and maintain'.

In this case though there's an additional reason we might consider not adding this. One of the arguments that's frequently deployed in favour of JSX-based frameworks over template-based ones is that JSX allows you to use existing language constructs:

```js
{boxes.map(box => {
  const area = box.width * box.height;

  return <div
    class="box"
    class:large={area >= 10000}
    style="width: {box.width}px; height: {box.height}px"
  >
    {box.width} * {box.height} = {area}
  </div>
})}
```

The complaint is that by choosing less powerful languages, template-based frameworks are then forced to reintroduce uncanny-valley versions of those constructs in order to add back in missing functionality, thereby increasing the mount of stuff people have to learn.

In general, I'm unpersuaded by these arguments (learning curve is determined not just by unfamiliar syntax, but by unfamiliar semantics and APIs as well, and the frameworks in question excel at adding complexity in those areas). But this *is* a case where it feels like we're papering over a deficiency in our language, and is the sort of thing detractors might well point to and say 'ha! see?'. Whether or not that's a reason not to pursue this RFC is a matter for collective judgment.

## Alternatives

The alternatives are shown above in the 'motivation' section; they are not good.

## Unresolved questions

* Is `@const` the best keyword?
* Should we allow multiple declarations per tag? (Probably not.)
* Does TDZ apply?
* Are declarations ordered as-authored, or topologically?