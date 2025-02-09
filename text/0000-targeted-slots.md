- Start Date: 2022-08-07
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Targeted Slots

## Summary

This allows you to pass and combine CSS styles added in parent and child component, read `bind:this={target}` inside child component, and pass other attributes.

It's easiest to think of Targeted Slots as regular SvelteJS slots, but adding attributes and special attributes to the slots also from inside the child component.

It combines attributes given to slot in the parent component, with attributes added in the child component.  
The original SvelteJS slots can only do in parent component.

Instead of a fallback for an element that is in simple slots, there will be a fallback for content and some attributes.

Generally - gives superpowers of combining to... `<svelte:element/>`, and improves the performance of slots.

## Motivation

In short - the same as in these RFCs:

- Declarative Actions - https://github.com/sveltejs/rfcs/pull/41
- Implement forward directive - https://github.com/sveltejs/rfcs/pull/60
- Targeted style inheritance - https://github.com/sveltejs/rfcs/pull/66
(I hope the links are enough, because otherwise I would have to copy the Motivation section with this RFC)
- And, in general, all the problems with `:global` and passing styles between parent and child components that everyone knows about.

I wrote the first glimpses of this idea here:

- that Declarative Actions are a bit like `<svelte:element/>` + slots - https://github.com/sveltejs/rfcs/pull/41#issuecomment-934364393
- the first idea to implement `<svelte:element/>` + slots - https://github.com/sveltejs/rfcs/pull/41#issuecomment-1126434719
- the idea of grouping attributes in Forward Directive - https://github.com/sveltejs/rfcs/pull/60#issuecomment-1073263370
- discovery that an idea useful in Declarative Actions, can be useful in Forward Directive - https://github.com/sveltejs/rfcs/pull/60#issuecomment-1204651747
- discovery that the same idea can also work in Targeted Style - https://github.com/sveltejs/rfcs/pull/66#issuecomment-1207270679  
  (funnily enough, the name is similar, although mine was taken from the `<target/>` tag in Declarative Actions)

## Detailed design

Basic example

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name" data-val="val"/></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name this="div" data-val2="val2"/>
```

It gives the effect of

```svelte
<div data-val="val" data-val2="val2"/>
```

So, combining attributes placed in `Parent` and in `Child`. The same way of combining SvelteJS special attributes.

In case of attribute name collisions, the value of the attribute in `Parent` overwrites the value of the attribute in `Child`.

Here you can see a big difference from simple slots, but despite appearances, there are great similarities.  
Below is an explanation of the differences in syntax, etc.

### Implementation

#### **Similarities to slots**

Using the `slot="name"` attribute.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name"/></Child>
```

---

Using one or multiple slots.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name"/></Child>
```

```svelte
<!-- Parent.svelte -->
<Child>
  <svelte:element slot="name"/>
  <svelte:element slot="name2"/>
</Child>
```

---

Using multiple slots with the same name in `Child`.  
(example in "Differences from slots", because the syntax itself differs)

---

Using `let:val`.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name" let:val>{val}</svelte:element></Child>
```

---

Reading the value of `$$slots.name` in `Child`.

---

The tag `<svelte:fragment slot="name"></svelte:fragment>` will not find any benefit in Targeted Slots, but it too can be used in the same way as before.  
(there is a variant of `<svelte:fragment slot="name"/>` that you will use to pass styles for element-child slot, in a special case, but about it in "Differences from slots")


#### **Differences from slots**

In `Child` using the `<svelte:element/>` tag, instead of the `<slot/>` tag, with the necessary `targeted:name` attribute instead of `name="name"`.

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name/>
```

This is because the `name` attribute can be useful in `<svelte:element/>`, because it is a simple HTML attribute, it is not just for slots.  
You will be able to use the `name` attribute in this sense - https://www.w3schools.com/tags/att_name.asp

The name `targeted` came from the name of the `<target/>` tag, from the **Declarative Directive** proposal.

---

Targeted Slots can only work with `svelte:element`, e.g. `<svelte:element slot="name"/>`. Therefore, to emphasize that there is something dynamic going on here.  
(although in "Rejected at this time" I wonder about that)

---

Using in `Child` multiple slots with the same name, but with a new syntax. This is possible in simple slots, but I show how to do it with the new syntax.

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name/>
<svelte:element targeted:name/>
```

(With that case, there are some "Unresolved questions" about bindings}

---

The ability to not add `this="tagName"` for `<svelte:element/>` - because `this="tagName"` can be added in `Parent`, or in `Child`.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name" this="div" data-val="val"/></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name data-val2="val2"/>
```

...or...

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name" data-val="val"/></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name this="div" data-val2="val2"/>
```

When it is not added either in `Parent` or in `Child`, it is like adding `this={null}`.

---

To be able to use in `<svelte:element targeted:name/>` other special attributes and simple HTML attributes, you need to implement `<slot key={value}>` differently - through an object.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name" let:val>{val}</svelte:element></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name={ {val} } this="div"/>
```

It's less pretty, but it's a compromise worth making.

This will make it possible to write something like this.

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name={ {val} } this="div" name="some-name" val="simple" />
```

...where the attributes `name="some-name"` and `val="simple"`, are simple HTML attributes, not things passed to `let:val`, nor the slot name.

---

No `<!-- optional fallback -->`(fallback of the whole element). It's important to be able to do a fallback of the content itself.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element targeted:name/></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name this="div"/>Content that <span>will not disappear</span>.</svelte:element>
```

If there is no content in `Parent` in `<svelte:element/>`, the content from `Child` is visible. That is, as if the fallback is for the content, not for the whole element.

If one needs, the fallback of the whole element can be done with `$$slots.name` and `{#if}`.

---


Using the special attribute `bind:this={target}` in the slot. This was not present in simple slots at all.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element targeted:name this="div"/></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name bind:this={target}/>
```

It's **very important** because it's essential for solving the problems described in **Declarative Actions**.  

Allows to hide complex logic in `Child`, hidden under a simple element visible in `Parent`.

Declarative Actions is the idea that for `use:Component` you can use a `Component`, with `<target bind:this={target}>` in the `Component`, to be able to use the advantages of SvelteJS syntax, and at the same time use the advantages of actions. 

My proposal also does this, but in a different (more universal and without using the action attribute) way.  
But I found that Targeted Slots does not have the equivalent of using multiple `use:action1 use:action2` in a single element. This is one of the Unresolved questions.

---

Passing the slot on to `SubChild`.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name" data-val="val"/></Child>
```

```svelte
<!-- Child.svelte -->
<SubChild><svelte:element slot:subname="name"/></SubChild>
```

```svelte
<!-- SubChild.svelte -->
<SubChild><svelte:element targeted:subname this="div"/></SubChild>
```

Here you are not creating a slot target in `Child`, but passing a slot from `Parent` to `SubChild`.  
All attributes set in `Parent`, will be assigned to `<svelte:element/>` in `SubChild`.

As you can see, neither in `Parent` nor in `Child` you don't need `this="name"` in `<svelte:element/>`, just in one place, this time in `SubChild`.

Simplified syntax when same name in `slot:name="name"` simplified to `slot:name`.

Such a need is written in the **Forward Directive** proposal, but also in **Targeted styles** proposal, for styles (more on that later).

---

Passing CSS classes from `Parent` to `Child`.

```svelte
<!-- Parent.svelte -->

<div class="blue">
    <Child><svelte:element slot="button" class="button"/></Child>
</div>

<style>
/* Effective */
.button {
  color: red;
}

/* Ineffective, because `this="button"` is declared in `Child` and is not part of `Parent` */
.blue button {
  color: blue;
}
/* Ineffective, because `.button"` declared in `Child` and in `Parent` have different hash-e (added by the SvelteJS compiler) */
.blue .button {
  color: blue;
}
</style>
```

```svelte
<!-- Child.svelte -->

<!-- Will be styled color: red; -->
<svelte:element targeted:button this="button">Foo</svelte:element>

<!-- Will remain unstyled -->
<button class="button">Bar</button>
```

The rule is simple - if any class or tagName is declared in `Parent`, then the styles written in `Parent` apply to the slot element.

---

Passing classes from `Parent` to `SubChild`.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="button" class="button"/></Child>
<style>
.button {
  color: red;
}
</style>
```

```svelte
<!-- Child.svelte -->
<SubChild><svelte:element slot:subbutton="button"></SubChild>
```

```svelte
<!-- SubChild.svelte -->
<!-- Will be styled color: red; -->
<svelte:element targeted:subbutton this="button"/>
```

The button in `SubChild` will have red text.

---

The class name of targeted element can be the same in `Parent` and `Child`, because SvelteJS adds its hash (one in `Parent`, one in `Child`), so it won't interfere.

```svelte
<!-- Parent.svelte -->

<div class="blue">
  <Child><svelte:element slot="button" class="button"/></Child>
</div>

<style>
/* Effective */
.button {
  color: red;
}
</style>
```

```svelte
<!-- Child.svelte -->

<!-- Will be styled color: red; background: yellow; -->
<svelte:element targeted:button this="button" class="button">Foo</svelte:element>

<style>
/* Effective */
.button {
  background: yellow;
}
</style>
```

---

The last few descriptions, exhaust the subject of **Targeted style inheritance** proposal.  
Only one more challenging case remains.

Pass styles for the element-child slot.

To get this, you need a specific selector.

Attribute `svelte:selector` for `<svelte:element slot="name"/>`.

It is done like this:

```svelte
<!-- Parent.svelte -->

<div class="blue">
    <Child><svelte:element slot="button" svelte:selector=".button span" class="button"/></Child>
</div>

<style>
/* Effective */
.button span {
  font-family: "Comic Sans MS";
}
</style>
```

```svelte
<!-- Child.svelte -->

<svelte:element targeted:button this="button">
  <!-- Will be styled font-family: "Comic Sans MS"; -->
  <span>Foo</span>
</svelte:element>
```

Then any style from `Parent` that is matched in some `svelte:selector` will be remembered.  
When `Child` is called, it will be searched with the selector from `svelte:selector` and the found elements will get the corresponding hashes.

You can list multiple selectors in `svelte:selector`, after a comma. As for example in standard `document.querySelectorAll("selector1, selector2")`.

Or more nicely:

```svelte
<svelte:element slot="name"
  svelte:selector="selector1"
  svelte:selector="selector2"
/>
```

Using `:scope`, e.g. `<svele:element svelte:selector=":scope > span" slot="name"/>` - when you want a direct child selector targeted slot.  
This is standard syntax - https://developer.mozilla.org/en-US/docs/Web/CSS/:scope

---

The `svelte:selector` attribute can also be useful in `<svelte:fragment slot="name"/>`.

```svelte
<!-- Parent.svelte -->

<div class="blue">
    <Child><svelte:fragment slot="name" svelte:selector=".button span"/></Child>
</div>

<style>
/* Effective */
.button span {
  font-family: "Comic Sans MS";
}
</style>
```

```svelte
<!-- Child.svelte -->

<svelte:fragment targeted:name>
  <button class="button">
    <!-- Will be styled font-family: "Comic Sans MS"; -->
    <span>Foo</span>
  </button>
</svelte:fragment>
```

This will bypass the compulsion to use an element-container in `Parent` when you just want to pass the style to elements in `Child`.

Note that it is not `<svelte:fragment></svelte:fragment>`, but `<svelte:fragment/>`.  
The first works in the standard way, the second allows you to keep content from `Child`, and pass styles for that content.

---

In **Forward Directive** they propose a complex system of attribute exceptions.  
With Targeted Slots, you can freely place attributes in separate named slots.  
This will completely replace this complicated exceptions system.


```svelte
<!-- Parent.svelte -->

<Child>
  <svelte:element slot="name" on:click={click}/>
  <svelte:element slot="name2 on:click={click} on:mouseover={over}"/>
</Child>
```

```svelte
<!-- Child.svelte -->

<svelte:element targeted:name this="div">Foo</svelte:element>

<svelte:element targeted:name2 this="div">Foo</svelte:element>
```

This is an example of simple exception, by using a separate name.  
In one place there is both `click` and `mouseover`, and in another place there is only `click`.


---

Ability to mix simple slots, with Targeted Slots.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name"/></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name this="div"/>
<slot name="name"/>
```

...or...

```svelte
<!-- Parent.svelte -->
<Child>
  <svelte:element slot="name"/>
  <div slot="name2"></div>
</Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name this="div"/>
<slot name="name2"/>
```

...or...

```svelte
<!-- Parent.svelte -->
<Child>
  <svelte:element slot="name"/>
  <Component slot="name2"></Component>
</Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name this="div"/>
<slot name="name2"/>
```

Also you will be able to use the unnamed slot, and at the same time inside the Targeted Slot.

```svelte
<!-- Parent.svelte -->
<Child let:val>
  <svelte:element slot="name">{val}</svelte:element>
</Child>
```

```svelte
<!-- Child.svelte -->
<slot val="val"/>
<svelte:element targeted:name this="div"/>
```

And many other forms of mixing.  
Simply no knowledge of the reason not to allow it. These are sometimes two separate purposes of use, and simplified syntax.


## How we teach this

Therefore, because it is similar to slots, but nevertheless a little different, you need to take advantage of the fact that slots everyone knows.  
But also make a clear distinction between what is different from slots, so that one is not confused with the other.

## Drawbacks

Possible confusion of this with ordinary slots. Therefore, it is rather necessary to give other names.  
On the other hand, if we say that these are slots + extras, it is easier to understand.  
This is the dilemma.


## Alternatives

- Declarative Actions - https://github.com/sveltejs/rfcs/pull/41
- Implement forward directive - https://github.com/sveltejs/rfcs/pull/60
- Targeted style inheritance - https://github.com/sveltejs/rfcs/pull/66

And everything written in the "Alternatives" section of these RFCs.

## Unresolved questions

Someone sees `<svelte:element slot="name"/>` and thinks it's a simple slot. And it is a Targeted Slot.  
Therefore, perhaps the name of the `slot` attribute should be changed into another one. But I don't know into which one?

---

What part of attributes and special attributes can be easily handled with this API. This is known to the maintainers who directly work on SvelteJS.

---
 
The issue of the order in which CSS classes are overridden isn't certain either, as it depends on which component is initialized first, or something like that.... It's also for the SvelteJS engine specialis.  
This can be solved, it seems to me, by double using .hashParent.hashParent from Parent to override .hashChild from Child. I think such a method is used somewhere in Svelte.

---

For the reason that in this proposal there is no fallback for the element, only for the content of the element, you can do it.

```svelte
<!-- Parent.svelte -->
<Child>
  <svelte:element slot="element" />
  <svelte:element slot="subelement" />
</Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:element>content <svelte:element targeted:subelement/></svelte:element>
```

But perhaps it's too complicated?

---

`Component` as Targeted Slot?  
I don't know how this could be handled.

```svelte
<!-- Parent.svelte -->
<Child>
  <svelte:component slot="name"/>
</Child>
```

```svelte
<!-- Child.svelte -->
<svelte:component targeted:name this={Component}/>
```

...?

I don't know if this is good.  
But it could be useful (example needed)

---

What about this? What does `el` refer to, in `Parent`?

```svelte
<!-- Parent.svelte -->
<script>
  let el;
</script>
<Child><svelte:element slot="name" bind:this={el}/></Child>
```

```svelte
<!-- Child.svelte -->
<script>
  let el1;
  let el2;
  let el3;
</script>
<svelte:element targeted:name bind:this={el1}/>
<svelte:element targeted:name bind:this={el1}/>
<svelte:element targeted:name bind:this={el2}/>
```

It seems that in `Parent`, `el` should be an array.

Perhaps a new syntax `bind:these={els]` is needed?

```svelte
<!-- Parent.svelte -->
<script>
  let els;
  let el1 = els[0];
  let el2 = els[1];
  let el3 = els[2];
</script>
<Child><svelte:element slot="name" bind:these={els}/></Child>
```

---

Targeted Slots does not have the equivalent of using multiple `use:action1 use:action2` in a single element.  
This was pointed out by user mimbrown.  

So far I have no idea how to solve this.

### Rejected at this time

The attribute value written in `Parent`, should overwrite the attribute value written in `Child`.  
Maybe it should be possible to decide the order of overwriting?

---

Perhaps allowing the use of simple `<div/>` etc., instead of `<svelte:element/>` - Both in `Parent` (with `slot="name"`) and in `Child`(with `targeted:name`) - this will give a slightly nicer look, but maybe it is technically impossible?  
Then you have to rely on the `targeted:name` `slot`, `slot:subname` attributes themselves - they would be the ones that would cause the dynamic behavior of the slots.

---

Using `<target/>` or simple `<slot/>`, instead of `<svelte:element targeted:name/>`.  
The first introduces an unnecessary new basic tag. The second can be confusing.  
There is an option with `<svelte:target targeted:name/>`. I don't know if it's worth it.

The `<svelte:element/>` better reflects a situation where attributes from two elements are combined.

---

Passing more than one `targeted:name` to one target and to `slot:subname`.

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name targeted:name2/>
```

```svelte
<!-- Child.svelte -->
<SubChild><svelte:element slot:subname="name" slot:subname2="name2" /></SubChild>
```

I don't know if it would be useful for anything, and if it wouldn't cause too much complication.

---

For the reason that in this proposal there is no fallback for the element, only for the content of the element, you can do it.

```svelte
<!-- Parent.svelte -->
<Child>
  <svelte:element slot="name"/>
</Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name>content</svelte:element>
```

This opens up the chance to use the `let:val` syntax in the other direction.

```svelte
<!-- Parent.svelte -->
<Child>
  <svelte:element slot="name" pass:val="val" pass:val2="val2"/>
</Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name let:val let:val2>content {val} {val2}</svelte:element>
```

New special attribute `pass:val="val"`.

But it is not necessary, because it is possible to pass data through `Child` parameters.

---

If we adopt the syntax `pass:val="val"`, it could also be used instead of the object in `targeted:name={ {val} }`.

```svelte
<!-- Parent.svelte -->
<Child>
  <svelte:element slot="name" let:val let:val2/>content {val} {val2}</svelte:element>
</Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name pass:val="val" pass:val2="val2"/>
```

---

Leaving the fallback element, and passing the content in this complicated and stupid way:

```svelte
<!-- Child.svelte -->
<svelte:element this="div" ><content:slot>only content fallback</content:slot> whole element fallback</svelte:element>
```

I'm writing this just so you know that I have different thoughts.

---

In the comments on the Forward Directive proposal, there is a desire to cross-mix attributes (between `Component` and `element`), but that would be strange. Although sometimes, in special cases, it would look more graceful.

This is another oddity that I wrote just for the record.
