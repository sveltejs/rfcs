- Start Date: 2020-12-13
- RFC PR: ([#44](https://github.com/sveltejs/rfcs/pull/44))
- Svelte Issue: (leave this empty)

# Local Components

## Summary

Allow `.svelte` files to define multiple Components

## Motivation

Each Component requires creating a brand new `.svelte` file, but sometimes a Component necessitates its own sub-Components, this happens in the following occasions :
1. a node tree pattern is repeated several times 
2. a node tree needs to be wrapped or not depending on a condition
3. a node uses completely different attributes/directives depending on a condition
4. a variable from a `let:` slot prop, an `#await` resolved promise or a `#each` value is transformed then used more than once
5. an `#each` block needs its own script for `$`subscriptions or lifecycle

Having to spread tiny parts of a Component across several files is unfortunate as it leads to duplicate component names, cluttered folders and quite the refactoring hazard

To solve those, @Rich-Harris has put forth a collection of RFCS: 
* #32 to allow nesting styles in node trees
* #33 to allow declaring `const`s in node trees
* #34 to allow declaring reusable node trees

Now all of those combined barely make up for a pseudocomponent, yet they introduce and bend several concepts in the framework, and issue 5) would still require an external component

**_Each one of our issues can be and are currently solved by making a new Component_**, we just don't like spreading them across several files

## Detailed design

A Svelte Component is composed of a script block, a style block and a node tree

A `.svelte` file is composed of a script context module and a Svelte Component that becomes the default export

```xml

<script context="module"/>

<script/>
<style/>
<content/>

```

The proposed design is simple: `.svelte` files may define named Svelte Components in series using a top-level Comment block. 

```xml
<script context="module">

<script/>
<style/>
<content/>

<!-- Foo -->
<script/>
<style/>
<content/>

<!-- Bar -->
<script/>
<style/>
<content/>
```
Everything above the first named Component (if any) stays the default export.

The `Foo` variable works exactly as if that file imported `import Foo from "./Foo.svelte"`, so `Foo` cannot access anything from the default Component,  it's all on its own with its own style, script and content.

The only thing that is different from a conventional import is that `Foo` has access to the script module and can use the other named Components present in the file if any.


## How we teach this

This RFC does not introduce any kind of directive, special element or new concept. 

If your component needs to be split into parts, just write it underneath.

## Drawbacks

Technically breaking

## Unresolved questions

* Is there something better than a Comment Node ? 
* Should the comment node include a specific word to better describe its intent ? `<!-- declare Foo -->` 
* Should local components be directly exportable ? `<!-- export Foo [as Bar] -->`
* Given that each component scopes its style, should there be a `<style context="module">` shared across all components in the same file ?
