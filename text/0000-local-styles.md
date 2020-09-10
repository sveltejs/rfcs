
- Start Date: (fill me in with today's date, 2020-09-09)
- RFC PR: [#32](https://github.com/sveltejs/rfcs/pull/32)
- Svelte Issue: (leave this empty)

# Local styles

## Summary

Allow components to contain `<style scoped>` elements (and in a future major version, just `<style>` elements), at arbitrary depths inside components, that only affect elements in the same subtree.

## Motivation

Since re-rendering in Svelte happens at a more granular level than the component, there is no artificial pressure to create smaller components than would be naturally desirable, and in fact (because one-component-per-file) there is pressure in the opposite direction. As such, large components are not uncommon.

Because of that, it's easy to end up in a situation where the styles for a given piece of markup are defined far away (in terms of number of lines) from the markup itself, which reduces the advantage of having styles and markup co-located in the first place.

When a component reaches such a size that this becomes a problem, the obvious course of action is to refactor it into multiple components. But the refactoring is complex for the same reason: extracting the styles that relate to a particular piece of markup is an error-prone manual process, where the relevant styles may be interleaved with irrelevant ones.

Even without going to that extreme, the constraint of having a single `<style>` can easily force component authors to resort to the kinds of classes-as-namespaces hacks that scoped styles are supposed to obviate.

## Detailed design

Long ago, the standards deities gifted us [`<style scoped>`](https://css-tricks.com/saving-the-day-with-scoped-css/), before removing it in favour of the arguably less-useful shadow DOM encapsulation mechanism.

We could bring it back in the context of Svelte components:

```svelte
<!-- Thing.svelte -->
<p>some red text</p>

<div>
  <p>some bold red text</p>

  <style scoped>
    p {
      font-weight: bold;
    }
  </style>
</div>

<style>
  p {
    color: red;
  }
</style>
```

This would be roughly equivalent to the following...

```svelte
<!-- Thing.svelte -->
<p>some red text</p>

<div class="my-namespace">
  <p>some bold red text</p>
</div>

<style>
  p {
    color: red;
  }

  .my-namespace p {
    font-weight: bold;
  }
</style>
```

...except that both `p` selectors would have the same specificity, with the priority determined by the order of the declarations in the outputted CSS (with deeper style elements following shallower/top-level ones).

In terms of rendered HTML and outputted CSS, it would look something like this:

```html
<p class="svelte-abc123">some red text</p>

<div>
  <p class="svelte-abc123 svelte-def456">some bold red text</p>
</div>
```

```css
p.svelte-abc123{color:red}
p.svelte-def456{font-weight:bold}
```

Alternatively, this:

```html
<p class="svelte-abc123">some red text</p>

<div>
  <p class="svelte-abc123-1">some bold red text</p>
</div>
```

```css
p[class^="svelte-abc123"]{color:red}
p.svelte-abc123-1{font-weight:bold}
```

Local styles would apply to blocks, not just elements:

```svelte
{#if condition}
  <style scoped>
    p {
      color: green;
    }
  </style>
  <p>condition green</p>
{:else}
  <style scoped>
    p {
      color: red;
    }
  </style>
  <p>condition red</p>
{/if}
```

For now, adding the `scoped` attribute is necessary for this to not be a breaking change, since non-top-level `<style>` attributes are actually rendered to the DOM. In Svelte 4, we could change that (probably undesirable) behaviour, and just have `<style>` by itself as we do at the top level.

## How we teach this

To differentiate `<style scoped>` from the existing top-level `<style>` elements, I propose that we refer to these as 'local scoped styles'.

It would require a small amount of new documentation, but wouldn't require changing what's already there, since it reuses existing concepts.

## Drawbacks

It's another thing to learn (albeit an easy thing to learn), and would increase implementation complexity.

## Alternatives

The alternative is to do nothing and leave this problem unsolved. People manage.