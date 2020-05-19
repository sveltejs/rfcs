- Start Date: 2020/05/19
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Allow applying CSS to child components from parent

## Summary

I think https://github.com/sveltejs/rfcs/pull/13#issue-340110972 is a good addition, and I support it, but it doesn't address all issues

I think Svelte is missing an important feature that other frameworks like Vue have. We need a way to easily apply CSS to a child component from the parent without having to use the verbose selector :global(childSelector) {} inside the CSS

We should be able to apply CSS via classes to child components from the parent element. Many believe that this is a missing feature that should be in svelte.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

Being able to apply CSS to generic child components in Svelte would be a very convenient feature that will make designing websites easier for developers. This is clearly a feature that a large amount of developers would like, as there have been a number of issues raised regarding this:

### Related issues:

- https://github.com/sveltejs/svelte/issues/901
- https://github.com/sveltejs/svelte/issues/4661
- https://github.com/sveltejs/rfcs/pull/13
- https://github.com/sveltejs/svelte/issues/4281
- https://github.com/sveltejs/svelte/pull/2888
- https://github.com/sveltejs/svelte/issues/4443
- https://github.com/sveltejs/svelte/issues/2762
- https://github.com/sveltejs/svelte/issues/1450
- https://github.com/sveltejs/rfcs/blob/style-properties/text/0000-style-properties.md
- https://github.com/sveltejs/svelte/issues/4843

## Detailed design
imagine `<Link/>` is a component that renders an `<a>` element and has some scripts for routing (eg: `event.preventDefault()`)

### Solution 1:
Add an attribute such as **descendants** or something to the `<style>` element to let svelte know that these styles should apply to child components too
```svelte
<Link href="/about">About Us</Link>
<style descendants>
// notice the descendants attributes (similar to how Vue has scoped)
// this would signal to svelte to pass the component's scope ID to children
a {
  color: red;
}
</style>
```
### Solution 2:
Allow Child components to be given some sort of attribute that tells svelte to pass the scope to the child component, eg:
```svelte
<Link:scoped/>
or
<Link scoped/>
<style>
a {
  color: red;
}
</style>
```
### Solution 3:
Allow targeting of components within the parent CSS. eg:
```svelte
<style>

Link {
  color: red;
}

// or
:component(Link) {
  color: red;
}

</style>
```

### Alternatives considered
I have considered taking the `selector :global(childSelector){}` but it is far too verbose, especially if you have something like a `<Link>` component for JavaScript routing, and it might be found in your nav, sidebar, content, headings, footer (with each instance styled differently)

Not to mention that this only works if you wrap the component in something (selector) otherwise it would apply the global everywhere, eg:
```svelte
<div>
   <Link href="/about">About Us</Link>
</div>
<style>
	div :global(a) {
		color: red;
	}
</style>
```

I would like to do something like this:
```svelte
<Link href="/about">About Us</Link>
<style descendants>
a {
 color: red;
}
</style>
```

**I am willing to contribute adding these features if supported**

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

### Solution 1:
would mean that we may need to add support for multiple style tags in Svelte. I don't think that this is a drawback, but rather a bonus that some other people have been asking for.
We could also have a style attribute for global (eg: `<style global>`)

> Why should we *not* do this? Please consider the impact on teaching Svelte,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.
Will add notes here once more feedback has been provided

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

### Why `:global` is inconvenient:
in here, if I want to target a specific element within a slot of a child component, it's not a great or easy thing to do when you're using a preprocessor

```stylus
div :global(a){
   h1 {
      color: red;  // this will not work
   }
   h2 {
       color: blue; // this will not work
   }
}
```
instead you must do something like

```stylus
div :global(a h1){
   color: red;
}

div :global(a h2){
   color: blue;
}
```
which I feel is verbose, and **additionally, you must wrap your child component in an element in order to target it (`div` in this case)**

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.
VueJS allows you to apply CSS to child components

## Unresolved questions

Please give feedback on which solution you would prefer, and why. Also provide criticisms of various solutions if applicable.

## Other notes
I will provide some thoughts as to why sveltejs/rfcs#13 is incomplete too. Firstly, I'd like to say that this is a **useful feature that has been added**, but doesn't address the fact of targeting sub-elements

```svelte
<ChildComponent
  --border="3px solid blue"
  --borderRadius="10px"
  --placeholderColor="blue"
></ChildComponent>
```
Some issues with this:
1. It's verbose, if you have to do this outside of an {#each} for multiple items it's not great.
2. You can't target child elements of the child component effectively
3. Writing CSS will feel more natural to users and having CSS in HTML feels hacky

I totally understand the reason for using CSS variables, and I think the addition to svelte is great, but it doesn't address the other concerns
