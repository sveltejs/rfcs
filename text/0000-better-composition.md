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
  ];
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
  ];
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


### Explicit slot scope

The above example works for *implicit* context, i.e. that shared between components without the app author having to worry about it. Sometimes, we need *explicit* context, or 'scope', such as with the `<VirtualList>` example.

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
const slot_scope = Object.assign(component_ctx, {
  item: ctx.item
});
```

> TODO draw the rest of the owl


## How we teach this

Terminology-wise, 'uncontrolled component' and 'context' are terms in fairly widespread use in the React community, and 'slot scope' is used in Vue-land. It makes sense to use the same language.

For the vast majority of users, these changes are purely additive, requiring no real reorganization of existing documentation. The only change (see 'drawbacks') is that `<slot>` no longer behaves exactly like its native HTML equivalent, since we're now able to inject slot scope.


## Drawbacks

It's tricky to implement, and adds complexity. Then again the current slot mechanism is also somewhat tricky.

The current method makes it possible to use the slots API programmatically, by passing a `slot: { default, named }` object at initialisation (where `default` and `named` are HTML elements or document fragments). That would no longer be possible if the lifecycle is controlled by the child rather than the parent. Personally I haven't ever used this API, and I'd be surprised if many people have, but it is a drawback nonetheless.

Finally, this proposal moves us away from alignment with web components. A Svelte component that had `<slot>` inside an each block (such as svelte-virtual-list) couldn't realistically be compiled to a web component. It wouldn't be much use as a web component in its current form though.

## Alternatives

The alternative is to do nothing, and rely on existing methods of composition. As we've seen, there are limits to the current approach.

Context could be designed differently. React does it like this:

```js
import { createContext, useContext } from 'react';

const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <ChildComponent/>
    </ThemeContext.Provider>
  );
}

function ChildComponent() {
  const theme = useContext(ThemeContext);

  return (
    <div className={theme}>
      <p>Current theme is {theme}</p>
    </div>
  );
}
```

In other words, context can be created anywhere inside the component's markup, rather than just in the `<script>` block upon instantation with `setContext`. This is a theoretical benefit but would become unwieldy when using more complex values (such as the context created by the `<Tabs>` component example). 'Context' in this RFC is more than just a simple value; it's potentially a way for related components to communicate with each other, obviating the need for a component to manipulate its children the way that React uncontrolled components typically do.


## Unresolved questions

Have we named everything correctly?