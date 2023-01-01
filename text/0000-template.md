- Start Date: 2023-01-01
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Make an assignment  reactive without $: if it has reactive dependencies

## Summary

> It would be nice to have plain/concise assignment without redundant `$:` for variables that have reactive dependencies.

## Motivation

This is how we do now
```js
let count = 1
$: doubled = count * 2
```

And this is how it can be

```js
let count = 1
let doubled = count * 2
```

In case if we need one time `doubled` to be non reactive we could define it like
```js
const doubled = count * 2
```
