- Start Date: 2020-01-01
- RFC PR: [#16](https://github.com/sveltejs/rfcs/pull/16)
- Svelte Issue: (leave this empty)

# Props as Directives

## Summary

Should we allow a Component's directives to be dynamic/configured via an exported prop?

## Motivation

Specifically, the `transition:` directive sparked this RFC.<br>
It would be amazing if we can allow Component consumers to configure transitions for the Components that allow them.

However, I figure that once we allow `transition:` to be dynamic, it may implicitly require we allow all other directives (included or custom) to be dynamic, too.

## Detailed design

Let's assume I have an `<Alert/>` component that wants to run some kind of transition when it enters & leaves the page.

For now, the `<Alert/>` will use a `fade` animation from the `svelte/transition` group, which (currently) is not available for configuration:

```html
{#if show}
<div transition:fade={settings} role="alert">
	<button type="button" class="close" on:click={onClose}>
		<span aria-hidden="true">×</span>
	</button>
	<slot/>
</div>
{/if}

<script>
	import { fade } from 'svelte/transition';

	export let show = true;
	const settings = {
		duration: 300,
		delay: 0
	};
</script>
```

We can export the `settings` object, which allows the Consumer to adjust the `fade` transition:

```diff
-const settings = {
+export let settings = {
```

...however, the Consumers are still stuck with `fade`! Boring.

Svelte includes (and allows) so many more interesting effects, and it's a shame that the Consumer isn't able to plug them into their `<Alert/>` usage :cry:

Ideally, they could, and it'd look like this:

```html
<!-- Alert.svelte -->
<script>
	export let show = true;
	// now NO transition by default
	export let transition = false;
</script>

{#if show}
<div transition:{transition} role="alert">
	<button type="button" class="close" on:click={onClose}>
		<span aria-hidden="true">×</span>
	</button>
	<slot/>
</div>
{/if}

<!-- Consumer.svelte -->
<script>
	import { Alert } from 'somewhere';
	import { fly } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';

	let visible = false;
	const transition = fly({
		delay: 250,
		duration: 300,
		x: 100,
		y: 500,
		opacity: 0.5,
		easing: quintOut
	});
</script>

<Alert {show} {transition}>
	<p>Custom Alert that flies in/away on visibility toggle!</p>
</Alert>
```


## How we teach this

We'd add a new chapter (or two) to the Svelte tutorials for props-based transitions and/or animations.

Since it's an opt-in behavior for existing APIs – and non-breaking – the barrier is pretty low.

## Drawbacks

* Begs the question of allowing _all_ directives to be configurable
* Probably requires new/different combination of curly brackets


## Alternatives

* Allow Components to receive `transition:`/directives directly.<br>However, this is currently disallowed since it generates more questions.

* Add a new, special directive which becomes the *only* one capable of receiving & applying other directives.<br>I think this is a bigger task?

## Unresolved questions

* Should we only allow specific directives?<br>_Eg, `transition`, `animate`, `in`, and `out`_
