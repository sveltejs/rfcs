- Start Date: 2019-01-19
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Better composition

## Summary

Svelte 3 components can be combined in various ways to do just about anything. But some tasks are unnecessarily difficult or boilerplate-ridden. This RFC proposes a set of changes to the way slotted content behaves to enable more idiomatic and powerful composition patterns.

## Motivation

### Uncontrolled components

We want to be able to do things like this (from https://github.com/reactjs/react-tabs)...

```html
<Tabs>
  <TabList>
    <Tab>Title 1</Tab>
    <Tab>Title 2</Tab>
  </TabList>

  <TabPanel>
    <h2>Any content 1</h2>
  </TabPanel>
  <TabPanel>
    <h2>Any content 2</h2>
  </TabPanel>
</Tabs>
```

...or this...

```html
<Tabs>
  <Tab title="Title 1">
    <h2>Any content 1</h2>
  </Tab>

  <Tab title="Title 2">
    <h2>Any content 2</h2>
  </Tab>
</Tabs>
```

...or any other configuration. Clicking on a `<Tab>` component should make the corresponding content visible. Ideally, invisible tabs shouldn't be rendered — in other words, there should only be one `<h2>` element existing at any given moment. Currently, Svelte renders slotted content *eagerly*, similar to light DOM in the web components arena.

Equally, we might want to create compound chart components...

```html
{#each anscombesQuartet as points}
  <Chart {points}>
    <XAXis ticks labels/>
    <YAXis ticks/>
    <Scatterplot/>
    <VoronoiOverlay on:highlight={highlight}/>
  </Chart>
{/each}
```

...in which the axis and plot components (and any other — e.g. `<Annotation>`) use the scale and data provided by the `<Chart>`.

This all implies inter-component communication that's invisible to the application author — something that's effectively impossible in Svelte currently — and a different approach to passing slotted content to components.


### Slot context and iteration

Another problem that comes up from time to time: slots can only be used once. You can't use slotted content to provide the definition for [svelte-virtual-list](https://github.com/sveltejs/svelte-virtual-list), for example, even though that would generally be more convenient than passing in a component as a prop.


#### Current approach

```html
<script>
  import VirtualList from '@sveltejs/svelte-virtual-list';
  import RowComponent from './RowComponent.html';

  const things = [
    { name: 'one', number: 1 },
    { name: 'two', number: 2 },
    { name: 'three', number: 3 },
    // ...
    { name: 'six thousand and ninety-two', number: 6092 }
  ]
  };
</script>

<VirtualList items={things} component={RowComponent} />
```

```html
<!-- RowComponent.html -->
<div>
  <strong>{number}</strong>
  <span>{name}</span>
</div>
```


#### Better approach?

```html
<script>
  import VirtualList from '@sveltejs/svelte-virtual-list';

  const things = [
    // these can be any values you like
    { name: 'one', number: 1 },
    { name: 'two', number: 2 },
    { name: 'three', number: 3 },
    // ...
    { name: 'six thousand and ninety-two', number: 6092 }
  ]
  };
</script>

<VirtualList items={things}>
  <div>
    <!-- markup goes here -->
  </div>
</VirtualList>
```


## Detailed design

### Lazy rendering

For much of what's outlined above, we need slotted content's lifecycle to be controlled by the child component, rather than the parent. This is at odds with the current approach, in which all slotted content is rendered by the parent and then passed to the child, much like light DOM is passed to a custom element.

I'll update this section with some example code as soon as I can unknot my brain.


### Uncontrolled component context

For a pattern like this to work...

```html
<Tabs>
  <TabList>
    <Tab>Title 1</Tab>
    <Tab>Title 2</Tab>
  </TabList>

  <TabPanel>
    <h2>Any content 1</h2>
  </TabPanel>
  <TabPanel>
    <h2>Any content 2</h2>
  </TabPanel>
</Tabs>
```

...each `<Tab>` component needs to be able to tell `<Tabs>` (possibly via `<TabList>`, depending on how the component is authored) that it has been selected. It also needs to know its index relative to its siblings, even if it is created after the initial render, or an earlier tab is removed after the initial render.

Similarly, each `<TabPanel>` needs to know its own index (which is again subject to mutations) and the currently selected index. If a panel *is* selected, it should render its child content, otherwise it should not.

Perhaps it could look like this:

```html
<!-- Tabs.html -->
<script context="module">
  export const TABS = {};
</script>

<script>
  import { setContext } from 'svelte';
  import { writable } from 'svelte/store';

  const tabs = [];
  const panels = [];
  const selected = writable(null);

  setContext(TABS, {
    registerTab: tab => {
      tabs.push(tab);
    },

    unregisterTab: tab => {
      const i = tabs.indexOf(tab);
      tabs.splice(i, 1);
    },

    registerPanel: panel => {
      panels.push(panel);

      // if this is the first panel, select it
      selected.update(current => current || panel);
    },

    unregisterPanel: panel => {
      const i = panels.indexOf(panel);
      panels.splice(i, 1);
    },

    selectTab: tab => {
      const i = tabs.indexOf(tab);
      selected.set(panels[i]);
    },

    selected
  });
</script>

<div class="tabs">
  <slot></slot>
</div>
```

```html
<!-- Tab.html -->
<script>
  import { getContext, onDestroy } from 'svelte';
  import { TABS } from './Tabs.html';

  const tab = {};
  const { registerTab, unregisterTab, selectTab } = getContext(TABS);

  registerTab(tab);

  onDestroy(() => {
    unregisterTab(tab);
  });
</script>

<button on:click="{() => selectTab(tab)}">
  <slot></slot>
</button>
```

```html
<!-- TabPanel.html -->
<script>
  import { getContext, onDestroy } from 'svelte';
  import { TABS } from './Tabs.html';

  const panel = {};
  const { registerPanel, unregisterPanel, selected } = getContext(TABS);
  registerPanel(panel);

  onDestroy(() => {
    unregisterPanel(panel);
  });
</script>

{#if $selected === panel}
  <slot></slot>
{/if}
```

> This is just a first draft — I'm glossing over a few things in this example

Here, `setContext` and `getContext` are functions that must be called during the component's initialisation, like lifecycle functions. `getContext(arg)` retrieves any context that was set using `arg` (which could be anything including a string like 'tabs', but using `{}` guarantees no collisions) *in a parent component*. This allows any given component to have multiple instances of `<Tabs>`, or even for `<Tabs>` to be nested.

> TODO draw the rest of the owl


### Explicit slot context

The above example works for *implicit* context, i.e. that shared between components without the app author having to worry about it. Sometimes, we need *explicit* context, such as with the `<VirtualList>` example.

React can achieve this with the render prop pattern:

```html
<VirtualList items={things}>{({ number, name }) =>
  <div>
    <strong>{number}</strong>
    <span>{name}</span>
  </div>
}</VirtualList>
```

That's a little trickier for us. Perhaps we could achieve the same result with a new directive, `expose` (🐃):

```html
<VirtualList items={things} expose:item>
  <div>
    <strong>{item.number}</strong>
    <span>{item.name}</span>
  </div>
</VirtualList>
```

The `expose:item` directive, short for `expose:item={item}`, makes `item` available to child content, in much the same way as `{#each items as item}`. The `<VirtualList>` component would make it available like so:

```html
{#each visible as row (row.index)}
  <div class="row">
    <slot item={row.item}></slot>
  </div>
{/each}
```

We could potentially allow destructuring as well:

```html
<VirtualList items={things} expose:item="{{ name, number }}">
  <div>
    <strong>{number}</strong>
    <span>{name}</span>
  </div>
</VirtualList>
```

For non-default slots, the directive would live on the slotted element:

```html
<div slot="footer" expose:year>
  <p>Copyright {year} SvelteJS Inc</p>
</div>
```

In terms of the generated code, the child component would augment its own context with any properties on the slot:

```js
const slot_context = Object.assign(component_ctx, {
  item: ctx.item
});
```

> TODO draw the rest of the owl


## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Svelte guides must be
re-organized or altered? Does it change how Svelte is taught to new users
at any level?

> How should this feature be introduced and taught to existing Svelte
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Svelte,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?