- Start Date: 2018-11-05
- RFC PR:
- Svelte Issue:

### THINGS TO WATCH:

https://www.youtube.com/watch?v=lK3OiJvwgSc
https://github.com/GoogleChromeLabs/houdini-samples/blob/master/layout-worklet/masonry/index.html
http://ishoudinireadyyet.com/
https://adamwathan.me/css-utility-classes-and-separation-of-concerns/


### WIP from the mouth of Rich:

* modularity. we need a way to import CSS that the compiler understands, and we need a way to compose styles a la modular-css
* optimisation. at present, the only form of optimisation is (incomplete) minification
* as you put the RFC together for the dynamic parts, I'd urge you to think about implementation approaches and how they affect the design. For example, if I have a class like .foo and some markup like <div class={passedInFromOutside}> the compiler doesn't know that class="foo" could be the result, so it becomes incredibly hard to know that it needs to apply particular dynamic styles to that element
* (if the implementation calls for CSS variables, then we should probably just use CSS variables in dynamic style tags, and be explicit about the whole thing)
* I'd also suggest studying how theming is applied to custom elements with shadow DOM (I'm not sure what the current state of the art is, I only know that shadow-piercing selectors were deprecated). This is for two reasons: 1) they will have already discussed and anticipated a lot of the problems we would likely face, and 2) it makes it more likely that we're able to reuse the solution when compiling to custom elements(edited)
* For the optimisation piece, it's worth looking at OptiCSS and thinking about that work could apply to ours (Svelte could even use OptiCSS directly, perhaps). That will likely have ramifications for the compiler API (since it needs to think beyond single components), so it touches on the question of whole-app optimisation
* CSS Shadow Parts https://drafts.csswg.org/css-shadow-parts/

# CSS-in-HTML

## Summary

This is a larger RFC focused on solving CSS scoping on both a stand-alone component and global level. This RFC also proposes an approach to reactive CSS as well as better tooling for incorporating 3rd party processors and compilers.

## Motivation

Svelte developers generally acknowledge that there needs to be a better answer for CSS both in components as well as a larger ecosystem consideration. There are varities of opinions about best methodologies for CSS. This proposal sidesteps opinions as much as possible, allowing developers to work with whatever framework or method best suits their purposes, yet cherrypicking the best design considerations.

One debate often boils down to scoping considerations. Currently, with Svelte 2, the inclusion of any CSS within a style tag is considered 100% scoped. Consider:

```html
<!-- main component -->
<div>
	<CascadingDiv/>
</div>
<style>
	div { color: goldenrod; }
</style>
<script>
export default {
	components: { CascadingDiv: './cascasing-div.html' }
};
</script>
```

```html
<!-- component: cascasing-div.html <CascadingDiv/> -->
<div>This is not goldenrod. Styles do not cascade!</div>
```

The only way out of this is to wrap your selector(s) in `:global()`. This, however, gets to be fairly verbose when you want to wrap all selectors on the page.

```html
<style>
:global(.foo) { text-align: center; }
:global(.foo span) { font-size: 80%; }
:global(.foo em) { color: pink; }
.bar { text-align: right; }
<!-- unscoped: `.foo *` -->
<!-- scoped: `.bar` -->
</style>
```

Arguably, the appeal of single file components is encapsulating all component-relatives into the single file. But the above also has the sometimes unwanted effect of cascading everywhere. This presents the complex challenge of controlling scope to a single component, a set of embedded components, or global.

It would be nice if there was a simple mechanism for controlling scope more easily at a component level, making JS variables behave dynamically in CSS and HTML, as well as passing style scopes from component to embedded component if necessary.

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

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

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?