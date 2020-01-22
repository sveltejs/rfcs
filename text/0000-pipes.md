- Start Date: 2020-01-22
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Pipes

## Summary

Pipes are a way to write display-value transformations that you can declare in your HTML.
A pipe takes in data as input and transforms it to a desired output

## Motivation

It is a more readable, declarative and reusable way to add in HTML transformation functions like formatting, i18n, array operations etc, matching the WLDM principle  - `write less, do more`.

Currently one can use a reactive binding or an in-place transformation function to get the same result, however, the in-place solution can quickly become hard to read, especially when chained and parameterized

## Detailed design

A pipe is a simple transformation function at its core. This function can be pure or impure. Pipes should be pure by default.
For start, I think only pure functions should be considered.

```js

export function myPipe(input, param1='default1', param2='default2'){
    // do something with input
    return transformedInput;
}

```

#### HTML representation

Inside the interpolation expression, the component variable is run through the pipe operator ( | ) to the pipe function on the right.
`{mytext|lowercase}`.

The pipe can have parameters. Each of these these would be represented as colon operator ( : ) to the right of the pipe function. A pipe can have one or multiple parameters. The order of parameters must be the same as in the transformation function.
`{mydate|date:format:timezone}`

#### Chaining

Pipes can be chained by continuing with pipe operator and the new pipe after the last parameter of the previous pipe.
The order of the chaining matters.


#### In-place assignment

A variable can be populated with the result of an intermediate pipe chaining.
See below the example for filteredCats


#### Compiled output

```html
<p>{foo|bar:param|qux}</p>

```
it is equivalent to 

```html
<p>{qux(bar(foo,param))}</p>

```


#### Examples

Since examples worth a thousand words:

##### Formatting example

Without pipes:

```html
<script>
import {formatDate, formatCurrency, camelize} from 'myUtils';

let mydate= new Date();
let price;
let someText;

$: formattedShortDate = formatDate(mydate,'short')
$: formattedCustomDate = formatDate(mydate,'YYYY-MM-DD HH:mm');
$: formattedPrice = formatCurrency(price,2);
$: formattedText = camelize(someText);

</script>

<p>Short date: {formattedShortDate}</p>
<p>Custom date: {formattedCustomDate}</p>
<p>Price: {formattedPrice}</p>
<p>CamelText: {camelize(someText)}</p>

<!-- or -->

<p>Short date: {formatDate(mydate,'short')}</p>
<p>Custom date: {formatDate(mydate,'YYYY-MM-DD HH:mm')}</p>
<p>Price: {formatCurrency(price,2)}</p>
<p>CamelText: {camelize(someText)}</p>

```

With pipes:

```html
<script>
import {date, currency, camelize} from 'svelte/pipes';

let mydate= new Date();
</script>

<p>Short date: {mydate|date:'short'}</p>
<p>Custom date: {mydate|date:'YYYY-MM-DD HH:mm'}</p>
<p>Price: {price|currency:2}</p>
<p>CamelText: {someText|camelize}</p>

```

##### Array operation example

A simple example used in tables,lists etc 


Without pipes:

```html
<script>
let cats= someArray(100,cat);
let catsPerPage = 10;
let page = 2;
let searchWord;

$: newCats = cats.filter((cat)=>cat.indexOf(searchWord)>=0)
$: newCats = newCats.sort((a,b)=>a.name>b.name);
$: newCats = newCats.slice(page*catsPerPage,(page+1)*catsPerPage);

</script>

<input bind:value={searchWord}>

<ul>
    {#each newCats as { id, name }, i}
		<li><a target="_blank" href="https://www.youtube.com/watch?v={id}">
			{i + 1}: {name}
		</a></li>
	{/each}
</ul>

```

With pipes:

```html
<script>
let cats= someArray(100,cat);
let catsPerPage = 10;
let page = 2;
let searchWord='';
let filteredCats=[];
</script>

<input bind:value={searchWord}>

<ul>
    {#each cats|filter:{searchWord}|orderBy:name|limit:{catsPerPage}:{page} as { id, name }, i}
		<li><a target="_blank" href="https://www.youtube.com/watch?v={id}">
			{i + 1}: {name}
		</a></li>
	{/each}
</ul>

<!-- or, with in-place assignment -->

<ul>
    {#each (filteredCats = cats|filter:{searchWord})|orderBy:name|limit:{catsPerPage}:{page} as { id, name }, i}
		<li><a target="_blank" href="https://www.youtube.com/watch?v={id}">
			{i + 1}: {name}
		</a></li>
	{/each}
</ul>

<p>{filteredCats.length} cats found</p>

```

##### Translation example

```html
<script>
let dynamicText;
</script>

<p>{'statictext'|translate}</p>
<p>{dynamicText|translate}</p>
```

## How we teach this

A new API page explaining the concept and some example would be sufficient.
Anybody with with previous Angular experience would feel natural.

## Drawbacks

It is a new feature, it should not impact any existing Svelte apps


## Alternatives

This concept is shamelessly stolen from Angular (pipes) or AngularJS (filters) and it is battle tested, imho.


## Unresolved questions

- whether should consider impure pipes, like an async pipe
- related to the static content compilation, this could alleviate some issues, because the pipes are pure functions, and they should not be further checked. TBD
```html
<script>
  export let name;
</script>

<h1>Hello {name|translate|lowercase}!</h1>

<!-- Greeting -->

<Greeting name="world"/>

```
