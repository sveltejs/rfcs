- Start Date: 2018-12-18
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Error Boundaries

## Summary

This RFC proposes a `{#try}` block as an error handling mechanism to catch and handle errors within svelte applications. Errors that occur during the rendering lifecycle in svelte can be hard to catch and will cause the entire app to crash or reach an unstable state. The addition of a `{#try}` block will allow errors to be handled in a convenient and idiomatic manner.

## Motivation

Properly handling errors is a key quality of developing robust systems. Currently, errors that occur during rendering in svelte will sometimes bubble all the way to the top and cause the entire app to crash or cause a subtree to crash, leading to an unstable state. Vue and React added `errorCaptured` and `componentDidCatch` hooks, respectively, to catch errors that occur within a subtree of an application. svelte, with it's use of blocks for advanced functionality in templates (e.g. `{#await}`), can add a similar error-catching primitive that will lead to more robust svelte applications.

## Detailed design

A new `{#try}` block is added that catches any rendering and lifecycle errors that occur within that block and passes them to the corresponding `{:catch ...}` block.

```html
{#try}
  <Flaky />
{:catch error}
  Uh oh, an error occurred.
  
  <details>
    <summary>Error Details</summary>
    {error}
  </details>
{/try}
```

These blocks can occur anywhere in the svelte application and can be placed at the top level to catch all errors or at a component level to catch granular errors that occur within specific components. When an error occurs, it propogates up the render tree until a `{#try}` block is reached, where the error stops propogation and is considered caught. Any sub-components of the `{#try}` block are unmounted (similar to an `{#if} ... {:else}` block) and the state of the subtree containing the error is thrown away.

In order for errors to propogate up to `{#try}` blocks, the error-handling of svelte would need to become more consistent, with rendering lifecycle errors always bubbling up to the top of the application. This would be a breaking change, but would most likely be welcome as an application in an unstable state is hard to diagnose and prone to further errors.

## How we teach this

Generally, an error boundary can be added at the top-level to display a nice error message for users, but they can be used throughout your app to handle errors at a more granular level. Examples in the guide can show the standard approach of adding a top-level error handler, with notes on usage at a component level as a next step.

## Drawbacks

The `{#try}` block can only catch certain kinds of errors, specifically those that occur during the rendering lifecycle. Errors that occur during event handlers or asynchronous code is unlikely to be caught.

## Alternatives

Add a new `onError` lifecycle hook for catching and dealing with errors. This would be consistent with the approaches of Vue and React and would allow for easier handling of errors from the javascript scope, but it would present some ambiguity on what is rendered when an error occurs and whether propogation stops or continues when an `onError` is encountered.

## Unresolved questions

- Best way to pass `error` from `{:catch error}` back to javascript scope for logging and other handling
- Best practices for recovering from the error and re-rendering the `{#try}` block
