- Start Date: (2021-08-20)
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# `<svelte:this/>`

## Summary

This RFC proposes an idiomatic, generic tag that can be used for features that are related
to the current Svelte component (not DOM).
```svelte
<svelte:this
    bind:this={component}
    bind:props={props}
    bind:dispatch={myDispatcher}
    on:mount={useWindow}
    on:error={logAnalytics}/>
```

I also propose deprecating current global variable "`$$feature`" APIs and
"`import { randomFeature } from "svelte" `" APIs where possible in the next major release. See more below.

## Motivation

Many features in Svelte 3 are currently implemented as "magic" globals (e.g `$$slots`)
or mystery imports from Svelte (e.g `import { createEventDispatcher } from "svelte"`).
This proposal provides a cleaner implementation for Svelte APIs that are tied to the current
component.

It's never obvious where many Svelte APIs should go, and as a result they end up feeling
unidiomatic. For example, the magic global `$$host` is proposed in
https://github.com/sveltejs/svelte/issues/3091 for custom elements, but I suggest `<svelte:this bind:this={myShadowRoot}`
 might be more idiomatic, and less magical.

Declarative APIs would also give the compiler information about which values are actually used.
Rendering logic around `get(+set)Context`, `$$props`, `$$slots`, `$$restProps` etc. and lifecycle
logic checking `beforeUpdate`, `afterUpdate` `onDestroy` etc. can be omitted if a component
doesn't use them.

Future compiler versions could make more aggresive optimizations to increase performance and
reduce bundle size. Even more, a future compiler which can "see" your full app might be
able to give you compile-time errors if you access `context` with an invalid key, while the imperative
`getContext("key")` might not.

## Detailed Design

Only a single top-level `<svelte:this />` is needed.

### Access to component properties or values
`bind:` would be the idiomatic way to access component metadata etc. instead
of "magic globals". This gives the compiler a heads up that a component
value will be used in the current scope, and allows
for arbitrary variable names for values.

Possible candidates:
```svelte
<svelte:this
    bind:this={myShadowRoot}
    bind:getContext|key={myContext}
    bind:setContext|otherKey={myOtherContext}
    bind:props={myProps}
    bind:restProps={undefinedProps}
    bind:slots={slots}
    bind:dispatcher={myDispatcher}/>
```

`<svelte:this bind:this={variable}/>` internally should behave the same way binding to a component externally would behave.

```svelte
<!-- Component.svelte -->
<script>
    let component;

    export function equal(bound) {
        return component === bound;
    }
</script>

<svelte:this bind:this={component}>

<!-- App.svelte -->
<script>
    let bound;
    let equal;

    $: if (bound) {
        equal(bound); // expects true
    }
</script>

<Component bind:this={bound} bind:equal>
```

```svelte
<!-- component.wc.svelte -->
<script>
    let component;

    export function equal(bound) {
        return component === bound;
    }
</script>

<svelte:option tag="my-component"/>
<svelte:this bind:this={component}>

<!-- App.svelte -->
<script>
    let bound;
    let equal;

    $: if (bound) {
        equal(bound); // expects true
    }
</script>

<my-component bind:this={bound} bind:equal/>
```

### Listening to lifecylce events

The `on:` syntax would be the idiomatic way to listen to lifecycle and other component events.

Lifecyle events should be have like other component events. `<svelte:this on:mount>`
for example should support event forwarding.

Possible candidates:
```svelte
<svelte:this
    on:afterupdate={afterUpdate}
    on:beforeUpdate={() => value = false}
    on:mount
    on:destroy
    on:futureRFC={doThing}
    />
```

## How we teach this

Adjust documentation include `<svelte:this/>`.

A blog post outlining the change, what new capabilities it unlocks, and a migration path.

## Drawbacks

- If we add the feature in Svelte 3, then it's yet another API to learn while
the current ones might be "good enough" already.

- I anticipate there may be some
initial confusion for completely new users differentiating between
`<svelte:this/>`, `<svelte:self/>` and  `<svelte:component/>`. Docs solve this.

## Alternatives

Leave things as they are. Prefer `$$feature` and `import { newAPI } from "svelte"`
for future features.

## Unresolved questions

* How does this interact with the script's lifecycle?

    When binding to the DOM, it is common to do this:
    ```svelte
    <script>
        let value = "value";
        let div;
        $: div && useDiv(div)
    </script>

    <div bind:this={div}>{value}</div>
    ```
    This is because the script runs, those values are used to render the DOM,
    then DOM node is bound to the variable. So `let div;` is undefined initially.

    From my suggested use cases, `bind:this` is the only possible DOM reference (the shadowRoot
    in custom elements), and the shadowRoot is independent of `script` values,
    so it can be created before running `script`.

    Can we assign bound values before running `script` to avoid the
    hustle of `$: div && useDiv(div)`? Are there `bind:`able values
    that might depend on `script`?

* This RFC is large-ish, should it be implemented in one go or incrementally?
If incrementally, what goes in the MVP?

* Should we eventually deprecate current APIs? Are there limitations to such a
declarative API relative to current imperative ones?

