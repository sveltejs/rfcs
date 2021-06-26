- Start Date: 2021-4-29
- RFC PR: 
- Svelte Issue: 

# Switch Block

## Summary

> This feature is a switch statement but for declaring reactive UI simmilar to `{#if}`.
> It will behave exactly the same as a switch statement in regular programming.

## Motivation

> This could be useful for simple single page app that needs to switch between layouts,
> like if you wanted a login/signup/logout/forgot password page, each page would only
> have their own seperate elements, but not enough to warrant multiple svelte files.

> It would be easier to do simple layout changes where you need to modify the layout
> based on dynamic data and you don't want to do a large amount of `{:else if}`
> blocks. Plus it would better describe the behaviour of what it's doing.

> With switch statements you do not have to reference your varible multiple times
> and possibly use more resources than required. That's the main benefit to
> switch statements in programming, performance. Although with modern hardware the
> benefits of switch statements speed are probably 1ms or less it may be beneficial
> to lower end hardware.

> My reasoning for adding this is mainly syntactic sugar (lots of else if blocks are ugly, 
> but the performance benefits of a switch statement very are real, but very minimal.

## Detailed design
```
{#switch *condition*} -- The default tags could be under the main switch statement, or it could be in a seperate tag as {:default}
  <Default Content>
{:case *constant expression*}
  <Case specific content>
{/switch}
```

### Technical Background

> There are the standard docs for many languages such as 
> c++ https://docs.microsoft.com/en-us/cpp/cpp/switch-statement-cpp?view=msvc-160
> JavaScript: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/switch
> Many languages I have seen just use the standard c syntax so there is no need to list more

> React just uses normal javascript switch statements and then returns different values. https://sebhastian.com/react-switch/
> There is a vue module which brings support for switch statements https://www.npmjs.com/package/v-switch-case

### Implementation

> It behaves very simmilar to `{#if}`, `{:else if}`, and `{:else}`, but
> the conditions are static and it's backed by the switch statement of normal javascript.
> The `{:else}` block is now in the `{#switch}` block, and the `{#if}` and `{#else if}`
> conditions now are located in `{:case}`. I came to this after I was making a page
> for doing various account auth related things with login, logout, signup, and reset password
> which could all be put in one svelte file and didn't need their own files but the switch
> statement which I use regularly in javsacript doesn't exist. So I had to use `{#else if}` blocks
> as a bulky, inefficient stand in for a switch statement.

## How we teach this

> It would talk about the logic

> Would the acceptance of this proposal mean the Svelte guides must be
re-organized or altered? Does it change how Svelte is taught to new users
at any level?

> How should this feature be introduced and taught to existing Svelte
users?

## Drawbacks

> The only real drawback that I could see is a very minescule bundle size increase,
> since this is only adding on a new conditional block and not modifying a core feature
> it most likely wont affect any current projects.

> It may be confusing to some newer programmers to why you need switch statements when you
> already have if statements.

## Alternatives

> Just not do this because you can already do this with relative ease, but this will
> bring slight performance improvements to apps that use it as well as cleaner syntax.

## Unresolved questions

> Wether it uses `{:default}` or the default markup tags will just go below the `{#if}` statement.
