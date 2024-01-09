- Start Date: 2024-01-09
- RFC PR:
- Svelte Issue:

# A way to disable unknown prop warnings

## Summary

Make it possible to disable unknown prop warnings on a per component basis using

```svelte
<svelte:options silenceUnknownPropWarnings={true} />
```

## Motivation

As of now, when a parent component renders a child component with props that are not present on the child component

```svelte
// Parent.svelte
<script>
import Child from './Child.svelte'
</script>

<Child foo="foo" bar="bar" />
```

```svelte
// Child.svelte
<script>
export let foo;
</script>

<p>{foo}</p>
```

a warning is printed to the console

```console
<Child> was created with unknown prop 'bar'
```

This is a sensible default behaviour in most cases and this RFC does not intend to change any of this.

Take for example our use case at [svelte flow](https://svelteflow.dev/) (a library that renders flowgraphs). With our current architecture, it's possible to mount any user-provided Svelte component as a node. These nodes receive information (which is managed by svelte flow) about their current state with props (its position in the flow, its label, if it's selected or not, etc.). It is not mandatory for a user-provided custom component to implement all of these props - some are only needed in special situations.

For example, if we want to render 10 custom node components in svelte flow, where the user-provided child component only implements 5 of the 10 possible passed props, the browser console will be flooded by 50 unknown prop warnings.

This issue [has already been discussed here](https://github.com/sveltejs/svelte/issues/5892#issuecomment-762660755).

AFAIK the only workaround for this is to actually implement all props, and use them to prevent further unused prop warnings.

```svelte
// Child.svelte
<script>
export let foo;
export let bar;
bar; // this statement prevents unused prop warnings
</script>

<p>{foo}</p>
```

Here is [a more extreme example in the svelte flow documentation.](https://svelteflow.dev/learn/guides/custom-nodes#suppress-unknown-prop-warnings)

## Detailed design

### Technical Background

`<svelte:options />` already exists as way to communicate certain per component configurations to the compiler.

### Implementation

The idea of this RFC is to have the possibility to set this at the top of a component:

```svelte
<svelte:options silenceUnknownPropWarnings={true} />
```

which suppresses all unknown prop warnings.

As already discussed, having unknown prop warnings as a default is sensible, but disabling it globally would defeat its purpose. Having a way to opt-in on specific components (ones that are expected to have unknown props) would be a good way forward.

## Alternatives

Two other possible names for this option include:

```svelte
<svelte:options suppressUnknownPropWarnings={true} />
```

```svelte
<svelte:options disableUnknownPropWarnings={true} />
```

## Unresolved Questions

In genral, the goal of the warning is to raise awareness to improper usage of child components. However, the motivation for this change stems from a architectural design, where the parent component acts as a black box and the control of the user is acted out inside of child components.
This raises the question where to silence the prop warnings? Does the parent silence the prop warnings (parent accepts unproper usage of child components) or does the child silence them(child accepts unproper calls by parent component)?

I prefer the former.
