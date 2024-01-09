- Start Date: 2024-01-09
- RFC PR:
- Svelte Issue:

# A way to disable unknown prop warnings

## Summary

Make it possible to disable unknown prop warnings on a per component basis using

```svelte
<svelte:options surpressUnknonwPropWarnings={true}/>
```

## Motivation

> Why are we doing this? What use cases does it support?

> Please provide specific examples. If you say "this would be more flexible" then
> give an example of something that becomes easier. If you say "this would be make
> it easier to do X" then give an example of what that looks like today and what's
> hard about it.

> Don't assume that others recognize the problem is one that needs to be solved
> Is there some concrete issue you cannot accomplish without this?
> What does it look like to accomplish some set of goals today and how could
> that be improved?
> Are there any workarounds that are necessary today?
> Are there open issues on Github where people would be helped by this?
> Will the change have performance impacts? Can you quantify them?

> Please focus on explaining the motivation so that if this RFC is not accepted,
> the motivation could be used to develop alternative solutions. In other words,
> enumerate the constraints you are trying to solve without coupling them too
> closely to the solution you have in mind.

When a parent component renders a child component with props that are not present on the child component

```svelte
// Parent.svelte
<script>
import Child from './Child.svelte'
</script>

<Child foo={'foo'} bar={'bar'}>
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

However, it poses issues for libraries that allow users to provide components that will be mounted by a library.
Take for example our use case at [svelte flow](https://svelteflow.dev/), a library that renders flowgraphs. With our architecture it is possible to mount any user provided Svelte component as either a node or an edge. Such nodes receive information (which is managed by our library) about their current state in the flow (its position, its label, if its it selected or not ...) over a number of props. Depending on the use case it is not mandotory to implment all of these props - some are only needed in special situations.

So in case we want to render 10 custom node components, where the component only uses 5 of the 10 possible props, the browser console is flooded by 50 unknown prop warnings.

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

Here is [a more extreme example of this.](https://svelteflow.dev/learn/guides/custom-nodes#suppress-unknown-prop-warnings)

## Detailed design

### Technical Background

> There are a lot of ways Svelte is used. It's hosted on different platforms;
> integrated with different libraries; built with different bundlers, etc. No one
> person knows everything about all the ways Svelte is used. What does someone who
> knows about Svelte but hasn't necessarily used anything outside of it need to
> know? Are there docs you can share?

> How do different libraries or frameworks implement this feature? We can take
> design inspiration from others who have done this well and improve upon the
> designs or make them better fit Svelte.

`<svelte:options />` already exists as way to communicate certain per component configurations to the compiler.

### Implementation

> Explain the design in enough detail for somebody familiar with the framework to
> understand, and for somebody familiar with the implementation to implement. Any
> new terminology should be defined here.

> Explain not just the final design, but also how you arrived at it. What
> constraints did you face? Are there corner cases you've come up with solutions for?

> Explain how your design fits into the larger picture. Are the other open problems
> in this area you're familiar with? How does this design fit with potential
> solutions for those issues?

> Connect your design to the motivations you listed above. When describing a part of
> the design, it can be useful to share an example of what it would look like to
> utilize the implementation as solution to the problem.

The idea of this RFC is to have the possibility to set this at the top of a component:

```svelte
<svelte:options surpressUnknownPropWarnings={true} />
```

which surpresses all unknown prop warnings.
As already discussed, having unknown prop warnings as a default is sensible, disabling it globally would defeat its purpose, thus having a way to opt-in on specific components that are expected to have unknown props seems like a good way forward.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
> idea best presented? As a continuation of existing Svelte patterns, or as a
> wholly new one?

> Would the acceptance of this proposal mean the Svelte guides must be
> re-organized or altered? Does it change how Svelte is taught to new users
> at any level?

> How should this feature be introduced and taught to existing Svelte
> users?

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Svelte,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the
> same domain have solved this problem differently.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD?
