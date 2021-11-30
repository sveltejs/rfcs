- Start Date: 2021-07-06
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Template ISO language support

## Summary

By default the `app.html` includes `<html lang="en">` but for localised pages need to replace the language used according to the context.

## Motivation

While building localised static pages, need to deliver correct language ISO on the html, to be SEO "friendly".
In the current setup to achieve the correct language code, all htmls files need to be parse and language code replace after the svelte build process is complete.

## Detailed design

### Technical Background

> How do different libraries or frameworks implement this feature? We can take

In other frameworks like [nextjs](https://nextjs.org/docs/advanced-features/custom-document) you can change the lang by doing

```
import { Html } from 'next/document';
<Html lang={htmlLang} />
```

### Implementation

From my quick investigation could implemented something similar to `svelte:head`

```
<svelte:head>
	<title>Home</title>
</svelte:head>
```

which replaces the string `%svelte.head%` from the template

https://github.com/sveltejs/kit/blob/6ef148df0eeeb9c29d863906068ecf1494056158/packages/kit/src/core/build/index.js#L291

So something within the lines of

```
// app.html
<html lang="%svelte.lang%">
```

```
/// page-to-render.svelte
<svelte:lang data-lang="{$isoLang}"/>
```

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

About terminolagy guess is good to follow same pattern describe on [w3](https://www.w3.org/2005/05/font-size-test/starhtml-test.html)
Since there are clear ISO defined about the topic

> How should this feature be introduced and taught to existing Svelte
users?

Would take same approach for what is defined for svelte:head
