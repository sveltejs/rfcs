- Start Date: 2020-10-10
- RFC PR:
- Svelte Issue:

# An evolution of the current reactivity system

## Summary

Improving the reactivity system, an evolution not a revolution:

- Improve predictability
- Improve stability

## Motivation

Svelte reactivity system is invisible, even considered "magic", when it doesn't
work as expected, it's very hard to debug.

In the current implementation, reactive statements are not always run or are
not run with the latest values, this depends on the ordering inside the
generated code [REPL](https://svelte.dev/repl/cbd9a4bb8c5f43c89fa61ffc2ace2e20?version=3.29.4):

```html
<script>
  let amount = 2;
  let price = 1.99;
  let total = 0;

  // $: formattedTotal doesn't work when placed above $: calculate
  $: formattedTotal = total.toFixed(2);
  $: calculate(amount);

  function calculate(n) {
    total = price * n;
    console.log("calculated " + total);
  }
</script>

{price} x <input type="number" bind:value="{amount}" /> = {formattedTotal}
```

While trying fix this single issue, i've noticed other areas that could be
improved and created this RFC to collect thoughts and bounce ideas.

### Improve predictability

A reactive statement must run (at least once) when one or more of its
dependencies change (even when the change happens during another reactive
statements).

Notify users when the svelte runtime detects that it's unable to process the change.

### Improve stability

With the current implementation a single exception inside a reactive statement crashes all svelte components on the page.

Prevent infinite reactivity loops.

## Detailed design

This chapter is split into three parts:

- Terminology explaining the terms
- Current design and its drawbacks and some suggestions
- Proposal for an improved Responding phase.

### 1. Terminology

**Reactive property**: Anything that will trigger the reactivity system. (`$store`, `state = ...`, `$: ...` combined)  
**User**: A developer which uses the svelte framework  
**Userland**: Code written by users  
**Phase**: A step in the reactivity system  
**(Dirty) flag**: A boolean-like indicator that an property was changed  
**(Dirty) mask**: A selection of dirty flags

Phases:

- **Init**: Component is just created `internal/Component.ts:init()` and needs to evaluate all reactive statements.
- **Idle**: Waiting for a change in a reactive property
- **Scheduled**: A reactive property changed, waiting for synchronous userland code to complete before applying all changes.
- **Responding**: Running reactive statements based on the reactive properties that changed.
- **BeforeUpdate**: Code that should run before the (DOM) updates.
- **Update**: The components uPdate function is run `$$.p(dirty)` which applies the DOM updates based on the reactive properties that changed.
- **AfterUpdate**: Code that should run after the (DOM) updates.
- **Destroyed**: Component has been destroyed any change to a responsive property is ignored.

**\$\$.update()**: the generated function of a component that runs reactive statements, NOT in the Update phase, but in the Responding phase.  
**\$\$invalidate**: function that is injected into the compiled code which checks if a reactive property has changed.

### 2. Current implementation and caveats

I've gathered these conclusions based on reading the code and performing tests, please correct me where I reached a wrong conclusion or left out vital information.

#### Phases and transitions

```
Init -> BeforeUpdate -> Update -> AfterUpdate
AfterUpdate -> Idle
AfterUpdate -> Scheduled
Idle -> Scheduled -> Responding -> BeforeUpdate -> Update -> AfterUpdate -> Idle
Idle -> Destroyed
Scheduled -> Destroyed
```

In the current design the phases and their transitions are not explicit:

Init is determined by a `ready` boolean, which becomes true after running all reactive statements.

Scheduled is determined by a dirty_components array.

During Responding and beforeUpdate the `$$.dirty` is positive: `$$.dirty[0] !== -1`.

Before the Update phase the mask is reset to `[-1]`

Destroyed is when `$$.fragment === null`

#### Phase: Init

The all reactive statements `$: ...` are run once, `[-1]` matches everything.  
If a statement triggers a change in a variable of a statement that already ran, this change is ignored.

Init ends before the first `beforeUpdate` so in this case a beforeUpdate can also reschedule if it changes a reactive property, this is not the case subsequent `beforeUpdate` calls. [REPL](https://svelte.dev/repl/4458777ee07f46ba9e885c36bd269128?version=3.29.0)

#### Phase: Idle

When `$$invalidate()` is called the reactive property is marked as dirty and the component is added to the dirty_components array, and if this is the first component the scheduler schedules a flush to apply all changes for all dirty components.

If the scheduler is already running is determined by a `update_scheduled` boolean, that's why when an exception occurs the scheduler stops and this boolean is never changed back.

#### Phase: Scheduled

The scheduler waits until the event/subscription handlers are finished (and calls to `$$invalidate()` mark reactive properties as dirty)

#### Phase: Responding

The generated `$$.update()` is called and statements are executed based on the dirty flags.  
Calls to `$$invalidate()` will mark additional properties as dirty, skip the schedule step and skip the adding to dirty_components. The compiler creates a smart ordering of the statements so that when an `$$invalidate()` is called the handlers of that property are below it.  
(But the ordering isn't perfect in all cases, creating subtle bugs)  
After the `$$.update()` ran it continues with the next phase (beforeUpdate)

#### Phase: BeforeUpdate

After the `$$.update()` is called all the beforeUpdate hooks are called (update refers to updating the dom)  
When a `$$invalidate` is called during beforeUpdate, this will update the mask and skip scheduling.

After the hooks ran the dirty mask is reset (after making a local copy)

Changes made in a beforeUpdate are rendered, but ignored by reactive statements.

I haven't used beforeUpdate a lot, an improvement would be to leave the dirty flags created by beforeUpdate, this would allow the next Responding phase to react to the changes. (but they are still ignored in the mean time or when the component is destroyed next)

#### Phase: Update

`$$.p()` is called with the local copy of the mask.  
When a `$$invalidate` is called during `p()` the mask is updated, and the component is added to the dirty queue.

Mutating variables directly in the template `{ count++ }` doesn't result in a call to `$$invalidate`  
(compiler only generates them in the `<script>` tag)

It does cause problems though, [out of sync](https://svelte.dev/repl/3eb88e72c2ed4adfb70271e6dc931d9c?version=3.29.0) and allows for an [infinite loop](https://svelte.dev/repl/8a169d8e47b04587ac3bd0fe5025e355?version=3.29.0)

I haven't encountered these bugs (or at least didn't consider them to be the fault of svelte)  
There are even valid use-cases for triggering `$$invalidate`. Updating state based on new DOM for example.

Compiler might suggest not to use assignments outside script tags (unless inside an event handler), but there might be valid use-cases, like [destructuring?](https://svelte.dev/repl/4984054513944e9689e0e69d3cbd0316?version=3).

### Phase: AfterUpdate

All the afterUpdate hooks are called, when a `$$invalidate` is called during beforeUpdate, this will update the mask and skip scheduling.

When a `$$invalidate` is called during `p()` the mask is updated, and the component is added to the dirty queue.

### 3. Proposed improvements

Explicit phases, with a `$$.phase` property.

```js
const PHASE_INIT = 0;
// ...
const PHASE_DESTROYED = 7;

if ($$.phase === PHASE_IDLE) {
  dirty_components.push(component);
  schedule_update();
}
```

This change would makes the code easier to reason about, and allows the scheduler to:

- Detect which component was throwing the Error, allowing it to recover and mark the component into an error state.
- Allow the scheduler to detect if the component is rescheduled n times in the same loop
- Allow svelte to warn users a change was triggered in a phase it shouldn't

#### Problem 1: Improve correctness

We can't statically analyse everything, so the solution of running the reactive statements with the latest values is running `$$.update()` multiple times, but we have to be smart about how and how many times.

During `$$.update()` calls to `$$invalidate()` should remove the dirty flag from the mask the `$$.update()` is operating on and mark the property as dirty on a separate dirty mask.
We now repeat the `$$.update()` with the new mask until no new `$$invalidate()` calls are made.
Then combine all dirty masks for the next phases.

Reasoning:

- If a handler of the flag already ran it had the previous value.
- If a handler was about to run and we didn't remove the flag, the handler would run twice with the same value.

This could be further optimized by adding checkpoints into the compiled `$$.update` that way the `$$invalide` knows if a flag was not yet processed (which is common due to the smart ordering).

#### Problem 1.1: Infinite loops

In most cases this will work, however if a change to A triggers a change in B and that triggers change in A, the loop will never end.

To combat this there are several solutions. Detecting a loop is hard, take the A - B - A for example, it's impossible to predict if the new value of A would also trigger a change in B.

If the combined mask changed, keep running the `$$.update` function. when the combined mask is the same as before, run the update one last time. If that last update still makes calls to `$$invalidate` remember those flags and set those the dirty when the update completed and warn the user: "postponed change detection for x".

Reasoning:

- Doesn't rely on counters with magic numbers
- Quicky stops executing on possible loops
- The reactive statement will run eventually (when another change is detected and scheduled)
- Not too complicated to implement
- Changes of not working correctly much lower than the current implementation

## Future improvements ideas:

- The compiler knows which a flags are used in the Responding and which are used in the Update phases
  That information can be used to avoid unnecessary calls to update() or p()
  (Do we want to call beforeUpdate and afterUpdate if p() isn't going to do anything?)
- Compiler knows the names corresponding with the flags, exposing those names on the component in dev mode would allow better error messages. "`name` was changed too many times"

## How we teach this

The improvements don't require changes of existing svelte code, the reactivity remains invisible, and for new users comes with less caveats.

If code depends on reactivity not working, that code will need to be updated.

Some improvements like error recovery will need to be added to the docs.

## Drawbacks

The current implementation will be faster but I favor correctness over speed.  
(reactive statements are run 0 or 1x per `$$.update()`)

The proposed system is still not 100% correct.

## Alternatives

### Detect loop by mask patterns

Detecting a looping patterns pattern in the masks:

```
update 1: changed A & B
update 2: changed C & A
update 3: changed D
update 4: changed A & B
update 5: changed C & A
update 6: changed D
```

update 1 to 3 is the same as 4 - 6, this is probably a loop.

Pro: Leads
Cons: Slower, more memory usage, difficult to implement, still not 100% correct

### Diffing the mask

In the reactive phase: A call to `$$invalidate()` should add a diry flag to a separate dirty mask but don't update the mask the `$$.update()` is working on. After the update ran, compare the dirty mask to the mask the `$$.update()` started with and call `$$.update()` again with only the flags that were missing before. This solves the bug of not running a handler at all based on ordering, but doesn't solve correctness, if a handler is called with the latest or an outdated value still depends on the ordering.

Pro's: No looping issues, no duplicate calls to handlers  
Cons: Not predictable

### Limit update to run n times

Run the `$$.update()` until no `$$invalidate` is called or a limit of say 100 is reached.
If the limit is reached either show a warning in the console and continue or silently start the next phase.

Pro: Easy to implement  
Cons: How many should n be? creating a computed property based on another computed property is allowed, but now suddenly has a limit.  
Creates slow downs when a component creates a loop.

### No protection against infinite loop

Just keep rerunning `$$.update()` move the responsibility to svelte users.

Cons: This effectively crashes the browser tab, making it hard to debug.

### Don't allow changing properties during an update

When an `$$invalidate` is called during an `$$.update()` warning in the console and let the user change his code.

This wont work, take this example:

```js
$: message = isFirstTime ? `Hi ${fullname}` : `Welcome back ${firstname}`;
$: fullname = `${firstname} ${lastname}`;
```

When firstname or lastname changed, an update is run and fullname is constructed during that `$$.update()` call which triggers an `$$invalidate` for fullname.  
This is valid svelte code and should be allowed.

## Reactivity system goals

- Reliable / Predictable
  - Reactive statements always work, independent of their ordering in the source.
  - Should be able to reason about without looking at the compiled output.
- Informative
  - When a user creates an infinite loop, report how.
  - Don't overly guide the users, (an each without key is ok)
- Invisible
  - No $scope.$apply()
- Fast / Small
  - No infinite loops
  - Low memory usage
  - Full error detection with details in dev mode, stripped out in production bundle
- Binary Compatible if reasonable
  - Compatible with components compiled with an older svelte compiler, unless there are significant benefits

## Unresolved questions

How to warn users about reactivity bugs?  
Not in 100% of the cases it's a bug, so an opt-in to these warnings is prefered (but don't call it strict mode)  
it's name should indicate that it's ok to leave it off and only toggle it on when debugging strange behavior.

When should triggering reactive statements/assignments be allowed?  
Allowed is maybe the wrong word, the variable is already changed, it just that svelte doesn't react to it.
