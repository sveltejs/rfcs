- Start Date: 2020-12-09
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Style Directive

## Summary

Add a `style:` directive:

```svelte
<div style:position="absolute" style:left="67px" />
```

## Motivation

Currently, when inline style attributes need to be set based on multiple variables, the construction of a style attribute string is left to the developer.

Constructing these attribute strings can become somewhat clumsy.

For [example](https://github.com/mhkeller/layercake/blob/master/src/LayerCake.svelte#L297-L301):

```svelte
<div
  style="
    position:{position};
    {position === 'absolute' ? 'top:0;right:0;bottom:0;left:0;' : ''}
    {pointerEvents === false ? 'pointer-events:none;' : ''}
  "
/>
```

It would be useful — and often less error-prone — to be able to set multiple inline styles without having to construct the attribute string:

```svelte
<div
  style:position="absolute"
  style:pointer-events={pointerEvents}
  style:left="{left}px"
/>
```

## Detailed design

Style directives would take the form `style:property="value"` where `property` is a CSS property and `value` is a CSS property value.

### Shorthand

A shorthand form could also exist when a variable name and CSS property name match:

```svelte
<script>
  const left = "234px";
</script>
<div style:left />
```

### Conflicting Styles

One question surrounding this approach how to combine style directives with style attribures that have conflicting properties.

Consider the example:

```svelte
<div style="position: absolute; left: 40px;" style:left="50px" />
```

What would be the value of this element's `left` property?

It seems there are three options available:

1. Conflicting CSS properties in style directives are not allowed, and a compiler error is thrown.
2. The style attribute takes precedence.
3. The style directive takes precedence.

The third option seems like the most reasonable, and most closely matches the behavior of the `class:` directive.

For example, class directives take precedence over class attribute strings:

```svelte
<div class="foo bar" class:bar={false}>
  My class is "foo" and not "foo bar"
</div>
```

### CSS Variables

These directives could also contain CSS variables:

```svelte
<div style:--border-color="saddleBrown" />
```

## How we teach this

This is pretty similar to the `class:` directive, except that a CSS value is used instead of a boolean.

## Drawbacks

As with any feature, this would add some complexity to the codebase, and add a bit of learning curve, although the impact seems minimal.

We may want to discourage the construction of complex inline styles based on variables, and encourage the use of CSS variables instead. 
Of course, dynamic CSS variables would also need to be passed into a style attribute. 

## Alternatives

React uses a style object:

```jsx
<div style={{ marginLeft: "30px" }} />
```

We could instead mimic React and allow this sort of object to be passed to an element's `style` attribute.

## Unresolved questions

### Camel Case Property Names?

We could also allow for the use of camel case property names. 

This would make using variable names that match CSS properties a bit easier:

```svelte
<script>
  const marginLeft = "20px";
</script>
<div style:marginLeft />
```

