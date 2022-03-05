- Start Date: 2021-07-04
- RFC PR: 
- Svelte Issue: https://github.com/sveltejs/svelte/issues/6488

# Syntax sugar for reactive declarations

## Summary

This RFC propose breaking change, that would simplify usage of reactive assignments, when we use both variables that should be dependency for reactivity and those, that shouldn't be.

## Motivation

Today, when we need to create reactive declaration with function, that contain two parameters, and we want to `react` only, when one of them change, we need to encapsulate it to another function (`callmyfunction` in our examples), and write variable name two times. 

```js
/* thanks to Idle for example code */
function callmyfunction() {
  myfunction(callfuncwhenthisvarchange, dontcallwhenchange)
}
$: callfuncwhenthisvarchange, callmyfunction()

```

or

```js
/* thanks to Rainlife for example code */
const callmyfunction = () => myfunction(callfuncwhenthisvarchange, dontcallwhenchange);
$: callfuncwhenthisvarchange, callmyfunction();
```

that is a lot of unnecessary code...

I suggest to add new feature / syntax sugar, that will not use variable for reactive assignments, if it will be prefixed with `_`, so previous code would be possible to rewrite to something like this:

```js
$: myfunction(callfuncwhenthisvarchange, _dontcallwhenchange);
```

Same can be aplied on expressions like:

```js
$: z = _x + y
```

and in other situations as well.


## Breaking change problem?

As You know, `_` is often used for unused variables. So if we say variable will not be used for calling reactive statement, then it's really something great and intuitive. In general it's possible to **call** function with `_` in parameter, but it's used rarely (more often we see usage of underscore for callbacks or function definitions/declarations), so I don't think, this will be problem, even if we implement it before Svelte 4.

## Drawbacks and IDE intellisense

One of the drawback is that IDEs and text editors like VS Code `mark` variables prefixed with `_` as unused. In some rare situations, some programmers call functions with `_` prefix in parameters (didn't even saw code like that, but it's valid), so it can probably break (really) small amount of projects. Maybe we can for few versions put warning in console, when programmer is using this.

## Alternatives

I also considered to design with writing array of variables, that would be dependencies for reactive statement. The problem is, it's more bloated, and require writing variable name two times. I think `_` (prefix) idea is most ideal. Alternative for prefix can be `#`.

## Unresolved questions

You are free to ask in PR.
