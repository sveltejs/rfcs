- Start Date: 2021-07.10
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Preprocessing API rework

## Summary

Introduce a new preprocessing API which is simpler but allows for more flexibility

## Motivation

The current preprocessing API is both a little hard to grasp at first and not flexible enough to satisfy more advanced use cases. Its problems:

- Ordering is somewhat arbitrary, as it runs markup preprocessors first, then script/style. Preprocessors that want to be executed at a different point are forced to do dirty workarounds. It also lead to a PR implementing somewhat of a escape hatch for this (https://github.com/sveltejs/svelte/pull/6031)
- Script/Style preprocessors may want to remove attributes, right now it's not possible to do (unless they become a markup preprocessor and do it themselves somehow) (https://github.com/sveltejs/svelte-preprocess/issues/260, https://github.com/sveltejs/svelte/issues/5900)
- In general, the distinction between markup/script/style forces a decision on the preprocessor authors that may lock them in to a suboptimal solution

The solution for a better preprocessing API therefore should be

- easier to grasp and reason about
- execute preprocessors predictably
- provide more flexibility

## Detailed design

The preprocessor API no longer is split up into three parts. Instead of expecting an object with `{script, style, markup}` functions, it expects a function to which is handed the complete source code, and that's it:

```typescript
result: {
	code: string,
	dependencies: Array<string>
} = await svelte.preprocess(
    (input: { code: string, filename: string }) => Promise<{
			code: string,
			dependencies?: Array<string>,
            map?: any
		}>
)
```

Additionally, `svelte/preprocess` exports new utility functions which essentially establish the current behavior:

### extractStyles

```typescript
function extractStyles(code: string): Array<{
  start: number;
  end: number;
  content: { text: string; start: number; end: number };
  attributes: Array<{
    name: string;
    value: string;
    start: number;
    end: number;
  }>;
}>;

extracts the style tags from the source code, each with start/end position, content and attributes

### extractScripts

Same as `extractStyles` but for scripts

### replaceInCode

```typescript
function replaceInCode(
  code: string,
  replacements: Array<{ code: string; start: number; end: number; map?: any }>
): { code: string; map: any };
```

Performs replacements at the specified positions. If a map is given, that map is adjusted to map whole content, not just the part that was processed. The result is the replaced code along with a merged map.

These three functions would make it possible to reimplement a script preprocessor like this:

```javascript
function transformStuff(...) { /* user provided function */ }
function getDependencies(...) { /* user provided function */ }
function script({code}) {
    const scripts = extractScripts(code);
    const replacements = scripts.map(transformStuff);
    return {
        ...replaceInCode(code, replacements),
        dependencies: getDependencies(replacements)
    }
}
```

Using these three functions, we could also construct convenience functions like `replaceInScript` which would make it possible for preprocessor authors to do `return replaceInScript(code, transformStuff)`. What functions exactly to provide is up for discussion, the point is that we should provide primitives to ensure more flexibility for advanced use cases.

Since preprocessors are now "just" functions, there's no ordering headache anymore, preprocessors are invoked in order, giving full control for composability.

### Roadmap, backwards-compatibility

This new functionality could be implemented in Svelte 3, where the `preprocess` function checks if the passed in preprocessor is an object (current API) or a function (proposed API). In Svelte 4, this would become the default, and we could provide a function for preprocessors that don't support the new API yet.

```javascript
export function legacyPreprocessor(preprocessor) {
  return async ({ code, filename }) => {
    const processedBody = await (preprocessor?.markup({ content, filename }) ??
      Promise.resolve({ code: content }));

    // .. etc
    return mergeParts(processedBody, processedScript, processedStyle);
  };
}
```

## How we teach this

Adjust docs

## Drawbacks

None that I can think of right now

## Alternatives

None that I can think of right now

## Unresolved questions

- We could expand the functionality of `extractScripts`/`extractStyles`. Right now, every script/style is processed, not just the top level ones. Enhance the Svelte parser with a mode that only parses the top level script/style locations, but not its contents?
- What about preprocessors inside moustache tags? Should the Svelte parser be adjusted for an opt-in parsing mode where a Javascript-like syntax for moustache tags is assumed to extract its contents and provide this as another utility function for preprocessing? (https://github.com/sveltejs/svelte/issues/4701)
