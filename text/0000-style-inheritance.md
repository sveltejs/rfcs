- Start Date: 2022-05-03
- RFC PR:
- Svelte Issue:

# Targeted Style Inheritance

## Summary

An idiomatic way to allow scoped CSS rules in parent components to target elements in child components, according to two rules intended to preserve encapsulation:

1. The targeted element must have an `inherit` attribute, to signal it wants this behavior
2. The parent CSS selector must explicitly name each component between the parent and target

## Motivation

This proposal aims to address the following issues with style properties, `:global` and the slot API:

- Style properties can't offer the flexibility of CSS to parent components
- Style properties don't allow you to keep CSS away from HTML
- `:global` is a blunt instrument that encourages violation of the principle that components are their own boss
- `:global` tends to lead to uncontrolled cascade of styles
- Slots are ideal for customizing structure, but too heavyweight if you only want to customize styles

## Detailed design

**Rule 1:** To be eligible for inheritance, selectors must be strictly scoped to each component between the parent component and target element.

```html
<!-- Parent.svelte -->

<div class="blue">
    <Child />
</div>

<style>
/* Effective: */
Child button {
    color: red;
}

/* Ineffective, because Child is not part of the selector: */
.blue button {
    color: blue;
}
</style>
```

**Rule 2:** Targeted elements must opt in to inheritance.

```html
<!-- Child.svelte -->

<!-- Will be styled: -->
<button inherit>Foo</button>

<!-- Will remain unstyled: -->
<button>Bar</button>
```

### Implementation

Components that are targeted by a selector and have at least one child element with the `inherit` attribute are wrapped in a `display: contents` div, which is assigned `data-svelte-this={name}`, where `{name}` is the name of the component.

Elements with `inherit` are assigned `data-svelte-inherit={name}`, where `{name}` is the parent component of that element.

Selectors targeting a component are automatically rewritten to use `:global` in a way that enforces rules 1 and 2:

| Selector | translates to |
| - | - |
| `Child.blue button` | `* :global([data-svelte-this=Child].blue button[data-svelte-inherit=Child])` |
| `Menu > MenuItem img` | `* :global([data-svelte-this=Menu] > [data-svelte-this=MenuItem][data-svelte-inherit=Menu] img[data-svelte-inherit=MenuItem])` |

## How we teach this

The new syntax is reasonably intuitive, and it helps that targeting a component in CSS is currently a no-op. It also helps that `inherit` is an existing CSS keyword with similar semantics.

This would require a new tutorial chapter and updated documentation.

Potentially, linters could warn if a CSS selector satisfies rule 2 but not rule 1. (The reverse is likely too noisy.)

## Drawbacks

One salient drawback is that rule 1 introduces something new to consider when determining whether CSS rules will apply to an element. This is somewhat mitigated by the fact that it's already often non-trivial to determine when CSS rules will apply, due to the complex nature of CSS specificity.

## Alternatives

This issue has been discussed to near-death and similar proposals have been floated in the past. Here is a brief summary of the feedback from core maintainers:

| Suggestion | RFC or Issue | Feedback |
| ---------- | ---------- | -------- |
| Allow parent to impact child component CSS | [#13](https://github.com/sveltejs/rfcs/pull/13) | Violates the principles of CSS encapsulation and components being their own boss. Does not prevent uncontrolled cascade. |
| Component-scoped `<style>` | [comment in #13](https://github.com/sveltejs/rfcs/pull/13#issuecomment-553144981) | Breaks encapsulation. |
| Unscoped CSS by default | [svelte#901](https://github.com/sveltejs/svelte/issues/901) | Scoped CSS by default is an intentionally opinionated feature, intended to avoid over-namespacing and configuration anxiety. |
| Passing class to components | [svelte#2870](https://github.com/sveltejs/svelte/issues/2870) | Breaks encapsulation. Theming should be part of the component interface. |
| Passing scoped styles to components | [svelte#6422](https://github.com/sveltejs/svelte/issues/6422) | Unclear how this could be implemented with acceptable performance. |

## Unresolved questions

By what means can component names be parsed from CSS selectors? Is it OK to rely on component names having an uppercase first letter?

Could `!important` play a special role here, by relaxing either rule 1 or 2?

Should `inherit="global"` permit any grandparent component to target this element, even if not all intermediary components have `inherit`?
