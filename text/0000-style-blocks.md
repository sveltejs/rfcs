- Start Date: 2022-01-22
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Style blocks
## Summary

A way to let components style their child
## Motivation

A way to style child from parents has been repeatedly asked and it has each time refused as it ofter amounted to simple syntactic sugar for `* :global(some-selector)` CSS selector.
While syntactic sugar would not be bad per any solution based on the above selector would pass the style to every child,not only its immediate, without any way to reset styles.

We propose a way for Parent to style only their immediate children, without breaking style encapsulation.
## Detailed design
```svelte
<!-- Parent.svelte -->
<script>
import Child from ‘./Child.svelte’
</script>
<Child>
Hello world!
</Child>
<style>
Child-block1{
  background-color:purple;
}
</style>

<!-- Child.svelte -->
<script>
import Child from ‘./Child.svelte’
</script>
<div><slot /></div>
<style>
div{
  -svelte-block:block1;
}
</style>
```
will be rewritten to
```svelte
<!-- Parent.svelte -->
<script>
import Child from ‘./Child.svelte’
</script>
<Child __classes={{block: `svelte-Parent-Child-hash`}}>
Hello world!
</Child>
<style global>
(div).svelte-Parent-Child-hash{
  background-color:purple;
}
</style>

<!-- Child.svelte -->
<script>
export let __classes
</script>
<div class={classes.block}><slot /></div>
<style>
div{
  -svelte-block:block1;
}
</style>
```

Exact syntax is open to bikeshedding but:
* A child can declare a point in their css where parents can put styles with the syntax `-svelte-block: blockName`
* A parent can put select such style blocks by writing the identifier of the svelte child  [https://developer.mozilla.org/en-US/docs/Web/CSS/Universal_selectors](including `svelte|self` for recursive calls),a `-` and the block name.This should be  interpreted by the PostCSS preprocessor as a single tag name and the capital initial letter(or `svelte|` prefix) should differentiate it from actual tag names.

In particular to pass a block style to every child a component that call itself recursively MUST write
```
svelte|self-blockRecursivelyPassed{
  -svelte-block:blockRecursivelyPassed;
}
```

Without the above CSS a child will not pass the style block to its children and a parent can only style its immediate children.

### Technical Background

> There are a lot of ways Svelte is used. It's hosted on different platforms;
> integrated with different libraries; built with different bundlers, etc. No one
> person knows everything about all the ways Svelte is used. What does someone who
> knows about Svelte but hasn't necessarily used anything outside of it need to
> know? Are there docs you can share?

> How do different libraries or frameworks implement this feature? We can take
> design inspiration from others who have done this well and improve upon the
> designs or make them better fit Svelte.

### Implementation
#### Web component
Each web component can expose a `__styles` prop, a dictionary from block names to string containing the style.
A child component can then simply interpolate each style block in the right places in the `style` tag, while a parent component can simply pass the styles to put.
#### Other implementations
Each component can expose a `__classes` prop, a string containing a class name.They will also export a dictionary `__selectors` form block names to the eventual selectors.
The child will apply  the class name to every tags in them.
The parent meanwhile will pass to the Child prop an hash of the parent name and the child name to `__class` and will take from the child’s `__selector` the eventual selector that will be used to generate the final style tag.

## How we teach this

A new chapter in the Svelte manual should be added, with an emphasis on how styles do NOT get uncontrollably inherited to every child. A chapter on how to pass styles to `svelte:self` should also be added to better enforce the above point.
## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Svelte,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the
> same domain have solved this problem differently.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD?
