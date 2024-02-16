- Start Date: 2022-08-07
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Combined Snippets

## Summary

He wants to transfer the concept of the abandoned Targeted Slots(my old proposal), to the Snippets syntax. Because Snippets replaces in Svelte 5, the entire syntax of slots.


## Detailed design

Basic example

```svelte
<!-- Parent.svelte -->
<Child>
  {#snippet name()}<svelte:element bind:snippet={name} data-val="val"/>{/snippet}
</Child>
```

```svelte
<!-- Child.svelte -->
{#snippet name()}<svelte:element bind:snippet={name} this="div" data-val2="val2"/>{/snippet}
{@render name()}
```

It gives the effect of

```svelte
<div data-val="val" data-val2="val2"/>
```

So, combining attributes placed in `Parent` and in `Child`. The same way of combining SvelteJS special attributes.

In case of attribute name collisions, the value of the attribute in `Parent` overwrites the value of the attribute in `Child`.

The difference between a regular Snippet, is the use of the same name in `bind:snippet={name}`, and in `Child`, followed by `{@render name()}`.

* Just use the same name so that the element with the `bind:snippet` attribute becomes a stargeted snippet, the only element of the snippet.
* Just use the same name of the second stargeted snippet so that it is treated as a linked element.
* Just use that snippet name in `@render` to render the merged element.

Other than that, there is no difference in syntax from regular Snippets.

...
