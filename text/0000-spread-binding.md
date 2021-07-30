- Start Date: 2021-7-30
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Spread Binding

## Summary

This RFC proposes a way to bind all properties inside an object, similar to the existing [spread syntax](https://svelte.dev/tutorial/spread-props).

```sv
<Component bind:...={object} />
```

## Motivation

The below motivators inherit any ones that spread props had, as it's similar in function.

### Bind Forwarding

One of the main motivations of this would be to support bind forwarding ([#2226](https://github.com/sveltejs/svelte/issues/2226)). The need for bind forwarding comes up a lot, especially for components that "extend" other components. This can be seen in places like [carbon-components-svelte](https://github.com/carbon-design-system/carbon-components-svelte/blob/master/src/Button/Button.svelte) where they'd use the `@extends` JSDoc keyword for props, but still have to type out each prop explicitly on the component that encapsulates it. Problems arise when the base component has default values or changes in a way that may not be immediately incompatible with the encapsulator. Such scenarios may include:

#### The base component switching to more restrictive types

```sv
<!-- Base.svelte -->
<script lang='ts'>
	// old
	// export let foo: Record<string, boolean>;
	
	// new
	export let foo: { a: boolean };
</script>
```

```sv
<!-- Encapsulator.svelte -->
<script lang='ts'>
	import Base from './Base.svelte'
	
	// is now too permissive
	export let foo: Record<string, boolean>;
</script>

<Base {foo} />
```

#### The base component deprecating an old prop & switching to a new prop

```sv
<!-- Base.svelte -->
<script>
	/** @deprecated Use bar instead */
	export let foo;
	
	export let bar;
</script>
```

```sv
<!-- Encapsulator.svelte -->
<script>
	import Base from './Base.svelte'
	
	// the `@deprecated` directive isn't passed to the user
	export let foo;
</script>

<!-- `bar` isn't passed to the component --> 
<Base {foo} />
```

#### The base component switching to a different default value while "child" supplies a different default

```sv
<!-- Base.svelte -->
<script>
	// old
	// export let foo = true;
	
	// new
	export let foo = false;
</script>
```

```sv
<!-- Encapsulator.svelte -->
<script>
	import Base from './Base.svelte'
	
	// This default is now mismatched
	export let foo = true;
</script>

<Base {foo} />
```

An implementation of bind forwarding will enable a developer to replace uses of regular spread in the encapsulator with bound spreads, as any props passed into the encapsulator (`var={var}`) not specified with the bind directive (`bind:var={var}`) will simply not bind & remain unreactive. (See [Sxxov/svelte-spread-bind-test-case](https://github.com/Sxxov/svelte-spread-bind-test-case) for an example)

### Store Abuse/Object of Props

Another big reason is to alleviate patterns of "store abuse" — where developers might pass in single stores containing many prop values — as well as ease situations where objects are passed in from outside of the context of svelte containing changing props. The current pattern would require a developer that would want to support certain reactive properties of an object to do as such:

```sv
<!-- Adapted from https://svelte.dev/tutorial/spread-props -->
<script>
	import Info from './Info.svelte';

	export let pkg = {
		name: 'svelte',
		version: 3,
		speed: 'blazing',
		website: 'https://svelte.dev'
	};
</script>

<Info
    bind:name={pkg.name}
    bind:version={pkg.version}
    bind:speed={pkg.speed}
    bind:website={pkg.website}
/>
```

The current solution to this would be to use the Context API & stores. Such approach might be a great fit for situations where data might be coming from a set place not requiring two-way binding, but it falls short in the context of passing data from the encapsulator component to the base component.

#### The single store approach

##### Data being bound to a store/namespace

The data will not be able to be detached from a store, else it will lose its two-way binding.

```sv
<script>
	const store = writable({ foo: 1, bar: 1 }); // can be replaced with a `getContext` call
	const { bar } = $store;
	
	++$store.foo;	// `$store.foo === 2`
	++bar;			// `$store.bar === 1`, but `bar === 2`
</script>
```

##### Component instance contexts

If data is generated/dependent on an instance of a component, the antipattern of passing a key to the context or a only single store, instead of actual props, will make the resulting code take much more time to decipher. This is exacerbated by non SvelteTS projects, as there is no way to assert a clear structure of what props are expected, furthermore preventing Intellisense from working correctly. (JSDoc may be used, but using it to type every prop doesn't sound fun)

```sv
<script>
	export let store = writable({ foo: 1, bar: 1 }); 
	// // or
	// export let key;
	// const { store } = getContext(key);
	
	++$store.foo; // this comes with the disadvantages of "Data being bound to a store/namespace" as well
</script>
```

#### The multiple store approach

#### Unnecessary store creation

If a developer would to create a component that only takes in stores, then the consumer of the component would have to create a store every time some  data is passed into the component, even if the data is static.

```sv
<!-- Component.svelte -->
<script>
	export foo = writable(1); // can be replaced with a `getContext` call
	
	++$foo; // the dollar would need to be used every time as well
</script>
```

```sv
<!-- Consumer.svelte -->
<script>
	import Component from './Component.svelte';
	
	// // example `setContext` call
	// setContext(key, writable(100) /* instead of just `100` */);
</script>

<Component foo={writable(100) /* instead of just `100` */} />
```

#### Duplicated props

The antipattern solution to the unnecessary store creation would be to create a "supplementary" prop that takes in a store. This breaks `bind` directives for the original variable, creates an unnecessary store anyways if the value is static, & cannot be used with the Context API.

```sv
<!-- Component.svelte -->
<script>
	export foo = 1;
	export fooW = writable(foo);
	
	++$fooW;
</script>
```

```sv
<!-- Consumer.svelte -->
<script>
	import Component from './Component.svelte';
</script>

<Component foo={100} />
<!-- 
	// or
	<Component fooW={writable(100)} />    
-->
```



## Detailed design

### Technical Background

From my research it seems that none of the major frameworks include this feature. The reasons might just be that only svelte implements two-way-binding in a style that makes sense for spread bindings, or simply that React/Redux's popularization of strict one-way-binding has limited its usefulness (eg. LitElement's removal of  `{{foo::bar}}`).

I've found tonnes of issues discussing something like this feature though, with some including attempts to do this currently in Svelte. (A few non-duplicates — [#5106](https://github.com/sveltejs/svelte/issues/5106) [#5137](https://github.com/sveltejs/svelte/issues/5137) [#2226](https://github.com/sveltejs/svelte/issues/2226))

### Implementation

Syntax was a key consideration in this proposal, as the `bind` directive is designed in a way that crosses the boundaries of both HTML & JS. A few examples of this coming in as a blocker is noted here by [Florian-Schoenherr](https://github.com/Florian-Schoenherr) in [#5137](https://github.com/sveltejs/svelte/issues/5137) discussing spread binding itself:

> * `bind:props={...props}` would make `props` reserved
> * `bind:{...props}` doesn't look like correct syntax
> * maybe `bind:$$props={...props}`?
>
> Basically, `bind:???={...props}`

From my deliberation, I think the best way to implement this within the constraints of semantics (anything inside `{}` should be mostly-valid JS & anything HTML should look close enough), would be truly be the last option in the list given by Florian. More specifically:

#### `bind:...={object}`

The largest reason this syntax was chosen was to avoid the pitfalls of [`bind:{...object}`](#bind...object). While it does make sense why it would be said syntax, the quirks it would require to fit in made me believe like this was a safer choice. It follows the current `bind` rules of a target & a source (`bind:target={source}`). Plus, `bind:...` is technically a valid XHTML attribute name, & `{source}` will be the same as the current `bind` directive's expected value.

A [variant](https://github.com/Sxxov/svelte) of this has been bodged together by me a couple months ago, with a different syntax ([`{...bind:object}`](#...bindobject)), but the main implementation strategy can be transferred over. 

##### What'll happen when the compiler finds a `bind...=`:

1. Assert `{object}` to be either an `Identifier` or a `MemberExpression` (like how `bind:object={object}` currently works).
2. Mark `object` as a spread bind

##### What'll happen after lexing/parsing:

1. For each spread bind, emit runtime code to loop through its properties & trigger subsequent callbacks, when it's marked as dirty

##### What'll happen during runtime:

1. If a spread bound variable or its properties are marked as dirty, loop through its properties & trigger subsequent callbacks.

One caveat however. With purely this implementation there'll still be one thing left unsolved gracefully — bind forwarding. This is because both `$$props` & `$$restProps` are immutable. We may get around this with some boilerplate, but this is not really practical to use everywhere bind forwarding has to be used. A utility function can't be made out of this either, as it uses reactive statements.

```sv
<!-- 
	Taken from https://github.com/Sxxov/svelte-spread-bind-test-case/blob/master/src/B.svelte
	Syntax modified to fit current decision.
-->
<script>
	import C from './C.svelte';
	
	let baz = 3; // baz was abstracted out from A.svelte
	$: restProps = $$restProps;
	$: restProps, updateRestProps(restProps);
	function updateRestProps(restProps) {
		Object.entries(restProps).forEach(([key, value]) => {
			$$restProps[key] = value;
		});
	}
</script>

<C bind:baz bind:...={restProps} />
```

Something that might work though, would be to add an exception to `$$props` & `$$restProps`, where if the compiler finds them being used as the bound variable it would generate this boilerplate, or more elegantly mark that instance of `$$props`/`$$restProps` as mutable while retaining the compiler error everywhere else (to prevent mutation not caused by `bind`).

This might look like this:

##### What'll happen after lexing/parsing:

1. For each spread bind, if the identifier is `$$props` or `$$restProps`, emit code to make a mutable copy of them to work on
2. ...

##### What'll happen during runtime:

1. If the identifier is `$$props` or `$$restProps`, make a mutable copy
2. ...

Regarding types, language tools implementers may follow how `$$props` & `$$restProps` currently behave (not exposing their types without explicitly declaring them), or they could recognize that this new spread `bind` directive is being used & merge the type of the bound variable onto the props, which would be especially useful for SvelteTS projects.

#### Solving the motivating problems

With everything above implemented, developers would be able to simply do the following to replace the examples given above:

##### Bind forwarding

```sv
<!-- Base.svelte -->
<script>
	export let foo = 1;
	export let bar = 2;
	export let baz = 3;
</script>
```

```sv
<!-- Encapsulator.svelte -->
<script>
	import Base from './Base.svelte'
</script>

<Base
	baz={30}
	bind:...={$$restProps}
/>
```

```sv
<!-- EncapsulatorConsumer.svelte -->
<script lang='ts'>
	import Encapsulator from './Encapsulator.svelte'
	
	let foofoo = 10;
	let barbar = 20;
</script>

<Encapsulator
    bind:foo={foofoo}
    bar={barbar}
/>
```

###### Result

| Variable | Bound to                                | Final Value |
| -------- | --------------------------------------- | ----------- |
| foo      | foofoo (at EncapsulatorConsumer.svelte) | 10          |
| bar      | n/a                                     | 20          |
| baz      | n/a                                     | 30          |

##### Store Abuse / Object of Props

```sv
<!-- Adapted from https://svelte.dev/tutorial/spread-props -->
<script>
	import Info from './Info.svelte';

	export let pkg = {
		name: 'svelte',
		version: 3,
		speed: 'blazing',
		website: 'https://svelte.dev'
	};
</script>

<Info
    bind:...={pkg}
/>
```

## How we teach this

A teacher could convey this concept as "spread props but two way bound". It has similar behaviour to a simple spread, but enables all properties of an object to be changed by a "child" component.

To new users, this concept can be shown by giving an example of inter-component communication when given an object, like the [spread props REPL](https://svelte.dev/tutorial/spread-props).

To existing users, this concept can be presented from either the perspective of bind forwarding or the "object problem", like above motivations.

Regarding the bigger picture, this should not violate nor affect any other part of Svelte, as this can live in it's own isolated bubble of an extra feature. The largest difference this change might make would be to developers who are writing components that "extend" each other, as well as ones who are creating components that are consumed by others. They would be able to change parts of their code to reduce code duplication, as well as be not as aggressive when it comes to component splitting. This ability will already have been communicated via the "motivation problems" above.

## Drawbacks

The only drawback I'm able to come up with is the possible increase of abuse for two-way-bindings. Two-way-bindings when written incorrectly can produce unclear code, & this proposal could be an avenue that pushes the usage of the `bind` directive even when it's not needed.

However, if the dangers of overuse can be communicated properly in the tutorial material, or if this feature is presented as an "advanced" feature similarly to the Context API, such cases might be mitigated.

## Alternatives

#### `bind:{...object}`

`bind` currently expects a prop name as the value after the colon & an optional `=` with a variable as the value after that. This is similar syntax to providing normal props. Following the syntax of that, `bind:{...object}` would be the most obvious syntax to developers just poking around. However, ignoring how this doesn't look & feel like valid HTML semantically, this would still violate the current implementation's rule of a `bind` requiring a variable name to be resolved (or more specifically according to the compiler: "an identifier (e.g. `foo`) or a member expression (e.g. `foo.bar` or `foo[baz]`)"). The spread syntax enables a developer to pass in (virtually any) expression to act as a target to be spread (as `{}` would suggest).

A solution to the problem of `bind:{...object}` accepting an expression, would be to fall back to a regular spread directive whenever an "anonymous" object is provided, as the provided return value would be an object that's not bind-able. However, the developer might get confused & assume that providing props within such object would facilitate binds anyways. The only reason we might be able to get away with this would be that the return result of IIFEs are not accepted by spread props, thus turning the syntax into somewhat of a pseudo-expression after `...`. While nested rests are accepted in spread props, developers might be more open to understanding why a copy of an object would not be bound as well.

However with all that said, If we would to go with simply retaining the current limitation of `bind`'s value (blocking anything not a variable), we'd be free to use this syntax.

#### `...bind:object`

This might look correct on first sight, but it requires the developer to use shorthand, as supplying `...bind:object={object}` would simply force us to ignore the `:object` part.

#### `...bind={object}`

This is semantically valid HTML, but it goes against `bind`'s design quite a lot, as bind expects a `:` & the `:` is present everywhere else bind is used.

#### `{...bind:object}`

This is the first syntax I came up with & is the one present in the test case. It's not valid JS, just putting it here for completeness.

## Unresolved questions

1. Which syntax would be the true preferred one.
2. How or whether the compiler should react to spread binding `$$props` & `$$restProps`.
3. (For `bind:{...object}`) If the compiler should error on a non-identifier/member expression, or fallback to regular spread behaviour.
