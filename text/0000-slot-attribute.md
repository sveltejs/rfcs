- Start Date: 2020-03-01
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Rules for slot attribute and `let:` directive

## Summary

This RFC propose rules for `slot` attribute and `let:` directive.

## Motivation

Currently, `slot` attribute and `let:` directive can be used freely, which causes some edge cases bugs like: [#4135](https://github.com/sveltejs/svelte/issues/4135).

The following proposes some new rules for using `slot` and `let:`

## Detailed design

### 1. No nesting of `slot` attribute:

Currently this is a valid syntax:

```html
<Component>
  <div slot="a">
    <div slot="b">
    </div>
  </div>
</Component>

<Component>
  <div> <!-- assumed as "default" slot -->
    <div slot="b">
    </div>
  </div>
</Component>
```

Should warn against nesting usage of `slot` attribute.

### 2. Disallow more than 1 named slot of the same name

Currently this is a valid syntax:

```html
<Component>
  <div slot="a">1</div>
  <div slot="a">2</div>
</Component>
```

The current behavior is to render both divs into the same named slot:

```html
<!-- <slot name="a"> -->
<div>1</div>
<div>2</div>
<!-- </slot> -->
```

The complication arise when using `let:` binding in this scenario

```html
<Component>
  <div slot="a" let:a={a}>{a}</div>
  <div slot="a" let:a={b}>{b}</div>
</Component>
```

[REPL](https://svelte.dev/repl/b714180a1feb44f7be6348d75374d689?version=3.19.1).

Currently both `div`s are currently created by the same fragment, sharing the same slot scope. This lead to cases with bugs due to conflicting slot scope:

```html
<Component>
  <div slot="a" let:a={b} let:b={a}>b: {a} a: {b}</div>
  <div slot="a" let:a={a} let:b={b}>a: {a} b: {b}</div>
</Component>
```
[REPL](https://svelte.dev/repl/af6949665963491e94732fa5590f0810?version=3.19.1)

So, 1 quick way of eliminating such edge cases is to disallow declaring mutliple same named slot.

As a workaround for the need of having more than 1 root nodes for the same slot, we can introduce the `<svelte:slot>` as a wrapper:

```html
<Component>
  <svelte:slot slot="a" let:a={a} let:b={b}>
    <div>{a}</div>
    <div>{b}</div>
  </svelte:slot>
</Component>
```

### 3. Disallow `slot="default"`

Currently we allow
```html
<Component>
  <div slot="default">1</div>
</Component>
```

which is equivalent to

```html
<Component>
  <div>1</div>
</Component>
```

However, 

```html
<Component>
  <div>1</div>
  <div>2</div>
</Component>
```

is valid, but

```html
<Component>
  <div slot="default">1</div>
  <div slot="default">2</div>
</Component>

<!-- or -->
<Component>
  <div>1</div>
  <div slot="default">2</div>
</Component>
```

will violate rule #2, so, an easier way out is to disallow `slot="default"`.

### 4. `let:` binding can only be used in Component or element with `slot` attribute

Currently, `let:` binding can be used anywhere, creating new slot scopes but actually has no effect.

```html
<Component let:a>
  <div slot="a" let:b>
		<div let:c>
			{a} {b} {c}
		</div>
	</div>
</Component>
```

[REPL](https://svelte.dev/repl/bd54399c10704b5ca1e414f307593fdd?version=3.19.1)

It is arguably a bug, so we should disallow it.

With this, it is invalid to use `let:` for default slot

```html
<Component>
  <div let:a>{a}</div>
</Component>
```

and should use the following instead:

```html
<Component let:a>
  <div>{a}</div>
</Component>

<!-- or -->

<Component>
  <svelte:slot let:a>
    <div>{a}</div>
  </svelte:slot>
</Component>
```

## How we teach this

The current slot behaviors create edge cases which can be prevented with the rules proposed.

We should provide meaningful compile errors to prevent them.

## Drawbacks

-

## Alternatives

-

## Unresolved questions

TBD?