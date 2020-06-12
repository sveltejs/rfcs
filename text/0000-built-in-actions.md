- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Built-in Actions

## Summary

This RFC proposes the idea of having a number of common built-in actions much like there
are built-in transitions.

## Motivation

Actions are currently a bit of a mystery to a lot of people but are incredibly powerful.
Having a number of ready-made actions would increase the simpleness of getting started
and adds ease-of-use to beginners of Svelte. 

This is one of those things that blows minds when they first see it, much like the
transitions. It would greatly ease efforts to market Svelte as *the* framework that
people should use.

## Detailed design

Much like the `svelte/transition` package a `svelte/action` package could be created.
You would import them like so: `import { longPress } from 'svelte/action'`.
In here we would put whatever actions that would make sense to put here. These
could be actions such as `use:longpress`, `use:clickOutside`, `use:lazyload` and so on.
Determining what actions should and should not be included needs to be discussed.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

There are no real new concepts added by this RFC but I can see how we would probably want
to add one or two new Tutorials and Examples. The documentation would need to be expanded
a bit to also show off these actions, much like how transitions are described there.

## Drawbacks

The first one is documentaiton. This would probably require the most work in the short-term
but I think it would be a pretty straight-forward thing once the actual actions that are
included are decided.

The other downside is that the amount of code to maintain increases. We would have to update
and keep these actions working but unless the syntax changes radically or the concept of
actions is going away (😡) I think it might be worth it.

## Alternatives

There are probably not any other alternatives to this, apart from not doing it. 


## Unresolved questions

The obvious question here would be to decide what actions should be included. What are some
popular use-cases that would fit well here?
