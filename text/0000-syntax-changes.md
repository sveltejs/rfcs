- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Significant changes to syntax for V4

As a preface: When I set out to make this RFC, it accidentally ended up being about removing a lot of mustache tags. That wasn't my intention though - it was to resolve issues I had with Svelte, which I go into detail about below.

***

## Summary

This, intentionally controversial, RFC proposes several breaking changes to the current Svelte syntax. These changes are to improve the terseness and readability of Svelte templates, improve intuition for newcomers, and to provoke dicussion about the weaknesses of Svelte's current syntax.

The proposed changes involve mustache _blocks_ (e.g. `{#if something} {/if}`), attribute values, and element directives that accept functions (e.g. `use:fn`). Please see the implementation on the specific syntax this RFC proposes, as it won't exactly fit in a short summary.

## Motivation

This RFC has a few principal assertions:
- Svelte is already incompatible with HTML tooling and should be treated as such
- Svelte needs to adapt and make potentially significant changes in order to stay as the 'next big thing' and not be overtaken by something else which filled the gap Svelte wouldn't
- Svelte must not be afraid to break things, as long as it is done in a controlled manner

Thus, these changes. Much of this is deeply personal - I've felt a certain 'friction' with Svelte and some of its syntax. I believe while how it handles script and styling is nothing but pure excellence, how it handles HTML is lacking. I often get this 'uneasy' feeling when I write templates in Svelte - where what I'm writing feels cobbled together by duct tape, and I feel that this is due to the syntax. It feels like it was jammed into HTML - because it was intended to barely squeeze through an HTML parser.

This appeasement of HTML parsers I don't think has worked anyways. As a personal experience, I wrote the Svelte VSCode TextMate grammar, which is for syntax highlighting. It was going to use the HTML TextMate grammar by wrapping around it - but this didn't work for technical reasons, but now I am glad that it didn't happen.

Why? Well, the TextMate HTML grammar already couldn't parse a Svelte file correctly - instead the way it was being used was to isolate known-safe HTML tags and only then parse them with the HTML grammar. But even then, I had to inject custom attribute handling, especially with interpolation, to make sure that worked too. The _HTML tooling that existed wouldn't have worked for Svelte, unless we just made it work ONLY for Svelte_.

You may say this is just the HTML TextMate grammar - but look at the current developer experience for Svelte. It has its own ESLint plugin, Prettier plugin, VSCode extension, language server, and more. HTML formatters don't understand Svelte conditional blocks. HTML linters won't understand them either. Who actually wants to develop anything with Svelte using inadequate and malfunctioning HTML tooling?

And with that - the most controversial facet of motivation for these changes. If we're not HTML, we can roll some of our own, 'spicier' syntax. And we should! HTML is an inherently verbose language and isn't very good at representing 'code-like' things, like conditional logic.

## Proposed Changes

### Anything using curly brackets must be _self-contained._
Svelte currently uses a syntax inherited from previously popular mustache-style template engines. This syntax can be confusing and/or problematic - constructs such as `{#if something} {/if}` break what a newcomer may expect from curly brackets. Why? Well, in JS, `${}` is self-contained. Nothing inside of those brackets ever has any affect on anything outside of it, unless you're doing probably bad things like reassigning variables.

So, in Svelte, when you see `{}`, you may assume that's quite literally the same thing as in a JS template, and it usually is! However, using 'special' forms of it like `{#if}` break this. They also cause issues with tooling, as telling the difference between interpolation and curly brackets requires that you look inside of them first.

So what do we replace 'special' curly syntax with? This RFC provides no direct recommendation for what syntax to use, because that's an extremely difficult question to answer. This is quite literally, a request for comments, and there should be a likely heated discussion on what syntax would be best.

But in the spirit of at least suggesting something, here is some ideas to think about:
- `<if something> <slot/> </if>` or `<if{something}> <slot/> </if>` (special tags)
- `$if (something) { <slot/> }` (JS-like syntax) (also my favorite)
- `#[if (something) <slot/>]` (a wrapping-type syntax)

All of these suggestions have quirks that would need to be worked out, of course.

One very personal take on this is that any syntax that would be accepted should avoid appearing 'HTML-like', e.g. `<each value=item in=list>`. I think this is just way too verbose and also 'hides the JS' when it should either literally be JS or appear very similar to JS.
	
### Attributes _implicitly assume_ Svelte's context for their values.

What does this mean? Well basically, this:
```html
<!-- current -->
<details bind:this={details} class=card-dark {...$$restProps}> </details>
<!-- proposed -->
<details bind:this=details class='card-dark' {...$$restProps}> </details>
```
Note that in this example you could possibly have `...$$restProps` with no curlies, but that should maybe not be allowed due to `{attributeWithSameNameAsVariable}` not really being practical without using curlies.

Why? Well, _everything_ in HTML is a string. You will not have native attributes that aren't strings. If you think about it, any attributes like `class='card-dark'` could just be represented like `class={'card-dark'}` in Svelte. So, just drop the curly brackets. They're not needed unless you have a space somewhere. Other than the major breaking change of breaking unquoted strings, which is probably bad practice anyways, this would be a change that reduces verbosity and allows for cleaner looking templates.

This change would also get rid of some confusing scenarios, such as a component accepting a number as a property. Currently, you must do `number={1}`, but with this you can just do `number=1`.

Finally, this change does bleed into the previous one a bit, at least conceptually. Syntax like `bind:this={details}` 'breaks-out' of the curly brackets technically. However, using curly brackets to block-out expressions in attributes I think is so heavily in-grained at this point there isn't much of a reason to change it. The only alternative syntax I can think of would be to use parenthesis instead, which is I suppose more reasonable than curly brackets but... it is a major change giving very little in return.
	
### Element directives that accept a function use a function-like syntax.
This is to make what these directives are actually doing much clearer and to allow for richer function usage in elements. It looks like this:
```html
<!-- current -->
<div use:onSwipe={{ dir: 'left', callback: swipeHandler}} in:fade={{ duration: 100 }} />
<!-- proposed, with changes made to the `onSwipe` function -->
<div use:onSwipe('left', swipeHandler) in:fade({ duration: 100 }) />
```
Note that `onSwipe` can now take multiple arguments.

Why? Well, I mean, look at it! These are functions, they should look like functions! Functions use parenthesis, and so now they use parenthesis. Additionally, this allows a native usage of function arguments, rather than restricting them into a single options object argument. This technically doesn't break any existing actions either, which is a bonus.

## How we teach this

These changes would require some reworking of Svelte documentation, along with a whole new major version.

However, these changes aren't really all that hard to actually grasp. I think teaching this syntax would actually be easier than how the current syntax. I'm not sure what else to say here - none of this is technically 'new'.

## Drawbacks

The obvious reason not to do this is because it breaks absolutely everything. It requires annoying a lot of people because now they will have to update potentially hundreds of Svelte templates to get new features.

A more focused reason not to do this would be a cautious hesitation to change what is already familiar to Svelte developers, or another would be a pure disagreement with the changes themselves. Perhaps Svelte developers prefer the mustache syntax being omnipresent in Svelte - I don't know - and because of that this is again, a request for comments.

## Alternatives

This RFC is quite literally begging for alternatives or extensions to the current syntax. I don't have any particular alternatives to the actual concept of this proposal however, other than just disregarding it.
