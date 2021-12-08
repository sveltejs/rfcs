- Start Date: 2020-09-20
- RFC PR: 
- Svelte Issue: https://github.com/sveltejs/language-tools/issues/442

# TypeScript: Typing Props/Events/Slots + Generics

## Summary

Provide possibilites for TypeScript users to strongly type a Svelte component's props/events/slots, including generics. For that, we introduce reserved interfaces named `$$Props`, `$$Events`, `$$Slots`. We also introduce a concept for generics and a `<script>` attribute for marking a component as having no other events besides the ones defined within.

While this is not a change to Svelte's core, it's still something that needs to be specified so intellisense implementers have something to adhere to.

## Motivation

Using TypeScript with Svelte provides a lot of goodness already, but there are some areas which lack support:

- There is no way currently to tell the intellisense that there's only a fixed set of events one can listen to. You can type `createEventDispatcher` but that does still make it possible to listen to other events
- There is no way currently to explicitely type slots
- There is no way currently to use generics
- There is no way currently to make a component implement some specified interface and have this checked by types

## Detailed design

### Typing events

Use case: You want to strictly type events. Listening to anything outside the defined events should throw a type error.

You start with one event which is from your own typed `createEventDispatcher` and one forwarded event.

```html
<script lang="ts">
    import {createEventDispatcher} from "svelte";

    const dispatch = createEventDispatcher<{own: boolean}>();
</script>

<button on:click={() => dispatch('own', true)}>Own</button>
<button on:click>Forwarded</button>
```

Now you want to ensure that listening to anything else than `on:own`/`on:click` throws a type error. For that you use the new `<script>` attribute `strictEvents`:

```html
<script lang="ts" strictEvents>
    import {createEventDispatcher} from "svelte";

    const dispatch = createEventDispatcher<{own: boolean}>();
</script>

<button on:click={() => dispatch('own', true)}>Own</button>
<button on:click>Forwarded</button>
```

Now you add one event which comes from a dispatcher mixin:

```html
<script lang="ts" strictEvents>
    import {mixinDispatch} from "somewhere";
    import {createEventDispatcher} from "svelte";

    const dispatch = createEventDispatcher<{own: boolean}>();
</script>

<button on:click={() => mixinDispatch.mixinEvent('foo')}>Mixin</button>
<button on:click={() => dispatch('own', true)}>Own</button>
<button on:click>Forwarded</button>
```

In this case `strictEvents` will not work anymore because we cannot know that `mixinDispatch` dispatches events. So now you use the `$$Events` interface.

```html
<script lang="ts">
    import {mixinDispatch} from "somewhere";
    import {createEventDispatcher} from "svelte";

    interface $$Events {
        mixinEvent: CustomEvent<string>;
        own: CustomEvent<boolean>;
        click: MouseEvent;
    }

    const dispatch = createEventDispatcher<{own: boolean}>();
</script>

<button on:click={() => mixinDispatch.mixinEvent('foo')}>Mixin</button>
<button on:click={() => dispatch('own', true)}>Own</button>
<button on:click>Forwarded</button>
```

### Typing Slots

This works the same as for typing events.

```html
<script lang="ts">
    interface $$Slots {
        default: { prop: boolean; };
    }
</script>

<slot prop={true}></slot>
```

### Typing Props

This works the same as for typing events. You probably won't use that because it's essentially doing the type work twice.

```html
<script lang="ts">
    interface $$Props {
        prop: boolean;
    }

    export let prop: boolean;
</script>
```

If you define `$$Props`, all possible props need to be part of it. If you use `$$props` or `$$restProps` then that does _not_ widen the type, still only those defined in `$$Props` are allowed.

### Generics

You want to specify some generic connection between props/slots/events. For example you have a component which has an input prop `item`, and an event called `itemChanged`. You want to use this component for arbitrary kinds of item, but you want to make sure that the types for `item` and `itemChanged` are the same. Generics come in handy then. You can read more about them on the [official TypeScript page](https://www.typescriptlang.org/docs/handbook/generics.html).

#### Solution
You use new reserved type called `$$Generic`.

```html
<script lang="ts">
    import {createEventDispatcher} from "svelte";

    type T = $$Generic<boolean>; // extends boolean
    type X = $$Generic; // any
    
    // you can use generics inside the other interfaces
    interface $$Slots {
        default: { aSlot: T }
    }

    export let array1: T[];
    export let item1: T;
    export let array2: X[];
    const dispatch = createEventDispatcher<{arrayItemClick: X}>();
</script>
```

##### Discarded alternative 1
You use a new `<script>` attribute called `generics`. The contents of that attribute have to be valid generic typings.

```html
<script lang="ts" generics="T extends boolean, X">
    import {createEventDispatcher} from "svelte";

    export let array1: T[];
    export let item1: T;
    export let array2: X[];

    const dispatch = createEventDispatcher<{arrayItemClick: X}>();
</script>

...
```

Discarded because it is invalid TypeScript without additional transformations.

##### Discarded alternative 2
You use a new reserved interface called `$$Generics` and do the typing on it, not declaring any properties on it.

```html
<script lang="ts">
    import {createEventDispatcher} from "svelte";
    
    interface $$Generics<T extends boolean, X> {}

    export let array1: T[];
    export let item1: T;
    export let array2: X[];

    const dispatch = createEventDispatcher<{arrayItemClick: X}>();
</script>

...
```

Discarded because it is invalid TypeScript without additional transformations.


### ComponentDef (likely discarded)

If you want to type all at once, because you like to have the definition in one place or want to better define a generic relationship, you can use the `ComponentDef` interface.

```html
<script lang="ts">
    // ...
    interface ComponentDef<T> {
        props: { items: T[]; someOptionalProp?: string; };
        events: { itemClick: CustomEvent<T>; };
        slots: { default: { item: T; }; };
    }
</script>

...
```

This is likely not a good idea because you then can achieve typing a component in multiple ways, which introduces maintenance overhead for implementations.

#### Discarded ComponentDef alternative: Namespace

As an alternative to the `ComponentDef` interface, one could use a namespace and put the interfaces inside it. That would make refactoring easier if you for example start of with typing only the events but want to add more typings to slots later on. This would come at the cost of uncanny-valley-stuff for defining the generics.

```html
<script lang="ts">
    // ...
    declare namespace Component {
      interface Generics<T> {}
    
      interface Events {
        itemClick: CustomEvent<T>; // use generic T here which will be the one defined in Generics
      }
    
      interface Props {
        itemClick: CustomEvent<T>;
        someOptionalProp?: string;
      }
    
      interface Slots {
        default: { item: T }
      }
    }
</script>

...
```

This is discarded because it provides no real benefit over the three-seperate-interfaces-solution.

### Summary

As you can see, there would be several options to achieve the same. You can use `ComponentDef` to type all at once, or you can mix and match the other possibilities to only type part of it. The drawback is that there is more than one way to achieve the same goal. But only having `ComponentDef` may be too much typing overhead of you only want to specifically type parts of the component. In general, props, slots and their types are already inferable quite nicely at this point. Only generics and events are where you really would need this.

### Implementation hurdles

We would need to make sure that we can provide some meaningful errors if the definition and the actual types don't match. So if someone types `$$Slots` as `{foo: boolean;}` but does `<slot foo={'aString'}></slot>`, we must highlight that. I have not looked closely into how this can be achieved yet because I want to first have agreement on the API.

## How we teach this

For users: Enhance docs. For intellisense devs: A more formal specification outlining the details.

## Drawbacks

- This will only work for TS users
- Uncanny-valley-stuff for generics
- Reserved interface names could collide with existing ones, but I think that's rare. It's also only a breaking change for the language-tools because it does not affect the core of Svelte
- If we implement `SvelteComponentDef`, then there are multiple ways to achieve the same goal (interface/generics combinations)

## Alternatives

- Don't do anything and say "well, there are some limits". VueJS for example also cannot deal with generics as far as I know.
- Only provide parts of this solution: `strictEvents` and generics, and from the interfaces only `ComponentDef`, and tell people "if you want to type it, type it all".
- Interface name alternatives: `Props` / `Events` / `Slots` / `Generic`. More likely that they clash with existing definitions. `ComponentProps` / ... - too verbose.

## Unresolved questions

- Interface wording ok?
- Attribute wording ok?
