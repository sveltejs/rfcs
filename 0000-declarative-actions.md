- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Declarative Actions

## Summary

This RFC proposes a declarative way to write actions.

Action:

```html
<!-- action.svelte -->
<script context="action">
  /*
    Not used in this example, I just bound the Element to demonstrate 
    actions could still be written purely imperatively as they are now
    with "onDestroy", "afterUpdate", "beforeUpdate" and "onMount". Or with
    an action specific API with only "onMount" and "onDestroy". "onUpdate"
    is no longer needed since you'd be able to use reactive declarations
    ("$: foo = bar;"), and Svelte would automatically update the DOM if you
    used the parameters in the markup.
  */
  let target;

  let isHeld = false;
</script>

<style>
  .red-background {
    background-color: red;
  }
</style>

<!-- target is an alias to the Element this action is applied to -->
<!--
  ".svelte" files with an action context script may only have
  one Element (excluding Special Elements), a target, and target 
  shall have 0 children **.
-->
<target
  class:red-background="{isHeld}"
  on:pointerdown="{() => {isHeld = true}}"
  on:pointerup="{() => {isHeld = false}}"
  bind:this="{target}"
/>
```

Consumer component:

```html
<!-- Consumer.svelte -->
<!-- works like current action consumption*  -->
<script>
  import action from "./action.svelte";
</script>

<style>
  .blue-text {
    color: blue;
  }
</style>

<div class="blue-text" use:action>
  I am declaratively styled by an action! yey
</div>
```

##### [\*parameters will slightly vary from current action consumption](#Parameters)

##### [\*\*more on the Target Element](<#The\ Target\ Element>)

## Motivation

# Think about how custom variables as props would affect this.

# Action arguments {} -> {{}} desugaring.

## Motivation

Currently, writing an action takes away all of Svelte's ergonomics. You can't use directives, you can't use {# ... } or {@ ... } blocks and you can't apply styles using css sheets (or `<style></style>` blocks) without somehow including them globally (my particular use case).

As an example of a slightly more complicated action than the one I provided in the [summary](#Summary), I will use the tutorial's [pannable](https://svelte.dev/tutorial/actions).

Something like:

```javascript
// pannable.js
export function pannable(node) {
  let x;
  let y;

  function handleMousedown(event) {
    x = event.clientX;
    y = event.clientY;

    node.dispatchEvent(
      new CustomEvent("panstart", {
        detail: { x, y },
      })
    );

    window.addEventListener("mousemove", handleMousemove);
    window.addEventListener("mouseup", handleMouseup);
  }

  function handleMousemove(event) {
    const dx = event.clientX - x;
    const dy = event.clientY - y;
    x = event.clientX;
    y = event.clientY;

    node.dispatchEvent(
      new CustomEvent("panmove", {
        detail: { x, y, dx, dy },
      })
    );
  }

  function handleMouseup(event) {
    x = event.clientX;
    y = event.clientY;

    node.dispatchEvent(
      new CustomEvent("panend", {
        detail: { x, y },
      })
    );

    window.removeEventListener("mousemove", handleMousemove);
    window.removeEventListener("mouseup", handleMouseup);
  }

  node.addEventListener("mousedown", handleMousedown);

  return {
    destroy() {
      node.removeEventListener("mousedown", handleMousedown);
    },
  };
}
```

Could become:

```html
<!-- pannable.svelte -->
<script context="action">
  let target;

  let x;
  let y;

  let panning = false;

  function handleMousemove(event) {
    const dx = event.clientX - x;
    const dy = event.clientY - y;
    x = event.clientX;
    y = event.clientY;

    target.dispatchEvent(
      new CustomEvent("panmove", {
        detail: { x, y, dx, dy },
      })
    );
  }

  function handleMouseup(event) {
    x = event.clientX;
    y = event.clientY;
    panning = false;

    target.dispatchEvent(
      new CustomEvent("panend", {
        detail: { x, y },
      })
    );
  }

  function handleMousedown(event) {
    x = event.clientX;
    y = event.clientY;
    panning = true;

    target.dispatchEvent(
      new CustomEvent("panstart", {
        detail: { x, y },
      })
    );
  }
</script>

<svelte:window
  on:mousemove="{panning ? handleMousemove: undefined}"
  on:mouseup="{panning ? handleMouseup: undefined}"
/>
<target on:mousedown="{handleMousedown}" bind:this="{target}" />
```

This way of writing actions seems more inline with Svelte's idiomaticity.
Since most of what you're doing with an action is modifying an Element I don't see why there shouldn't be a way to do so declaratively. This is the sole reason for the existence of directives, they make it easier to use DOM features without having to worry about cleanup or what is happening behind the scenes.

Apart from the ergonomic differences there are also functional differences. As I mentioned [above](#Motivation), being able to use Svelte's extensions to the Markup and being able to style the target Element without bypassing Svelte's style system are benefits of this implementation of actions that are in no way possible with the current one.

### More on CSS

Personally, styling is my greatest concern. Currently there is no way to abstract styles applied directly to an element cleanly. As of today there are 3 routes you can take:

1.  Make a wrapper Svelte Component with the styles which you wish to abstract:

```html
<!-- RedBackground.svelte -->
<style>
  .abstracted-styles {
    background-color: red;
  }
</style>

<div class="abstracted-styles" style="height: min-content; width: min-content;">
  <slot />
</div>
```

A problem with this specific solution would be, for example, if a consumer sets `border-style` and `border-radius`, the background would ignore that styling:

```html
<!-- Consumer.svelte -->
<script>
  import RedBackground from "./RedBackground.svelte";
</script>
<style>
  .rounded-corners {
    border-style: solid;
    border-radius: 30px;
    width: 160px;
    height: 90px;
  }
</style>

<RedBackground>
  <div class="abstracted-styles" />
</RedBackground>
```

This would cause the red background to extend past the border corners.
We could forward the `styles` attribute of the top level `div` but then, our consumers still wouldn't be able to use stylesheets (or `<style></style>` blocks) to style the component.

---

2.  Make a wrapper or an inner Svelte Component with the styles which you wish to abstract (for this example I will use the "inner" implementation, since you can guarantee that a Component only has one parent but not that it only has one child):

```html
<!-- RedBackground.svelte -->
<script>
  let target;
  $: {
    if (target)
      target.parentElement.style.setProperty("background-color", "red");
  }
</script>

<div bind:this="{target}" />
```

```html
<!-- Consumer.svelte -->
<script>
  import RedBackground from "./RedBackground.svelte";
</script>
<style>
  .rounded-corners {
    border-style: solid;
    border-radius: 30px;
    width: 160px;
    height: 90px;
  }
</style>

<div class="rounded-corners">
  <RedBackground />
  I am imperatively styled by a Svelte Component!
</div>
```

This option works fine but has the disadvantage of not being able to use Svelte's style system. You can only use inline styles. To use a CSS sheet (be it for style abstraction purposes or because you need to use an external library) you have to do something like:

```css
/* external.css */
.red-background {
  background-color: red;
}
```

```html
<!-- RedBackground.svelte -->
<script>
  let target;
  $: {
    if (target) target.parentElement.style.classList.add("red-background");
  }
</script>

<div bind:this="{target}" />
```

This way you need to rely on the consumer to somehow include your stylesheet globally.

---

3. Since you are doing everything imperatively you might as well write an action:

```javascript
// redbackground.js
export function redbackground(node) {
  node.style.setProperty("background-color", "red");
}
```

```html
<!-- Consumer.svelte -->
<script>
  import redbackground from "./redbackground.js";
</script>
<style>
  .rounded-corners {
    border-style: solid;
    border-radius: 30px;
    width: 160px;
    height: 90px;
  }
</style>

<div class="rounded-corners" use:redbackground>
  I am imperatively styled by a Svelte Action!
</div>
```

This has the same disadvantages as option 2.

## Detailed design

### Technical Background

> There are a lot of ways Svelte is used. It's hosted on different platforms;
> integrated with different libraries; built with different bundlers, etc. No one
> person knows everything about all the ways Svelte is used. What does someone who
> knows about Svelte but hasn't necessarily used anything outside of it need to
> know? Are there docs you can share?

> How do different libraries or frameworks implement this feature? We can take
> design inspiration from others who have done this well and improve upon the
> designs or make them better fit Svelte.

### The Target Element

The Target Element should alias to the the Element the action is applied to. Any attributes set in this Element should be set in the Element the action is applied to.

### `<style></style>`

Any style created in an action should be scoped, and should be removed if it is not used by the action itself, just like what happens in Svelte Components.

### Parameters

Currently the only way to pass multiple arguments to an action is by having an object as the second parameter. This is the convention I am proposing to implement parameters in declarative actions. The exported props of an action would be set with an object (it's really simple with an example).

Example (TypeScript for clarity):

Current implementation of actions:

```javascript
// parametersExample.ts
export type Args = {
    fontSizePixels: number;
    color?: string;
}
export function parametersExample(node: Node, args: args)
    const element = node as HTMLElement;
    element.style.setProperty("color", args.color);
    element.style.setProperty("fontSize", args.fontSizePixels + "px");
```

```html
<!-- Consumer.svelte -->
<script lang="ts">
  import { parametersExample } from "parametersExample";

  const color = "blue";
  const fontSizePixels = 34;
</script>

<div use:parametersExample="{{fontSizePixels, color}}" />
<div use:parametersExample="{{fontSizePixels}}" />
```

Declarative action:

```html
<!-- parametersExample.svelte -->
<script lang="ts">
  export let color: string = "";
  export let fontSizePixels: number;
</script>

<target
  stye="color: {color}; font-size: {fontSizePixels ? fontSizePixels + 'px' : 'initial'};"
/>
```

```html
<!-- Consumer.svelte -->
<script lang="ts">
  import { parametersExample } from "parametersExample.svelte";

  const color = "blue";
  const fontSizePixels = 34;
</script>

<div use:parametersExample="{{fontSizePixels, color}}" />
<div use:parametersExample="{{fontSizePixels}}" />

<!-- could be sugared to the following for single arguments-->
<div use:parametersExample="{fontSizePixels}" />
```

## How we teach this

> What names and terminology work best for these concepts and why? How is this
> idea best presented? As a continuation of existing Svelte patterns, or as a
> wholly new one?

> Would the acceptance of this proposal mean the Svelte guides must be
> re-organized or altered? Does it change how Svelte is taught to new users
> at any level?

> How should this feature be introduced and taught to existing Svelte
> users?

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Svelte,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the
> same domain have solved this problem differently.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD?
