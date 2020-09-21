- Start Date: 2020-09-18
- RFC PR:
- Svelte Issue:

# Strongly Typed Svelte Component Interface

## Summary

Provide a Svelte component type definition which library authors can use to provide a strongly typed Svelte component which the language server understands. Inside a `.d.ts` file:

```ts
import { SvelteComponent } from "svelte";

export class MyComponent extends SvelteComponent<{...prop definition}, {...events definition}, {...slots definition}> {}
```

## Motivation

Currently there is no official and easy way to provide type definitions. Devs who want to write type definitions for libraries which don't have those, or library authors who want to provide them, have no clear path on how to write these. It's also confusing to them that the current `SvelteComponent` definition does not work for this.

## Detailed design

### Usage would look like this:

Inside a `d.ts` file:

```ts
import { SvelteComponent } from 'svelte';

declare module "myModule" {
  export class MyComponent extends SvelteComponent<
    { propA: boolean },
    { someEvent: CustomEvent<string>; click: MouseEvent },
    { default: { aSlot: number } }
  > {}
}
```

This type definition would result in this being correct usage of the component:

```
<script>
  import { MyComponent } from "myModule";
</script>

<MyComponent propA={true} on:someEvent={e => e.detail.toUpperCase()} let:aSlot={aNumber} />
```

The props/events/slots are typed and other props/events/slots would throw type errors.

### Implementation:

Inside `language-tools`, we already have such a type definition which is used internally and looks like this:

```ts
class Svelte2TsxComponent<
  Props extends {} = any,
  Events extends {} = any,
  Slots extends {} = any
> {
  // The following three exist for type checking capabilities only
  // and do not exist at runtime:
  $$prop_def: Props;
  $$events_def: Events;
  $$slot_def: Slots;

  constructor(options: {
    // ...
    props?: Props & Record<string, any>; // <-- typed as Props, Record<string, any> for the $$restProps possibility
  });

  $on<K extends keyof Events>(
    event: K,
    handler: (e: Events[K]) => void
  ): () => void; // <-- typed
  $set(props: Partial<Props> & Record<string, any>): void; // <-- typed, Record<string, any> for the $$restProps possibility
  // ...
}
```

The Svelte component definition is enhanced with typings for Props, Events, Slots. Enhancing the types for `$on`/`$set` and the `props` constructor param is straightforward. The `$$prop_def`/`$$events_def`/`$$slot_def` properties exist purely for type checking: They are used by the language server to provide intellisense. For the intellisense to extract the correct types, they need to be defined on the component instance. Furthermore, `$$prop_def` is a must because we transform HTMLx to JSX/TSX for the intellisense, and for that we need to provide a property for input props (`$$prop_def`).

Obviously it's not ideal to have type-only-properties on the definition, but with a big enough disclaimer/comment on it, this should be okay in my opinion, especially since the `$$`-prefix is already the "internal, don't use this"-notion for other properties/methods.

Regarding "where to put this": This is up to discussion. Currently I think enhancing `SvelteComponentDev` (which is `SvelteComponent` on the outside) with that definition is the best option. This would not be a breaking change to the existing definition because of the `any` type by default for props/events/slots.

## How we teach this

Docs on the official site as well as in the `language-tools` docs. Using it should feel straightforward for anyone who has worked with TS / library definitions before.

## Drawbacks

The `$$prop_def`/`$$events_def`/`$$slot_def` leak.

## Alternatives

We need to provide these type definitions for a clear path forward, so the only alternatives are _where_ to put them.

- One would be a dedicated package. But I think this feels odd/inconsistent from a developer perspective.
- Another alternative would be to provide this as part of `"svelte"`, but as its own class definition (name could be `SvelteComponentDefinition`). Feels slightly less odd/inconsistent than the first option.

## Unresolved questions

- Anything that can be fine-tuned on the proposed class type definition?
- Anything that I forgot that is no longer possible/throws a type error when the types are more strict?
