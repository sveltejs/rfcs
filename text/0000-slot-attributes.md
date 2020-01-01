- Start Date: 2020-01-01
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Slot Attributes

## Summary

Slot attributes allow a Component to assign attributes (eg, classnames) to the content of a slot if & when it holds content.

## Motivation

> I'll be using Bootstrap to help illustrate examples for this RFC, but the problem is **most definitely** not limited to Bootstrap by any means. I've run into this in _every_ design system I've worked with. Bootstrap is chosen solely for its familiarity and simplicity.

The majority of design systems have conditional content slots for their elements – we know & expect this. However, when a Svelte developer goes to wire up the UX slots to actual `<slot/>`s, they'll run into problems:

#### 1. Slot elements generally have CSS side-effects when empty.

Most of these slots' act as parent wrappers for what they contain. This means that there's almost always some level of `padding`, `background-color`, `margin`, etc applied to that parent, even when it contains no children. This, then, prevents the Svelte developer from including those parent containers within the Component directly:

```html
<div class="card">
	<div class="card-header">
		<slot name="header"/>
	</div>

	<div class="card-body">
		<slot />
	</div>

	<div class="card-footer">
		<slot name="footer"/>
	</div>
</div>
```

> [See REPL example](https://svelte.dev/repl/e73cff770b274fe89117608232277145?version=3.16.7)

As you will see, the `.card-header` and `.card-footer` carry their own paddings & background colors, rendering a visual effect even though they weren't defined/used by the Card's consumer.

Y Self-contained component styles
Y Does not require internal knowledge
Y&N Slot is configurable from consumer POV
N Slot leaves no trace when unused/empty.


| Pass/Fail          | Requirement                             |
|--------------------|-----------------------------------------|
| :white_check_mark: | Self-contained component styles         |
| :white_check_mark: | Does not require internal knowledge     |
| :wavy_dash:*       | Slot is configurable from consumer POV  |
| :no_entry_sign:    | Slot leaves no trace when unused/empty  |

> <sup>*</sup> You can't add additional classes to `.card-footer` (ideal) but still can to internals

This then forces us into the second problem...

#### 2. External `<slot/>` access requires internal knowledge

In order to truly conditionally render parent classes (eg, `card-header`) we, the `<Card>` consumers, have to know _how_ to recreate the `<slot name=header/>` in a way that the design element (eg, Bootstrap's `.card`) expected.

What this means is that we have to have internal knowledge of the Component's design mechanics AND it forces us to split the Component's markup across multiple boundaries.

Additionally, this forces `<Card>` to ship `:global(.card-footer)` style selectors since it does not _actually contain_ any `.card-footer` elements in its markup.

```html
<Card class="text-center">
	<!-- requires knowledge of "card-header" -->
	<div slot="header" class="card-header">Featured</div>

	<h5>Special title treatment</h5>
	<p>With supporting text below as a natural lead-in to additional content.</p>
	<button type="button" class="btn btn-primary">Go somewhere</button>

	<!-- requires knowledge of "card-footer" -->
  <div slot="footer" class="card-footer text-muted">
    2 days ago
  </div>
</Card>
```

> [See REPL example](https://svelte.dev/repl/64594c7de23e4b1fa4a123f15ec479e2?version=3.16.7)

| Pass/Fail          | Requirement                             |
|--------------------|-----------------------------------------|
| :no_entry_sign:    | Self-contained component styles         |
| :no_entry_sign:    | Does not require internal knowledge     |
| :white_check_mark: | Slot is configurable from consumer POV  |
| :white_check_mark: | Slot leaves no trace when unused/empty  |


## Detailed Design

A `<slot>` can hold attributes like any other tag.<br>
In English, this is the same as saying "this slot's (top-level) children _will_ inherit these attributes". This is true for named & unnamed slots alike.

```html
<!-- Card.svelte -->
<div class="card">
	<!-- when "header" is defined, it gain the "card-header" class -->
	<slot name="header" class="card-header" />

	<!-- when unnamed content is received, it gains the "card-body" class -->
	<slot class="card-body" />

	<!-- when "footer" is defined, it gain the "card-footer" class -->
	<slot name="footer" class="card-footer" />
</div>
```

> **Note:** Example is restricted to `class` for simplicity.

As mentioned, the slot's attributes will be coerced/assigned to the incoming content.
In this example, we have not defined any of our own `class` values, so it's a direct assignment:

```html
<!-- input -->
<Card>
	<!-- define my header slot -->
	<div slot="header"><h4>My title</h4></div>

	<!-- pass default/body content -->
	<div>
		<h5>Special title treatment</h5>
		<p>With supporting text below as a natural lead-in to additional content.</p>
		<button type="button" class="btn btn-primary">Go somewhere</button>
	</div>

	<!-- define my footer slot -->
	<div slot="footer">2 days ago</div>
</Card>

<!-- output -->
<div class="card">
	<div class="card-header"><h4>My title</h4></div>
	<div class="card-body">
		<h5>Special title treatment</h5>
		<p>With supporting text below as a natural lead-in to additional content.</p>
		<button type="button" class="btn btn-primary">Go somewhere</button>
	</div>
	<div class="card-footer">2 days ago</div>
</div>
```

Now, when a `<slot/>` is unused, it has nothing to spread its attributes onto:

```html
<!-- input -->
<Card>
	<div>
		<h5>Special title treatment</h5>
		<p>With supporting text below as a natural lead-in to additional content.</p>
		<button type="button" class="btn btn-primary">Go somewhere</button>
	</div>
</Card>

<!-- output -->
<div class="card">
	<div class="card-body">
		<h5>Special title treatment</h5>
		<p>With supporting text below as a natural lead-in to additional content.</p>
		<button type="button" class="btn btn-primary">Go somewhere</button>
	</div>
</div>
```

And finally, _when there are attribute conflicts_, we simply merge the `<slot>` values into the consumer values.<br>
Again, the Component author has said "this `<slot/>` needs to have these attributes" – anything else is presumably relevant to the Consumer-side:

```html
<!-- input -->
<Card>
	<!-- define my header slot w/ extra attr -->
	<div slot="header" aria-label="My title" ><h4>My title</h4></div>

	<!-- pass default/body content w/ extra class -->
	<div class="text-center">
		<h5>Special title treatment</h5>
		<p>With supporting text below as a natural lead-in to additional content.</p>
		<button type="button" class="btn btn-primary">Go somewhere</button>
	</div>

	<!-- define my footer slot w/ extra class -->
	<div slot="footer" class="text-muted">2 days ago</div>
</Card>

<!-- output -->
<div class="card">
	<div class="card-header" aria-label="My title"><h4>My title</h4></div>
	<div class="card-body text-center">
		<h5>Special title treatment</h5>
		<p>With supporting text below as a natural lead-in to additional content.</p>
		<button type="button" class="btn btn-primary">Go somewhere</button>
	</div>
	<div class="card-footer text-muted">2 days ago</div>
</div>
```

## How we teach this

This would be a simple add-on tutorial for the Svelte website.

In plain English, we say "Any attributes (except `name`) on a defined `<slot/>` will be applied to its top-level children"

## Drawbacks

These are pretty lame, but it's all I could muster:

1) There's a built-in exception of `name` not being passed down – reserved for slot identification.
2) This becomes an additional API to learn (lol)
3) This is unique to Svelte – the closest is a React HoC using a fairly lengthy `cloneElement` loop to add properties to children.

## Alternatives

Current alernatives and workarounds are:

1) Do nothing – expect Component consumers to know which classes need to appear when defining the slot:

	```html
	<Card>
		<div slot="header" class="card-header">
			<h4>My title</h4>
		</div>
		<div slot="body" class="card-body">
			<p>My content</p>
		</div>
		<div slot="footer" class="card-footer">
			<Button>Submit</Button>
		</div>
	</Card>
	```

2) Extract a Component's slots as separate Sub-Components:

	```html
	<Card>
		<CardHeader>
			<h4>My title</h4>
		</CardHeader>
		<CardBody>
			<p>My content</p>
		</CardBody>
		<CardFooter>
			<Button>Submit</Button>
		</CardFooter>
	</Card>
	```

3) Limit all slotted content to plain text:

	This is a bit extreme – and a no-go for most designers – but it's possible to get what we want by replacing all `<slot>`s with props and a series of `{#if}` blocks.

	It semi-works for this example, but still requires one of the other two workarounds for non-plaintext content.

	```html
	<Card title="My title">
		<p>My content</p>
		<CardFooter>
			<Button>Submit</Button>
		</CardFooter>
	</Card>
	```


## Unresolved questions

* Which attributes, if any others besides `name`, should not be allowed?
