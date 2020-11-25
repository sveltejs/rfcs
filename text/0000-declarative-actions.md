- Start Date: 2020-11-25
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
  one Element (excluding some Special Elements), a target, and target 
  shall have 0 children.*
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
<!-- works like current action consumption**  -->
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

##### [\*more on the `<target />` Element](<#The\ \`\<target /\>\`\ Element>)

##### [\*\*parameters will slightly/greatly differ from current action consumption](#Parameters)

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
Since most of what you're doing with an action is modifying an Element I don't see why there shouldn't be a way to do so declaratively. This is the sole reason for the existence of some directives, they make it easier to use DOM features without having to worry about cleanup or what is happening behind the scenes.

Apart from the ergonomic differences there are also functional differences. As I mentioned [above](#Motivation), being able to use Svelte's extensions to the Markup and being able to style the target Element without bypassing Svelte's style system are benefits of this implementation of actions that are in no way possible with the current one.

### More on styling

Personally, styling is my greatest concern. Currently there is no way to abstract styles applied directly to an element cleanly. As of today there are 3 routes you can take:

- Not abstracted:

```html
<!-- BaseLine.svelte -->
<style>
  .styles-to-abstract {
    background-color: red;
  }
</style>

<div class="styles-to-abstract">I'm a styled div!</div>
```

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
  <div class="rounded-corners">I'm a div styled by my parent!</div>
</RedBackground>
```

This would cause the red background to extend past the border corners.
We could forward the `styles` attribute of the top level `div` but then, our consumers still wouldn't be able to use stylesheets (or `<script></script>` blocks) to style the component.

---

2.  Make a wrapper/inner Svelte Component that imperatively styles the desired Element (for this example I will use the "inner" implementation, since you can guarantee that a Component only has one parent but not that it only has one child):

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
  I am imperatively styled by an inner Svelte Component!
</div>
```

This option works fine but has the disadvantage of not being able to use Svelte's style system. You can only use javascript to do the styling. To use a stylesheet (be it for style abstraction purposes or because you need to use an external library) you have to do something like:

```css
/* external.css */
.abstracted-styles {
  background-color: red;
}
```

```html
<!-- RedBackground.svelte -->
<script>
  let target;
  $: {
    if (target) target.parentElement.style.classList.add("abstracted-styles");
  }
</script>

<div bind:this="{target}" />
```

This way you need to rely on the consumer to somehow include your stylesheet (`external.css`) globally.

---

3. Since you are doing everything imperatively you might as well use an action:

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

---

With Declarative Actions it would look like this:

```html
<!-- redbackground.svelte -->
<script context="action"></script>

<style>
  .abstracted-styles {
    background-color: red;
  }
</style>

<target class:abstracted-styles="{true}" />

<!------------------------------------------------------------------------------------------- -->
<!-- I avoided: -->
<target class="abstracted-styles" />
<!-- or -->
<target style="background-color: red;" />
<!-- because, in this case, I didn't want to overwrite the styles/classes set by the consumer -->
```

Consumer component:

```html
<!-- Consumer.svelte -->
<script>
  import redbackground from "./redbackground.svelte";
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
  I am declaratively styled by a Svelte Action!
</div>
```

## Detailed design

### The `<target />` Element

The `<target />` Element should alias to the the Element the action is applied to. Any attributes set in the `<target />` Element should be set in the Element the action is applied to and vice versa.

In the summary I alluded to the fact that `<target />` should be the only Element present in an action, apart from `svelte:window`, `svelte:head` and `svelte:body`. Should this be the case? I could see someone wanting to modify the DOM tree with an Action. Should we encourage this? If someone really needs to do this, they can do so imperatively anyway, but would lose all the goodness of Declarative Actions. We could even allow `<target />` to have children (?!?! iffy). The children could be added at the end/start of the children the Element the Action is applied to already has. Or could replace them (!?! idk). In my opinion adding elements around the `<target />` (parents, grandparents, etc.) could be an idea worth discussing but adding children seems too convolute. Siblings should not be supported in my opinion.

Something like:

```javascript
export function domTreeManipulation(node) {
  const dupNode = node.cloneNode(true);

  const div = document.createElement("div");

  const span = document.createElement("span");
  span.innerText = "DOM Tree Manipulation";

  const dupSpan = span.cloneNode(true);

  div.appendChild(span);
  div.appendChild(dupNode);
  div.appendChild(dupSpan);

  node.replaceWith(div);
}
```

Could become:

```html
<!-- domTreeManipulation.svelte -->
<script context="action"></script>

<div>
  <span>DOM Tree Manipulation</span>
  <target />
  <span>DOM Tree Manipulation</span>
</div>
```

### `<style></style>`

Any style created in an action should be scoped, and should be removed if it is not used by the action itself, just like what happens in Svelte Components.

### Parameters

Currently the only way to pass multiple arguments to an action is by having an object as the second parameter. This is a convention we could use to pass arguments to Declarative Actions. The exported props of an action would be set with an object (it's really simple with an example).

Example (TypeScript for clarity):

Current implementation of actions:

```javascript
// parametersExample.ts
export type ArgsType = {
    fontSizePixels: number;
    color?: string;
}

export function parametersExample(node: Node, args: ArgsType) {
    const element = node as HTMLElement;
    if (args.color) element.style.setProperty("color", args.color);
    element.style.setProperty("font-size", args.fontSizePixels + "px");
}
```

```html
<!-- Consumer.svelte -->
<script lang="ts">
  import { parametersExample } from "parametersExample";

  const color = "blue";
  const fontSizePixels = 34;
</script>

<div use:parametersExample="{{fontSizePixels, color}}">2 arguments</div>

<div use:parametersExample="{{fontSizePixels}}">1 argument</div>
```

Declarative action:

```html
<!-- parametersExample.svelte -->
<script context="action" lang="ts">
  export let color = "initial";
  export let fontSizePixels: number;
</script>

<target style="color: {color}; font-size: {fontSizePixels + 'px'};" />
```

```html
<!-- Consumer.svelte -->
<script lang="ts">
  import { parametersExample } from "parametersExample.svelte";

  const color = "blue";
  const fontSizePixels = 34;
</script>

<div use:parametersExample="{{fontSizePixels, color}}">2 arguments</div>

<div use:parametersExample="{{fontSizePixels}}">1 argument</div>

<!-- could be sugared to the following for single arguments-->
<div use:parametersExample="{fontSizePixels}">1 argument</div>
```

Another option would be to rework the parameters completely. An idea I had, which could even potentially be used with the current actions, would be to use [Custom Data Attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/data-*). The action could capture the respectively named Data Attribute. A parameter called `foo` would have the value of an attribute called `data-foo` (or even `data-[actionname]-foo` to prevent name collisions, if the consumer wants to use Data Attributes for other purposes, maybe even other actions).

Example Consumer (the Action would be the same as the Declarative Action above):

```html
<!-- Consumer.svelte -->
<script lang="ts">
  import { parametersExample } from "parametersExample.svelte";

  const color = "blue";
  const fontSizePixels = 34;
</script>

<div
  use:parametersExample
  data-color="{color}"
  data-fontSizePixels="{fontSizePixels}"
>
  2 arguments
</div>

<div use:parametersExample data-fontSizePixels="{fontSizePixels}">
  1 argument
</div>

<!-- there could even be a directive to set Data Attributes -->
<div use:parametersExample data:fontSizePixels="{fontSizePixels}">
  1 argument
</div>

<!-- or, for short -->
<div use:parametersExample data:fontSizePixels>1 argument</div>
```

## How we teach this

We should teach this roughly the same as we teach Components with Slots. Though `<target />` "only accepts one child", since the action is applied with a directive, therefore it can alias to said child.

Current svelte guides shouldn't have to be reorganized since there should be a section for Actions already.

## Drawbacks

1. This would be a breaking change. Although it may be possible to keep the old actions around (I assume, I'm not well versed in the Svelte internals), it might not be worth the complexity. Having multiple ways to write an action might be more confusing than it needs to be, specially for new users.

2. Verbosity. For simple actions that need to be written imperatively anyway, this will be more cumbersome.

3. As for teaching, although this is a little more complicated than just a simple function, it shouldn't be that hard to teach/learn (I think ?).

4. Performance? I have no clue (it could even be a benefit (?!) rather than a drawback since Declarative Actions would be statically analyzable).

##### I will add more as/if they rise in the pull request.

## Alternatives

The only alternative I could come up with was to introduce the `<target />` Element, as described, and allow its use in Components, which would leave the current Action developer experience (arguably lesser than Declarative Actions) as it is.

By not making this change, as I mentioned above, it would still be impossible to apply styles directly to an Element defined, styled, etc.. by the consumer.

##### I will add more as/if they rise in the pull request.

## Unresolved questions

I have brought up some already:

1. Should we allow more Elements than the `<target /> ` Element in an action?
2. What approach should we use for parameters (Data Attributes vs Object)?

More:

3. If we chose the "Data Attributes" approach to parameters could we integrate this with Style Properties? PR [#13](https://github.com/sveltejs/rfcs/pull/13)

   Something like `data:--some-style` could be interpreted as a Style Property. If we don't, it might be hard to receive styles as arguments and use them without overwriting the `styles` attribute without using some sort of CSS-in-JS solution.

4. If we chose the "Data Attributes" approach, could we just provide both a directive to set Data Attributes and a directive to set Action parameters and use Data Attributes in the background?
   Ex.: `data:actionname:parametername={value}`. This way we could obfuscate the parameter names to avoid collisions.

5. How would constant props work?

6. How should we distinguish Components from Actions?

   My idea would be to add `context="action"` to scripts in actions. `context="module"` would still be supported. A single `.svelte` file shouldn't have a script without context and a script with `context="action"`. `<target />` shouldn't be used in `.svelte` files without a `context="action"` script. Another way could be to force actions' files' names to start with a lower case letter.

   I'm open to suggestions.

7. This is a personal annoyance. What is the current convention for action names? All lower case? Camel case? What should it be for Declarative Actions?

8. Should custom CSS properties set inside the Action be available to the children of the component the Action is applied to?

##### I will add more as/if they rise in the pull request.
