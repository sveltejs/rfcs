- Start Date: 2022-04-11
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Conditional Slots

## Summary

Conditionally pass in slot fragments into component

```svelte
<Component>
  {#if condition}
    <svelte:fragment slot="foo">
      something
    </svelte:fragment>
  {/if}
</Component>
```

## Motivation

When we have slot fallbacks, it is determined based on if there are any slots are passed in, to conditionally pass in any slot, you would have to write:

```svelte
{#if condition}
  <Component>
    <svelte:fragment slot="foo">Something</svelte:fragment>
  </Component>
{:else}
  <Component />
{/if}
```

However, this will recreate the `Component` when the condition changes.

It would be helpful, if we can conditionally provide a slots based on a condition

```svelte
<Component>
  {#if condition}
    <svelte:fragment slot="foo">Something</svelte:fragment>
  {/if}
</Component>
```

## Detailed design

### Technical Background

`$$slots` and the slot fallbacks is determined determined based on whether if there are any slots are passed in, rather than whether the slot passed in has no content:

```svelte
<!-- Component.svelte -->
<div>
  <slot name="foo">Fallback</slot>
</div>

$$slots: {$$slots}

<!-- App.svelte -->
<Component>
  <svelte:fragment slot="foo">
    Something
  </svelte:fragment>
</Component>
<!-- <div>Something</div>$$slots: {"foo": true} -->

<Component>
  <svelte:fragment slot="foo"></svelte:fragment>
</Component>
<!-- <div></div>$$slots: {"foo": true} -->

<Component>
  <svelte:fragment slot="foo">{#if false}Something{/if}</svelte:fragment>
</Component>
<!-- <div></div>$$slots: {"foo": true} -->

<Component>
  <svelte:fragment slot="foo">{#each [] as _}Something{/each}</svelte:fragment>
</Component>
<!-- <div></div>$$slots: {"foo": true} -->

<Component />
<!-- <div>Fallback</div>$$slots: {} -->
```

However it is not trivial to determine whether there's any "content" rendered.

See the example below, whether the content is empty, will depends on the the logic blocks conditions, as well as whether the inner component renders anything, which may depends on the slot passed into the component, slot fallbacks, ...

```svelte
<Component>
  <svelte:fragment slot="foo">
    {#each list as item}
      {#await promise then}
        <AnotherComponent1>
          <div slot="bar" />
        </Another>
      {/await}
    {/each}
    {#if condition}
      <AnotherComponent2 {props} />
    {/if}
  </svelte:fragment>
</Component>
```

An alternative implementation design would be to allow `{#if}` block to wrap around `<svelte:fragment slot="xxx">` or `<element slot="xxx">`:

```svelte
<Component>
  {#if condition_1}
    <svelte:fragment slot="a">...</svelte:fragment>
  {/if}
  {#if condition_2}
    <div slot="b">...</div>
  {/if}
</Component>
```

In this case, we just need `condition_1` or `condition_2` to determine whether the slot will be provided.

#### Wrapping `{#if}` outside or inside of the `<svelte:fragment slot="...">`

```svelte
<Component>
  {#if condition}
    <svelte:fragment slot='foo'>hello</svelte:fragment>
  {/if}
</Component>

<Component>
  <svelte:fragment slot='foo'>
    {#if condition}
      hello
    {/if}
  </svelte:fragment>
</Component>
```

When `condition` is `false`, you get

```html
<div>Fallback</div>
$$slots: {}

<div></div>
$$slots: {"foo": true}
```

We determine whether to render `fallback` only based on whether the `<svelte:fragment slot="xxx">` is provided

#### Allow `<svelte:fragment slot="xxx">` to be direct children of `<Component>` and `{#if}`

```svelte
<Component>
  <!-- Error! -->
  {#each list as item}
    <svelte:fragment slot='xxx' />
  {/each}

  <!-- Error! -->
  {#await promise then}
    <svelte:fragment slot='xxx' />
  {/await}
</Component>
```

#### Only allow `<svelte:fragment slot="xxx">` within `{#if}` if the `{#if}` is a direct children of `<Component />`:

```svelte
<Component>
  {#if condition}
    <svelte:fragment slot='xxx' />
  {/if}

  <!-- Error! -->
  <div>
    {#if condition}
      <svelte:fragment slot='xxx' />
    {/if}
  </div>
</Component>
```

#### Do not mix between `<svelte:fragment slot="...">` and other elements

If you have

```svelte
<Component>
  {#if condition}
    <div />
  {/if}
</Component>
```

The `{#if}` block and the `<div />` goes into the content of the default slot

We do not allow mixing content of default slot with a conditional slots

```svelte
<Component>
  {#if condition}
    <svelte:fragment slot="foo"></svelte:fragment>
    <!-- Error: disallow elements besides conditional slots -->
    <div />
  {/if}
</Component>
```

#### Do not allow same slot name provided multiple times across different `{#if}` block

This is to follow the rules laid down in [2. Disallow more than 1 named slot of the same name](https://github.com/sveltejs/rfcs/blob/4efcda208abe007e6a786c18fd38377e25707589/text/0000-slot-attribute.md#2-disallow-more-than-1-named-slot-of-the-same-name), to prevent bugs related to `let:` directives.

If the same slot name is used in different branches of the `{#if}` block, we know that they are going to mutually exclusive, only 1 slot would appear in any 1 time.

```svelte
<!-- Example 1. -->
<Component>
  {#if condition}
    <svelte:fragment slot="foo"></svelte:fragment>
    <!-- Error: duplicated slot name -->
    <svelte:fragment slot="foo"></svelte:fragment>
  {/if}
</Component>

<!-- Example 2. -->
<Component>
  <!-- OK 👌 -->
  {#if condition}
    <svelte:fragment slot="foo">xxx</svelte:fragment>
    <svelte:fragment slot="bar">zzz</svelte:fragment>
  {:else}
    <svelte:fragment slot="foo">yyy</svelte:fragment>
  {/if}
</Component>

<!-- Example 3. -->
<Component>
  <!-- OK 👌 -->
  {#if condition}
    <svelte:fragment slot="foo">xxx</svelte:fragment>
  {:else if condiiton_2}
    <svelte:fragment slot="foo">yyy</svelte:fragment>
  {:else}
    <svelte:fragment slot="bar">zzz</svelte:fragment>
  {/if}
</Component>

<!-- Example 4. -->
<Component>
  {#if condition}
    <svelte:fragment slot="foo">xxx</svelte:fragment>
  {/if}
  <!-- Error: duplicated slot name -->
  {#if condition_2}
    <svelte:fragment slot="foo">yyy</svelte:fragment>
  {/if}
</Component>

<!-- Example 5. -->
<Component>
  {#if condition}
    <svelte:fragment slot="foo"></svelte:fragment>
  {/if}
  <!-- Error: duplicated slot name -->
  <svelte:fragment slot="foo"></svelte:fragment>
</Component>
```

A preferred way of writing for Example 4. would be

```svelte
<Component>
  {#if condition}
    <svelte:fragment slot="foo">xxx</svelte:fragment>
  {:else if condition_2}
    <svelte:fragment slot="foo">yyy</svelte:fragment>
  {/if}
</Component>
```

#### Conditional slot for default slot

You'll have to use `<svelte:fragment>` when you want to have conditional slot for default slot

```svelte
<Component>
  {#if condition}
    <svelte:fragment slot="default">XXX</svelte:fragment>
  {/if}
</Component>

<Component>
  {#if condition}
    XXX
  {/if}
</Component>
```

when `condition` is false, will render

```html
<div>Fallback</div>
$$slots: {}
<div></div>
$$slots: {"default": true}
```

### Implementation

Slots are provided through a special props called `$$slots`.

We can conditionally 

```diff
 // index.svelte.js
 p(ctx, [dirty]) {
   const component_changes = {};
+  if (dirty & /*condition*/ 1) {
+    if (ctx[0]) {
+      component_changes.$$slots = {
+        foo: [create_foo_slot],
+      };
+    } else {
+      component_changes.$$slots = {
+        foo: [create_foo_slot],
+      };
+    }
+  }
   component.$set(component_changes);
}
```

Since the component has no idea if the slots will be passed in conditionally, **additional code** will be added to allow handling if the slots have changed

```diff
 // Component.svelte.js
 p(ctx, [dirty]) {
+  if (!current || dirty & /*$$slots*/ 4) {
+    if (/*$$slots*/ ctx[2].foo) {
+      if (foo_slot_template !== (foo_slot_template = ctx[2].foo)) {
+        // destroy previous slot
+        foo_slot.d();
+        // create a new slot
+        foo_slot = create_slot(
+          foo_slot_template,
+          ctx,
+          /*$$scope*/ ctx[1],
+          get_foo_slot_context
+        );
+        foo_slot.c();
+        foo_slot.m(t0.parentNode, t0);
+      }
+    } else {
+      foo_slot_template = ctx[2].foo;
+      foo_slot.d();
+      foo_slot = null;
+    }
+  }
 }
```

## How we teach this

It is important to teach how `$$slots` are considered passed in to the component, and differentiate between

```svelte
<Component>
  {#if condition}
    <svelte:fragment slot="xxx">...</svelte:fragment>
  {/if}
</Component>
```

and

```svelte
<Component>
  <svelte:fragment slot="xxx">
    {#if condition}...{/if}
  </svelte:fragment>
</Component>
```

## Drawbacks

This will introduce additional code size when using `<slot/>` element within a Svelte component, as we do not know whether the slot will be passed in conditionally or will stay static throughout.

## Alternatives

-

## Unresolved questions

-
