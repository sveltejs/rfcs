- Start Date: 2021-08-07
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# The `forward` directive.

## Summary

This RFC proposes a new `forward` directive, which lets you forward directives used on a component instance to a DOM element in said component. This allows you to call directives such as `use`, `transition`, `animation`, etc... on a component declaration. It also allows you to delegate all events to a certain element in a component's template.

## Motivation

At the moment, there are a long list of pitfalls with community-provided component libraries. Designing a "one-size-fits-all" component in svelte can be rather tedious in the current enviornment, notably when dealing with directives. For example, if you wish to forward commonly used events to a reusable `<Button>` component, you'll have to retype a long list of events like so:

```html
<button
    on:click
    on:blur
    on:focus
    on:dblclick
    on:contextmenu
    on:mousedown
    on:mouseup
    on:mouseover
    on:mouseout
    on:mouseleave
    on:keypress
    on:keydown
    on:keyup
    class="button {customClass ?? ''}"
    {...$$restProps}
>
    <slot />
</button>
```

A [solution](https://github.com/sveltejs/svelte/pull/4599) to this event issue using the `on:*` directive has been proposed. This syntax doesn't come without issues either, as `*` is nonstandard syntax in the scope of HTML events, and it doesn't reach the core limitations of component directives.

Another common problem that comes up is when working with actions. Components are incapable of utilizing the `use` directive, meaning library authors often need to export a prop like so:
```html
<script>
	export let use = [];
	
	const [action, options] = use;
</script>

<button use:action={options}>
	<slot />
</button>
```

While this also is effective in solving the issue, it adds further boilerplate and nonstandard syntax that other developers are forced to work with.

```html
<Button use={[myAction, {option: true}]}>Hi</Button>
```

Similar problems arise with the `transition` and `animation` directives.

## Detailed design

### Forwarding all events to an element

This example uses the same `<Button />` component described earlier.

```html
<button forward:on class="button {customClass ?? ''}" {...$$restProps}>
  <slot />
</button>
```

This version of the component has an internal click handler that will run along with the forwarded `click` event.

```html
<script>
    function internalFunction() {
        console.log("Button Clicked!");
    }
</script>

<button forward:on on:click={internalFunction}" {...$$restProps}>
  <slot />
</button>
```

## Forwarding actions and transitions

The `forward` directive can be similarly used to forward actions and transitions using the native syntax:

```html
<button forward:use forward:transition>
  <slot />
</button>
```
  
Using the above component in parent context would look like this:

```html
<Button use:myAction>I'm a button with an action!</Button>
```

Similarly to the `on` forwarding, internal actions may be used along with the `forward` directive:
```
<button forward:use use:internalAction>
  <slot />
</button>
```

Note: A transition directive cannot be both forwarded and used internally, since [DOM elements can only have one transition](https://svelte.dev/repl/08236eb2bbc6474bad75d1aee43b2628).

## Forwarding Style Directives

The newly introduced [style](https://svelte.dev/docs#template-syntax-element-directives-style-property) directive can also be forwarded.

```html
<button forward:style>
  <slot />
</button>
```

Parent:
```html
<script>
  import Button from "./Button.svelte";

  let color = "red";
</script>

<Button style:color>I'm a red button!</Button>
```

### Implementation

The `forward` directive's syntax would take in the name of the directive to be forwarded. For example, to forward all events using the `on` directive, the corresponding directive would be `forward:on`.

## How we teach this

The directive should be taught as a method of using directives such as `use` or `transition` through the parent context of a component. It should be taught as a relatively advanced feature towards the end of the tutorial, mainly intended for package authors. It wouldn't majorly affect other documentation topics, although it might be worth noting on the [Event Forwarding](https://svelte.dev/tutorial/event-forwarding) topic of the tutorial.

## Drawbacks

- There are some directives that could not be forwarded, most notably `bind`.
- You cannot combine `forward:transition` and `transition:fn` on the same element, since *[An element can only have one 'transition' directive](https://svelte.dev/repl/08236eb2bbc6474bad75d1aee43b2628)*.

## Alternatives

- Keep things the way they are.
- Use the `on:*` or the `bubble` directive proposed in [#4599](https://github.com/sveltejs/svelte/pull/4599) could be used as a solution for the event problem. Component authors could continue using nonstandard syntax for delegating other events such as `transition`, `use`, or `animation`.

## Unresolved questions

- Should the `bind` directive be handled here at all?
- Should there be a syntax to exclude certain types of a directive? e.g. *forward all events except ones with the name "keydown"*.
