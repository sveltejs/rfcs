- Start Date: 2021-04-10
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# return:condition directive

## Summary

> **return:condition** directive, used on tags (and only tags!) which disables all attributes after it if condition is truthy

## Motivation

There are situations when it's needed to set a group of attributes or directives to a tag only when a certain condition is truthy.

Here is an [example of app with simple Input component](https://svelte.dev/repl/b90be4c311f04cac9736d420743199ad?version=3.37.0), written using `{#if}`.

Purpose of this component is to control styling and directives depending on the props passed to it.

##### Input.svelte

> _Script and style tags will be used in all following examples, but won't be specified to reduce size of the RFC_

```svelte
<script>
    import { someStore } from "./store.js";

    export let property;
    export let hasBorder, isNormal, isNumber;
    export let placeholder = "", value = "";
</script>

{#if isNormal}
    <input
        type="text"
        class:hasBorder

        {placeholder} {value}
    />
{:else if isNumber}
    <input
        type="number"
        class:hasBorder

        {placeholder}
        bind:value={$someStore[property]}
    />
{:else}
    <input
        type="text"
        placeholder="No value and placeholder"
    />
{/if}

<style>
    input {
        width: 225px;
        padding: 16px 24px;

        background: rgb( 200, 200, 200 );
        outline: none;

        color: rgb( 0, 0, 0 );

        font-size: 18px;
    }

    .hasBorder {
        border: 1px solid rgb( 0, 0, 0 );
    }

    input[ type="number" ] {
        width: 175px;

        background: rgb( 150, 140, 140 );
    }

    input::placeholder {
        color: rgba( 0, 0, 0, 0.5 );
    }
</style>
```

However, this approach is not good _enough_ because in order to add another input type you have to duplicate markup (which means at least 4 unnecessary lines of markup for each type. And that's not counting duplicated attributes).

It is also possible to _remake_ this example code by combining all attributes and directives in one input:

```svelte
<input
    type={!isNumber ? "text" : "number"}
    class:hasBorder={(isNormal || isNumber) && hasBorder}
    placeholder={isNormal || isNumber ? placeholder : "No value and placeholder"}
    value={isNormal ? value : ""}
    on:input={e => {
        if(!isNormal && isNumber) {
            $someStore[property] = e.target.value;
        }
    }}
/>
```

However, the whole process is more like encryption than refactoring and this kind of code is hard to maintain.

Situation can be improved a bit by moving some javascript to `script` tag

```svelte
<script>
    // imports and exports

    const isSpecial = isNormal || isNumber;
    const setProperty = e => {
        if(!isNormal && isNumber) {
            $someStore[property] = e.target.value;
        }
    }
</script>

<input
    type={currentType}
    class:hasBorder={isSpecial && hasBorder}
    placeholder={isSpecial ? placeholder : "No value and placeholder"}
    value={isNormal ? value : ""}
    on:input={setProperty}
/>
```

However, this seems more like **_avoiding the problem_** instead of **_solving_** it.

I propose to implement a `return:condition` directive that can help solve such problems.

This is how the original example would have looked if it had been reworked with this directive:

```svelte
<input
    type="text"
    placeholder="No value and placeholder"

    return:defaultInput={!isNormal && !isNumber}

    class:hasBorder
    value={value}
    placeholder={placeholder}

    return:isNormal

    bind:value={$someStore[property]}
/>
```

It may look a bit tricky at first but after a few uses you can quite easily figure out how this kind of code works.

## Detailed design

### Technical Background

This directive is quite similar to how javascript and programming languages as such work.

It's just `if(condition) return;`

The only difference is that if `condition` is truthy, then processing of all following attributes stops and all (except duplicated) previous attributes and directives are applied to the tag.

> How do different libraries or frameworks implement this feature?

Svelte is (officially) first* :wink:

### Implementation

I will explain how this directive might work using input from previous section as an example.

```svelte
<input
    type="text"
    placeholder="No value and placeholder"

    return:defaultInput={!isNormal && !isNumber}

    class:hasBorder
    value={value}
    placeholder={placeholder}

    return:isNormal

    bind:value={$someStore[property]}
/>
```
<br/>
Let's suppose that tag has its "object" with tag's attributes and directives.

First, two values are assigned to this object:
```js
object["type"] = "text";
object["placeholder"] = "No value and placeholder";
```
If condition(`!isNormal && !isNumber`) is falsy, then existing attributes are assigned to the tag(**done!**).

Otherwise, new attributes and directives are assigned to object
>Don't forget that this is just an example!
```js
object["class"] = hasBorder ? "hasBorder": "",
object["value"] = value,
object["placeholder"] = placeholder
```
---
Note that old `placeholder` value before `return:` directive has been replaced by new value.
It's valid syntax(according to my understanding of this directive).

However, this use case will still give **`Attributes need to be unique`** error :

```svelte
<input
    type="text"
    placeholder="No placeholder"
    placeholder="New one!"

    return:condition

    placeholder="And another one!"
/>
```
---
And lastly, if isNormal is truthy, bind:value is applied to the tag(**done!**).
Otherwise, all previous attributes are assigned to the tag(**done!**)

That's all for now. If you have any questions about the behavior of this directive I can tell you my opinion

> Will the change have performance impacts? Can you quantify them?

The directive itself costs practically nothing in terms of performance. (Almost) All calculations can be even performed at compile time and, maybe, it's even possible to reduce bundle size.

However, depending on how it's implemented, there could be situations where performance could drop.

If `return:` directive is implemented as an action ( only executed when an element is created ), there will be no problem, however there will be fewer uses for it.

If it will be called every time `condition` changes, then in this example:

```svelte
<script>
    export let left = false;
</script>

<div
    style="left: 50px;"

    return:isLeft

    style="right: 50px;"
></div>
```
If `isLeft` changes frequently, but not slowly enough for the browser to optimize style change, this can lead to performance problems.

However, this behavior is not covered as with `class:` directive, so this should be discussed further.

#### Options

`return:condition` directive _may_ have three options:

**Full**. When all conditions are placed in curly brackets after a certain label.
You can call this label as you like. It only performs a descriptive role.

```svelte
<div
    class:light
    return:darkness-shall-not-pass={!light}
    data-light="It's light!"
></div>
```

**Shorthand**. When there is no curly brackets and condition-variable is the label itself (as in `class:condition` syntax)
```svelte
<div
    class:light
    return:light
    data-dark="It's dark >:)"
></div>
```

> When `light` is truthy `data-light` attribute is added to the tag.
> When `light` is falsy initially or after it was truthy - `data-light` disappears or `data-dark` appears.

**Even shorter (to be discussed)**. When `return:` shares a condition with `class:condition` directive
```svelte
<div
    return:class:light
    data-dark="It's dark >:)"
></div>
```
Which is equivalent to **Second** mode, but shorter.

In my opinion, this implementation fits well with the Svelte design, but you can suggest your own options.

This directive will be especially useful for UI library developers, but it can be useful in any kind of project.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

Described in
**Detailed design -> Technical Background**
and in
**Detailed design -> Implementation -> Options**

> Would the acceptance of this proposal mean the Svelte guides must be
re-organized or altered? Does it change how Svelte is taught to new users
at any level?

If a person begins to learn svelte after learning any programming language - no

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Svelte,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

This is not yet implemented anywhere AFAIK, so _theoretically_ there _may_ be some unknown difficulties in developing this directive

## Alternatives

> ..., how other frameworks in the same domain have solved this problem differently.

Svelte is (officially) first :wink:

## Unresolved questions

* Should `return:class:condition` syntax be added?
* Should this directive be executed on each change of condition or only on creation of a tag?
