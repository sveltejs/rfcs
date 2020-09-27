- Start Date: 2020-09-27
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Better Slots and Component Compositon

## Summary

This RFC aims to provide a better way to create nested components and improve component composition using slots. This also aims to provide a more *svelte* like syntax for creating slots.

**Input**
```html
<div class="has-slot-named">
  <svelte:slot name="named">
    Default Content
  </svelte:slot>
</div>
<svelte:slot>
  This is the default slot.
</svelte:slot>
```
**Component**
```html
<svelte:fragment slot="named">
  Some Content
  <SomeOtherComponent />
</svelte:fragment>
```
**Output**
```html
<div class="has-slot-named">
  Some Content
  <SomeOtherComponent />
</div>
This is the default slot.
```

## Motivation

The current way of using slots is very simple but also does not have many features such as:

1. If you want a named slot to have no parent node.
This is only possible if you use the default slot. There is no way to exclude the `p` element from the slot.
```html
<slot name="named" />
```
```html
<p slot="named"></p>
```

2. There is no way to directly slot a svelte component
Svelte components have to be wrapped in a node element such as a `span` or a `div` to be slotted.
```html
<slot name="named" />
```
```html
<span slot="named">
  <Component />
</span>
```
These problems pose a major problem for component composition and styling (especially with the new flexbox system which almost everyone is using).
This RFC aims to solve all these problems and add some new features along the way.

## Detailed design

### Basic Composition
This is not so different from the current implementation, replacing the `slot` tag with the `svelte:slot` tag aims to join all the components used by svelte ( _svelte:\*_ ) and not part of the native list.

**Slots.svelte**
```html
<div>
  <svelte:slot>
    This is the default content for the default slot.
  </svelte:slot>
</div>
<svelte:slot name="named">
  Default content for named slot.
</svelte:slot>
```

**Component.svelte**
```html
<script>
  import Slots from './Slots.svelte';
</script>
<Slots>
  Hello There
  <svelte:fragment slot="named">
    Named Slot
  </svelte:fragment>
</Slots>
```

**Output**
```html
<div>
	Hello There
</div>
<svelte:slot name="named">
	Named Slot
</svelte:slot>
```

### Slot Props

**Slots.svelte**
```html
<script>
  function someFunc(a, b) {
    return a + b;
  }
</script>
<svelte:slot />
{#each ['fizz', 'buzz', 'foo', 'bar'] as i}
  <svelte:slot word={i} {someFunc} name="repeat">
    This is the default content for the default slot.
  </svelte:slot>
{/each}
```

**Component.svelte**
```html
<script>
  import Slots from './Slots.svelte';
</script>
<Slots let:word const:someFunc={add}>
  Hello There
  <svelte:fragment slot="repeated" let:word const:someFunc>
    <span>{add(word, 'world!')}</span>
  </svelte:fragment>
</Slots>
```

**Output**
```html
Hello There
<span>fizz world!</span>
<span>buzz world!</span>
<span>foo world!</span>
<span>bar world!</span>
```

## How we teach this

- Much changes to the documentation will not be needed as `slot` will be renamed to `svelte:slot` and `svelte:fragment` will be used instead of `<node slot="name">` but users must be informed of this name change and the slight change in functionality.
- Component authors must change thier `<slot />` to `<svelte:slot />` and add a wrapper node to named slots as one will not be present by default.
- This should be easy to teach as it is very similar to the existing implementation.

## Drawbacks

- Would break existing components unless backwards compatibility is offered.
- Would be best if merged in a major release.

## Alternatives

This problem has no alternative.

## Unresolved questions

- Is there a better name for `svelte:fragment`?

