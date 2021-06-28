- Start Date: 2020-12-12
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Commenting out HTML attributes

## Summary

This RFC proposes a way to be able to comment out individual attributes within an HTML tag. It would make this valid:

```html
<Draggable
  {x} {y}
  bind:m={ matrices[i] }
  //on:some      <-- this whole line would be ignored
>
```

The current implementation (Svelte 3.31.0) passes the `//on:some` tag to Rollup, which breaks the build:

```
[rollup] [!] (plugin svelte) ParseError: Expected >
[rollup] app/App.svelte
[rollup] 150:           x={x} y={y}
[rollup] 151:           bind:m={ matrices[i] }
[rollup] 152:           //on:some
[rollup]                 ^
[rollup] 153:           {validate}
[rollup] 154:           {onBan}
```

## Motivation

Comments are commonplace in programming but not allowed by the XML syntax, within a tag. Svelte, being a compiler, could extend the syntax by allowing them and simply remove such lines from the text passed on to Rollup.

The proposal is:

- Within a tag
   - if a line matches `^\s*\/\/.*$` regex (starts with white space, followed by `//`
      - such line is replaced by an empty line (to keep line numbering unchanged)

Sample:

```
<Sample a=10 b=12
  some="42 //"
  //other    tbd.
  //further=13   <-- disabled
/>
```

Use cases:

- one wants to temporarily disable an attribute
- reminder that something needs to be done, in the future


### Prior discussions

Prior to making this RFC, I mentioned the idea at the [Svelte Discord](https://discord.com/channels/457912077277855764/653341885674553345/786609908807499796) and got two favourable comments, so decided to push further.

>I second your idea @asko ...I find myself wanting to do this often :slight_smile:

>me too, it's kinda annoying to move the property to other line and comment it and then put it back to where it was when I want to uncomment it.


### Why not proper end-of-line comments?

The author feels that the beginning whitespace + `//` is good enough to cover all current real world use cases. Proper end-of-line comments would allow eg. `a=12  // 12 = 4*3` but attributes rarely need commenting like that. It is, instead, the disable/enable use case that is leading this RFC.

### Why `//`?

XML itself does not sport an end-of-line comment syntax.

`//` should be intuitive from JavaScript and (as we have seen in the summary) it does not break existing working code, since Rollup builds choke on it.

`#` (bash, Makefile) is already being used for something else.

`--` (Lua) is suggested in the [style properties RFC](https://github.com/sveltejs/rfcs/blob/master/text/0000-style-properties.md) for other purposes.


## Detailed design

The implementation would be a single-line-ish change in the Svelte parser.

The larger implications are with the ecosystem: Svelte syntax highlighters (VS Code, WebStorm et.al.) should eventually render such within-tag lines as comments. The author is not aware, whether the Svelte parser is used by these highlighters (easy to change) or whether they implement their own (takes longer).

### Early prototyping

One could try to make custom highlighting rules for WebStorm, to dim down the suggested lines.

One could do a filter to Svelte parsing (if there is support for such?), to make an early implemantation of this RFC.


## How we teach this

Adding `//` to the beginning of attributes seems intuitive to web developers so they might just find it to work, at least if IDE syntax highlighting supports it.

However, since this would be a non-standard HTML feature, it likely deserves to be mentioned in the docs, maybe between [HTML tags](https://svelte.dev/tutorial/html-tags).

Sample text:

>Svelte allows you to disable/enable HTML attributes by preceding them with `//`. Such lines are ignored. [Next]

For existing users, a notice in the [Changelog](https://github.com/sveltejs/svelte/blob/master/CHANGELOG.md) should do, since it's a non-breaking change.


## Drawbacks

It feels wrong to make XML (HTML) do dance steps it doesn't otherwise do.

This kind of changes can be seen as further distancing Svelte from the HTML/CSS/JS knowledge base, and that may not be a good thing. 

However, it's mostly (intended to be) a development-level feature. This is the author's only objection, and therefore the RFC, to have a discussion and decision, whether this is aligned with Svelte's general trajectory.


## Alternatives

### Use of current HTML commenting (after tag)

```
<Sample a=10 b=12
  some="42 //"
/>
  <!-- other tbd. -->
  <!-- further=13 - disabled -->
```

The context of such lines is lost, and seems confusing. It can be done, but requires a team convention to know that comments after a tag refer to the tag attributes.

### Preceding attributes with `__` or `--`

This passes Rollup builds, unlike `//`.

```
<Sample a=10 b=12
  some="42 //"
  __other
  __further=13
/>
```

Pros:

- already works; no changes to Svelte needed

Cons: 

- no additional explanation at the end of line can be made
- the attributes are just renamed - they are still there in build output. So this is not really commenting.

### Prior art

The author is not aware of preceding art that introduces within-HTML-tag commenting. References to such are a welcome addition.


## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD?
