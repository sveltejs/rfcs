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
    class="button {customClass || ''}"
    {...$$restProps}
>
    <slot />
</button>
```

A [solution](https://github.com/sveltejs/svelte/pull/4599) to this event issue using the `on:*` directive has been proposed. This syntax doesn't come without issues either, as `*` is nonstandard syntax in the scope of HTML events, and it doesn't reach the core limitations of component directives.

Another common problem that comes up is when working with actions. Components are incapable of utilizing the `use` directive, meaning library authors often need to export a prop like so:
```html
<script>
    export let use = () => {};
</script>

<div use:use></div>
```

While this also is effective in solving the issue, it adds further boilerplate and nonstandard syntax that other developers are forced to work with. Similar problems arise with the `transition` and `animation` directives.

## Detailed design

### Forwarding all events to an element

This example uses the same `<Button>` component described earlier.
  
```html
<button forward:on class="button {customClass || ''}" {...$$restProps}>
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

### Implementation

The `forward` directive's syntax would take in the name of the directive to be forwarded. For example, to forward all events using the `on` directive, the corresponding directive would be `forward:on`.

## How we teach this

The directive should be taught as a method of using directives such as `use` or `transition` through the parent context of a component. It should be taught as a relatively advanced feature towards the end of the tutorial, mainly intended for package authors. It wouldn't majorly affect other documentation topics, although it might be worth noting on the [Event Forwarding](https://svelte.dev/tutorial/event-forwarding) topic of the tutorial.

## Drawbacks

- There are some directives that could not be forwarded, most notably `bind` and `class`.

## Alternatives

- Keep things the way they are.
- Use the `on:*` or the `bubble` directive proposed in [#4599](https://github.com/sveltejs/svelte/pull/4599) could be used as a solution for the event problem. Component authors could continue using nonstandard syntax for delegating other events such as `transition`, `use`, or `animation`.

## Unresolved questions

- How should the `bind` directive be implemented in this context?
- How should `forward` interact with existing `on`/`use`/`transition`/etc directives?

## Additional Proposal - `foward` effects and singular event forwarding

This section provides a possible answer to the second unresolved question, as well as a discussion of how `forward` can be integrated into Svelte 4 and onward.

Another common problem that people run into is adding side-effects to forwarded events. Suppose a developer had the aforementioned `<Button>` component that forwarded the `click` event, but wanted to execute their own code along with the parent's click callback. The go-to solution would be event dispatchers:

```html
<script>
    import { createEventDispatcher } from "svelte";
    
    const dispatch = createEventDispatcher();
    
    function handleClick(event) {
        console.log(event.target);
    
        dispatch("click");
    }
</script>

<button on:click={handleClick}>
    <slot />
</button>
```
This solution has underlying issues:
- Event dispatchers lose the properties of `MouseEvent`.
- This is a lot of boilerplate to perform a simple action.

Instead of using dispatchers to add side-effects to events, we can extend the forward event to bubble singular event types as well. This could be done like so:

```html
<script>    
    function handleClick(event) {
        console.log(event.target);
    }
</script>

<button forward:on:click={handleClick}>
    <slot />
</button>
```

This pattern could be similarly applied to actions:

```html

<script>
    function internalAction(node) {
        // do something with `node`
    }
</script>

<button forward:use:internalAction></button>
```

In this example, the `use` directive would be both forwarded to parent context as well as `internalAction` being applied. A possible pitfall would be how the animation API would play into this, as it's unclear how the compiler would deal with multiple animations.

## Potential Changes with Svelte 4

It has been further proposed to completely replace the typical event forwarding syntax in the past for Svelte 4. This additional proposal could also cover that:

```html
<button on:click></button>
```

would be changed to

```html
<button forward:on:click></button>
```

This is a breaking change, and likely out-of-scope.
