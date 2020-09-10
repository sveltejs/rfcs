- Start Date: 2020-09-09
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Inline components

## Summary

A new `<svelte:template>` element that defines an inline logic-less component.

## Motivation

In some frameworks, a module can contain multiple components, some of which need not be exported (and are therefore effectively private). In Svelte, there is a strict one-component-per-file rule.

This is nice and simple to understand and explain (not to mention implement), but it does have drawbacks. Users sometimes complain that it causes a proliferation of `.svelte` files, many of which contain very simple components that would be better declared inline.

The alternative is to duplicate markup inline:

```svelte
{#each images as image}
  {#if image.link}
    <a href={image.link}>
      <figure>
        <img src={image.src}>
        {#if image.caption}
          <figcaption>{image.caption}</figcaption>
        {/if}
      </figure>
    </a>
  {:else}
    <figure>
      <img src={image.src}>
      {#if image.caption}
        <figcaption>{image.caption}</figcaption>
      {/if}
    </figure>
  {/if}
{/each}
```

Neither is ideal.

## Detailed design

With `<svelte:template>` (🐃), combined with new functionality for `<svelte:component>` in which a string `this` attribute is taken to refer to an inline component, we can avoid both the problems articulated above:

```svelte
<svelte:template name="image" let:image>
  <figure>
    <img src={image.src}>
    {#if image.caption}
      <figcaption>{image.caption}</figcaption>
    {/if}
  </figure>
</svelte:template>

{#each images as image}
  {#if image.link}
    <a href={image.link}>
      <svelte:component this="image" {image}>
    </a>
  {:else}
    <svelte:component this="image" {image}>
  {/if}
{/each}
```

Together with the RFCs for [local scoped styles](https://github.com/sveltejs/rfcs/blob/local-styles/text/0000-local-styles.md) and [constants in markup](https://github.com/sveltejs/rfcs/blob/markup-constants/text/0000-markup-constants.md), this gives us everything we need to create self-contained logic-less components, even including slotted content:

```svelte
<svelte:template name="tooltip" let:dangerousness>
  {@const danger = dangerousness > 5}

  <div class="tooltip" class:danger>
    <slot></slot>
  </div>

  <style>
    .tooltip {
      background: white;
      padding: 1em;
    }

    .danger {
      color: red;
    }
  </style>
</svelte:template>
```

(At the point at which it *does* make sense to turn this into a separate `Tooltip.svelte` component, the extraction is a completely mechanical process that could even be automated by tooling.)


### No `<script>` in inline components

It's tempting to suggest that inline components should be allowed their own `<script>` blocks:

```svelte
<svelte:template name="tooltip" let:dangerousness>
  <script>
    $: danger = dangerousness > 5;
  </script>

  <div class="tooltip" class:danger>
    <slot></slot>
  </div>

  <style>
    .tooltip {
      background: white;
      padding: 1em;
    }

    .danger {
      color: red;
    }
  </style>
</svelte:template>
```

I think this would be a mistake: lifecycle and scope would get tricky for component authors to reason about, and (not that this should be an overriding consideration) I would expect it to result in a significant increase in implementation complexity.


### Change to `<svelte:component>`

At present, the `this` attribute of a `<svelte:component>` must be either a component constructor or a falsy value. In order to make it work with inline components, we allow `this` to be a string instead. In many cases this could be optimised to be a direct reference to the locally defined constructor, though in the case of a dynamic `this`...

```svelte
<svelte:component this={string_or_constructor}/>
```

...it would be necessary to maintain a mapping of names to constructors instead.

## How we teach this

'Inline component' is an obvious term, though it doesn't quite explain `<svelte:template>`. In other template language contexts, the word 'partial' is sometimes used to mean something similar.

Accepting this proposal would mean adding new documentation, and updating the `<svelte:component>` documentation to accommodate `this="string"`.

## Drawbacks

As ever, the concern is that this increases the amount of stuff to learn, and the amount of stuff to implement and maintain.

In a similar vein to ([#33](https://github.com/sveltejs/rfcs/blob/markup-constants/text/0000-markup-constants.md#drawbacks)), it *is* arguably just something that compensates for the lack of power in the template language relative to JavaScript.

## Alternatives

Instead of a new special element, we could use syntax:

```svelte
{#template "image" { image }}
  <figure>
    <img src={image.src}>
    {#if image.caption}
      <figcaption>{image.caption}</figcaption>
    {/if}
  </figure>
{/template}

{#each images as image}
  {#if image.link}
    <a href={image.link}>
      {@partial "image" {image}}
    </a>
  {:else}
    {@partial "image" {image}}
  {/if}
{/each}
```

I think this is more confusing than the special element proposal, and doesn't make effective use of existing concepts (`let:`, slots, etc).

## Unresolved questions

* Is `<svelte:template name="...">` right? (What if there was a prop called `name`? Should we use `this` instead, or is that confusing?)
* Can templates be scoped, or must their names be unique within the component? Must they be defined at the top level of the component, or can they be defined anywhere (and inherit context and styles that apply to their subtrees)?