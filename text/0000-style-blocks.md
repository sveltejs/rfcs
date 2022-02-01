- Start Date: 2022-01-22
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Style blocks
## Summary

A way to let components style their child
## Motivation

A way to style child from parents has been repeatedly asked and it has each time refused as it ofter amounted to simple syntactic sugar for `* :global(some-selector)` CSS selector.
While syntactic sugar would not be bad per any solution based on the above selector would pass the style to every child,not only its immediate, without any way to reset styles.
Specifically we want an approach that would address the concerns with past proposal expressed [https://github.com/sveltejs/rfcs/pull/22#issuecomment-664047806](here).

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
* A child can declare a point in their css where parents can put styles with the syntax `-style-block: blockName`
* A parent can put select such style blocks by writing the identifier of the svelte child  [https://developer.mozilla.org/en-US/docs/Web/CSS/Universal_selectors](including svelte|self for recursive calls),a `-` and the block name.This should be  interpreted by the PostCSS preprocessor as a single tag name and the capital initial letter(or `svelte|` prefix) should differentiate it from actual tag names.

This should answer a few of the concerns expressed by @pngwn like:

>A component should be in complete control of itself. Not only should a component's styles not leak out but other component's style should never leak in. Consider this 'Encapsulation part 2' if you will.
>
>When writing a component, you have certain guarantees that not only will the styles you write be contained within the component, but nothing from the outside should affect you either. You have a certain confidence that if it works in isolation, then it will continue to work when embedded within a complex application.
>
>There are tools in Svelte that break this expectation to a degree, but they are a bit annoying to use, which makes it an active decision on the part of the developer. The API hints at the way we want you to do things because we feel that this will give the better experience.
>
>If a component is to accept styles from outside, then that should be part of the component's API, however that is expressed. It could be props, or it could be css variables, or it could be something else we haven't thought of yet. What we don't want to happen is to bless an approach that inverts this control, allowing an arbitrary parent to impact the inner-workings of a component.
>
>We also believe that this explicit API should be relatively granular and work at the selector or value level, not at the element level. This is why we think classes are "a blunt instrument", as Rich put it. Exposing even a single class gives you a clear entry point to the entire component's markup allowing complete control over every aspect. We need a more granular way for a component to define an explicit style interface.
Each child defines their styling API explicitly in the form of style blocks.
>And they don't solve the biggest issue with global
>
>Not only are they only sugar, but all of the proposed expansions to global also don't address its most significant issue: once you open the gate, it never stops. Global selectors, even when scoped to a subtree, cascade just like regular CSS would. This might be fine for a leaf component, but anywhere else in your app, this is the CSS equivalent of crossing your fingers and hoping that bad things won't happen.
>
>If there was a way to mark a global 'boundary', a point at which the cascade might start and stop, then expanding global as a per-component opt-in might be more reasonable. But I don't know if this would be possible or desirable, it still lacks the granularity mentioned earlier.

In particular to pass a block style to every child a component that call itself recursively MUST write
```css
svelte|self-blockRecursivelyPassed{
  -style-block:blockRecursivelyPassed;
}
```

Without the above CSS in the child component a child will not pass the style block to its children and a parent can only style its immediate children.

This way we have effectively created a selector that does NOT cascade down its children.

>We don't want additional runtime checks
>
>Another issue with some CSS-based proposals is that the components don't know whether or not they will be affected. In some cases, there would probably need to be some runtime checks to work this out.
>
>Svelte components are compiled file-by-file; they have no idea how they will be used or what the context will be. They do not know at compile-time, what their parent is doing (not their children). To see whether or not classes or other CSS from the parent is meant to impact it/ should be applied, there may need to be runtime checks on every component, regardless of whether or not they were using the feature. We are pretty strict about trying to avoid features that add cost to components even when those features aren't used.

While not in scope of this rfc, a future api for checking whether a parent is styling a children could be implemented by checking whether the prop `__class` or `__styles` are passed and have the relevant fields as seen in the implementation section.
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

##### `@keyframes`

Because duplicate `@keyframes` are never cascated, if a `-style-block` is encountered inside a keyframe the entire keyframe block is duplicated with the style slots inlined as needed in the parent.
`animation-name:` properties in the child referring to the keyframes with the style block are removed and put in the style generated by the parent, while the keyframe generated by the parent gets a unique name.

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
