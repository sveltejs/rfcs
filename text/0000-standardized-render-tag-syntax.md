- Start Date: 2022-11-12
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Standardized Render Tag Syntax

## Summary

This RFC is a proposal for enhancing the existing control
flow blocks (`{#if}`, `{#each}`, and `{#await}`) and tags
(`{@html}` and `{@const}`), and add support for custom
blocks and tags. The potential solution will use a developer
friendly API, and the solution will use a standardized
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
That difference could be taken care of with this RFC.

---

At this point, for some more background, I would like to point out that Angular has a special
[micro-syntax for structural directives](https://angular.io/guide/structural-directives#structural-directive-syntax-reference);
this syntax makes the usage of structural directives at times more like writing JavaScript or TypeScript inside the directive.
Below is what the last two examples would look like in Angular:

```html
<p *ngFor="let name of names">Hello, {{names}}!</p>
```

The implementation below goes into detail about how a block and tag syntax could be implemented.

### Implementation

> Explain the design in enough detail for somebody familiar with the framework to
understand, and for somebody familiar with the implementation to implement. Any
> new terminology should be defined here.

> Explain not just the final design, but also how you arrived at it. What
> constraints did you face? Are there corner cases you've come up with solutions for?

> Explain how your design fits into the larger picture. Are the other open problems
> in this area you're familiar with? How does this design fit with potential
> solutions for those issues?

> Connect your design to the motivations you listed above. When describing a part of
> the design, it can be useful to share an example of what it would look like to
> utilize the implementation as solution to the problem.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Svelte guides must be
re-organized or altered? Does it change how Svelte is taught to new users
at any level?

> How should this feature be introduced and taught to existing Svelte
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Svelte,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the
> same domain have solved this problem differently.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD?
