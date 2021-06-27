- Start Date: 2021-02-09
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# An Error Boundary Tag / Directive / Component Archetype

## Summary

> Creation of an error boundary which catches errors and allows the system to handle them.

## Motivation

1. After using Sentry in production to handle errors, having specific boundaries for handling errors in different ways is desirable.
2. Catching errors in Sapper is critical and currently infeasible. Errors in Sapper break the router, and a user sees a "frozen" site. I would imagine this is similar for Kit.

It's a bit of a [dealbreaker](https://news.ycombinator.com/item?id=22544837) for Sapper users currently.

## Detailed design

There's actually prior art. I use this in production myself. The source code for such is here:

https://github.com/crownframework/svelte-error-boundary

and a working example here:

https://svelte.dev/repl/9d44bbcf30444cd08cca6b85f07f2e2a?version=3.29.4

My suggestion for alternative syntax is detailed below, but I am proposing a tag which can be added to any component to make it a boundary:

```svelte
<svelte:error handler={myErrorHandler} />
```

This solves a solution such as `on:error` attribute being a breaking change.

### Technical Background

This exists in a variety of frameworks, React has native support, whilst vue currently does not.

The impetus for creating it as a native feature of Svelte is to prevent the "brittle" nature of the way the current third-party solution works.

See:

* [React Error Boundaries](https://reactjs.org/docs/error-boundaries.html)
* [vue-error-boundary](https://www.npmjs.com/package/vue-error-boundary)
* [svelte-error-boundary](https://github.com/crownframework/svelte-error-boundary)

### Implementation

There is actually a [reference implementation](https://github.com/crownframework/svelte-error-boundary)
which has been proven to work in production environments. I would suggest that we go with
this implementation.

The stark advantage to making this part of core is that the current implementation
relies on non-public parts of the Svelte API which can change at any time,
causing it to be brittle.

The pre-existing component uses `onError` for technical reasons, I can suggest two alternatives for handling
this as `onError` doesn't really fit the Svelte language:

1. To allow components to simply implement `on:error` as with existing handlers, however the naming of such would have to be considered, as it is likely that people will have alread created custom events with the name `error` and this would conflict with them.
1. To allow a new svelte tag `<svelte:error handler={myErrorHandler}>`. This tag could be included in any component to make itself a boundary.

The important factor here is allowing a handler function, which can then be wired up to
[https://forum.sentry.io/t/requesting-svelte-support/8477](Sentry) or LogRocket. Sentry have noted
that lack of an error boundary component is a blocker to them having native support for
Svelte.

## How we teach this

* Is API documentation required? Yes.
* Is a tutorial section required? No.

Community support will be available for this, for example, Sentry would likely add a section to their documentation about specific Svelte integration, which they currently can't do due to lack of this component.

## Drawbacks

Implementation drawbacks:

1. `on:error` would be a breaking change.
1. `onError` is not Svelte-esque syntax
1. A directive doesn't feel like a core component

Therefore I propose the tag approach `<svelte:error handler={} />`.

With regards to drawbacks of a tag:

Q. You can't put this tag into third-party components
A. This isn't really a drawback, implementors can either add the tag in their libraries and allow the user to pass in a handler function, or, you can wrap any component you want as an error boundary, with a new component which has the tag.

## Alternatives

I don't have any alternatives right now, other than sticking with the crownframework boundary and being careful during Svelte upgrades, which isn't very scalable.

## Unresolved questions

Merely Syntactical. I don't know if my suggestion around a new tag is feasible/desirable/can solve all use-cases.
