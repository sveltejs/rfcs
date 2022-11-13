- Start Date: 2022-11-13
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Standardized Render Tag Syntax

## Summary

This RFC is a proposal for enhancing the existing control
flow blocks (`{#if}`, `{#each}`, and `{#await}`) and tags
(`{@html}` and `{@const}`), and add support for custom
blocks and tags. The potential solution will use a standardized
block and tag syntax.

## Motivation

Regarding this RFC, Svelte's existing syntax is customizable,
but it requires the knowledge of how to parse the tag or block,
read its content, and generate the output to tell the final output code how to behave.
For instance, in order to provide `switch-case` syntax to a template, a plugin
([`svelte-switch-case`](https://www.npmjs.com/package/svelte-switch-case))
will need to be used to provide that syntax.

According to `svelte-switch-case`'s documentation, the plugin is added like so:

```js
// svelte.config.js
import switchCase from 'svelte-switch-case';

const config = {
  preprocess: [switchCase()],
};

export default config;
```

Then the plugin can be used in the template:

```svelte
<!-- Component.svelte -->
<script>
  let animal = 'dog';
</script>

<section>
  {#switch animal}
    {:case "cat"}
      <p>meow</p>
    {:case "dog"}
      <p>woof</p>
    {:default}
      <p>oink?</p>
  {/switch}
</section>
```

Plugins *can* be used for such cases; however, there are a few drawbacks to writing and using plugins.
- First of all, there are not many plugins available to provide custom blocks (`svelte-switch-case` may be the only plugin for blocks on GitHub).
- Second, there is the risk of having buggy code in the compiled output or the plugin itself.
- Third, the plugin will most certainly provide extra bloat to the `node_modules` folder.
- Finally, the syntax provided by the plugin in question may not be noticed or properly processed by
  - syntax highlighters,
  - type checkers,
  - code formatters (e.g. `prettier`), and
  - linters
  
  unless other plugins are used.

This RFC could eliminate those issues and much more.
- The syntax is provided by Svelte (see the next section for details).
- No hassle for writing complicated block-parsing plugins.
- No extra bloat added to the `node_modules` folder (except for one caveat).
- The syntax is standardized (see below), which makes it easier for IDEs and linters to parse.


## Detailed design

### Technical Background

Handlebars has a syntax for using ["block tags"](https://handlebarsjs.com/guide/block-helpers.html#basic-blocks) that is very similar to Svelte's blocks.

Here is some Handlebars code:

```handlebars
{{#each names as |name|}}
  <p>Hello, {{name}}!</p>
{{/each}}
```

Here is what the above Handlebars example looks like in Svelte:

```svelte
{#each names as name}
  <p>Hello, {name}!</p>
{/each}
```

The syntactic similarities are very prominent with one exception:
with Handlebars, custom block helpers can be defined while Svelte cannot (yet).
That difference could be taken care of with the implementation below

### Implementation

#### Syntax

On the syntax side, the syntax for the main block would look like this:

```
{#blockName [|param1[, param2[, param3[, ..., paramN]]]|] [ arg1[, arg2[, arg3[, ..., argN]]]] [ key1: value1[, key2: value2[, key3: value3[, keyN: valueN]]]]}
  content
{/blockName}
```

- `blockName`: a JavaScript identifier to identify the block.
- `|param1[, param2[, param3[, ..., paramN]]]|`: a parameter list for the content inside the block.
  The parameters are scoped to content and cannot be used in in the arguments of the block.
- `[ arg1[, arg2[, arg3[, ..., argN]]]]` and `[ key1: value1[, key2: value2[, key3: value3[, keyN: valueN]]]]`: Those are the arguments supplied to the the block; the latter are named arguments in the form of `key: value`.
- `content`: The block's *main* content that begins after a `{#block ...}` and ends before a `{/block}` or a sub-block (`{:subBlock ...}`).

The syntax for sub-blocks (`{:else}`, `{:then}`, `{:catch}`, etc.) is very similar to regular blocks with a few differences:
1. The `#` is replaced with `:`.
2. Parameters used in the main content (before the first `{:subBlock ...}`) cannot be used as the parameter in each sub-block.
3. The content is now between a `{:subBlock}` and a `{/block}` or another `{:subBlock}`.

Finally, the syntax for a tag (e.g. `{@html}`) would look like a block without content to render and no parameters.

As can be seen, the syntax is more regular, which allows it to be parsed by development tools.

#### Under the Hood

As the Svelte compiler is very complex, the only part that will be proposed is how the block would work under the hood.
Blocks are just functions that have the following signature:
```ts
type SvelteBlock = (context: {
  args: () => any[], // from `[ arg1[, arg2[, arg3[, ..., argN]]]]`
  kwargs: () => Record<string, any>, // from `[ key1: value1[, key2: value2[, key3: value3[, keyN: valueN]]]]`
  content : SvelteFragment, // from `|param1[, param2[, param3[, ..., paramN]]]|` and `content`
  subBlocks: {
    name: string, // from `:subBlock`
    args: () => any[],
    kwargs: () => Record<string, any>,
    content : SvelteFragment,
  }[]
}) => SvelteFragment
```

Note that `SvelteFragment` needs to be discussed.

## How we teach this

Teaching how to build custom blocks needs to be discussed (see note above); however the new syntax can be taught with the tutorial and docs.
Here are a few examples:

#### `{#if}`
```
{#if name && admin}
<p>Welcome, {name}. Here is your dashboard:</p>
{:elseif name}
<p>Welcome, {name}!</p>
{:else}
<p><a href="/sign-in">Sign in &rarr;</a></p>
{/if}
```

#### `{#each}`
```
<script>
  // ...
  const trackById = ({id}) => id
</script>
...
<ul>
{#each |{name}, index| in: items, trackBy: trackById}
  <li>{index + 1}: {name}</li>
{/each}
</ul>
```

#### `{#await}`
```
{#await somePromise}
  <p>Loading...</p>
{:then |value|}
  <p>{value}</p>
{:catch |error|}
  <p>Uh-oh: {error}</p>
{/await}
```

#### [New] `{#try}` (error boundaries)
```
{#try}
  <BreakApp/>
{:catch |error|}
  <p>Uh-oh: {error}</p>
{/try}
```

## Drawbacks

1. *This is a **breaking change.***
2. Moving the compiled logic to block functions may be less efficient.
3. Depending on the implementation of `SvelteFragment`, the API may be hard to maintain and understand.

## Alternatives

Besides preprocessors, many other frameworks take care of this differently.

- Angular has a special [micro-syntax for structural directives](https://angular.io/guide/structural-directives#structural-directive-syntax-reference);
  this syntax makes the usage of structural directives at times more like writing JavaScript or TypeScript inside the directive.
  ```html
  <p *ngFor="let name of names">Hello, {{names}}!</p>
  ```
- [Marko](https://markojs.com/) is previewing a new API [built only with HTML tags](https://dev.to/ryansolid/introducing-the-marko-tags-api-preview-37o4).

If neither of these API ideas were considered, Svelte could use components for control flow, but that would be clunky and error-prone.

## Unresolved questions

- How should blocks and `SvelteFragment`s be implemented?
