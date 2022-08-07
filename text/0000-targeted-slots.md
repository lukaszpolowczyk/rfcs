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

Generally - gives superpowers of combining to... `<svelte:element>`, and improves the performance of slots.

## Motivation

In short - the same as in these RFCs:

- Declarative Actions - https://github.com/sveltejs/rfcs/pull/41
- Implement forward directive - https://github.com/sveltejs/rfcs/pull/60
- Targeted style inheritance - https://github.com/sveltejs/rfcs/pull/66
(I hope the links are enough, because otherwise I would have to copy the Motivation section with this RFC)
- And, in general, all the problems with `:global` and passing styles between parent and child components that everyone knows about.

I wrote the first glimpses of this idea here:

- that Declarative Actions are a bit like svelte:element + slots - https://github.com/sveltejs/rfcs/pull/41#issuecomment-934364393
- the first idea to implement svelte:element + slots - https://github.com/sveltejs/rfcs/pull/41#issuecomment-1126434719
- the idea of grouping attributes in Forward Directive - https://github.com/sveltejs/rfcs/pull/60#issuecomment-1073263370
- discovery that an idea useful in Declarative Actions, can be useful in Forward Directive - https://github.com/sveltejs/rfcs/pull/60#issuecomment-1204651747
- discovery that the same idea can also work in Targeted Style - https://github.com/sveltejs/rfcs/pull/66#issuecomment-1207270679
  (funnily enough, the name is similar, although mine was taken from the `<target>` tag in Declarative Actions)

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
<Child><svelte:element slot="name"/ let:val>{val}</svelte:element></Child>
```

---

Reading the value of `$$slots.name` in `Child`.


#### **Differences from slots**

No possibility to use unnamed slots - for great simplicity and to avoid collision of component and element attribute names.  
(if you read to the end, you'll know, probably no need to explain separately)

---

In `Child` using the `<svelte:element/>` tag, instead of the `<slot>` tag, with the necessary `targeted:name` attribute instead of `name="name"`.

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name/>
```

This is because the `name` attribute can be useful in `<svelte:element/>`, because it is a simple HTML attribute, it is not just for slots.  
You will be able to use the `name` attribute in this sense - https://www.w3schools.com/tags/att_name.asp

The name `targeted` came from the name of the `<target>` tag, from the **Declarative Directive** proposal.

---

Targeted Slots can only work with `svelte:element`, e.g. `<svelte:element slot="name"/>`. Therefore, to emphasize that there is something dynamic going on here.  
(although in "Alternatives" I wonder about that)

---

Using in `Child` multiple slots with the same name, but with a new syntax. This is possible in simple slots, but I show how to do it with the new syntax.

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name/>
<svelte:element targeted:name/>
```

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
<!-- Child.svelte -->
<svelte:element targeted:name={ {val} }/>
```

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name" let:val>{val}</svelte:element></Child>
```

It's less pretty, but it's a compromise worth making.

---

No `<!-- optional fallback -->`(fallback of the whole element). It's important to be able to do a fallback of the content itself.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element targeted:name/></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name/>Content that <span>will not disappear</span>.</svelte:element>
```

If there is no content in `Parent` in `<svelte:element/>`, the content from `Child` is visible. That is, as if the fallback is for the content, not for the whole element.

If one needs, the fallback of the whole element can be done with `$$slots.name` and `{#if}`.

---


Using the special attribute `bind:this={target}` in the slot. This was not present in simple slots at all.

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name bind:this={target}/>
```

It's **very important** because it's essential for solving the problems described in **Declarative Actions**.  

Allows to hide complex logic in `Child`, hidden under a simple element visible in `Parent`.

Declarative Actions is the idea that for `use:action` you can use a Component, with `<target bind:this={target}>` in the component, to be able to use the advantages of SvelteJS syntax, and at the same time use the advantages of actions. 

My proposal also does this, but in a different (more universal and without using the action attribute) way.

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
<SubChild><svelte:element this="div" targeted:subname/></SubChild>
```

Here you are not creating a slot target, but passing a slot from `Parent.svelte` to `SubChild.svelte`.  
All attributes set in `Parent`, will be assigned to `<svelte:element/>` in `SubChild`.

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
Not counting the case of "Styles for the element-child slot", about which in "Unresolved questions".

---

In **Forward Directive** they propose a complex system of attribute exceptions.  
With Targeted Slots, you can freely place attributes in separate named slots.  
This will completely replace this complicated exclusion system.


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

This is an example of simple exclusion, by using a separate name.  
In one place there is both `click` and `mouseover`, and in another place there is only `click`.


---

Ability to mix simple slots, with Targeted Slots.

```svelte
<!-- Parent.svelte -->
<Child><svelte:element slot="name"/></Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name/>
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
<svelte:element targeted:name/>
<slot name="name2"/>
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

---

Using `<target>` or simple `<slot>`, instead of `<svelte:element targeted:name/>`.  
The first introduces an unnecessary new basic tag. The second can be confusing.  

The `<svelte:element>` better reflects a situation where attributes from two elements are combined.

---

Perhaps allowing the use of simple `div` etc., instead of `<svelte:element/>` - Both in `Parent` and in `Child` - this will give a slightly nicer look, but maybe it is technically impossible?  
Then you have to rely on the `targeted:name` `slot`, `slot:subname` attributes themselves - they would be the ones that would cause the dynamic behavior of the slots.

This fits more with "Unresolved questions", but I don't know myself.

---

Leaving the fallback element, and passing the content in this complicated and stupid way:

```svelte
<!-- Child.svelte -->
<svelte:element this="div" ><content:slot>only content fallback</content:slot> whole element fallback</svelte:element>
```

I'm writing this just so you know that I have different thoughts.

## Unresolved questions

The attribute value written in `Parent`, should overwrite the attribute value written in `Child`.  
Maybe it should be possible to decide the order of overwriting?

---

What part of attributes and special attributes can be easily handled with this API. This is known to the maintainers who directly work on SvelteJS.

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

Styles for the element-child slot.

```svelte
<!-- Parent.svelte -->

<div class="blue">
    <Child><svelte:element slot="button" class="button"/></Child>
</div>

<style>
/* Effective? */
.button span {
  font-family: "Comic Sans MS";
}
</style>
```

```svelte
<!-- Child.svelte -->

<svelte:element targeted:button this="button">
  <!-- Will be styled font-family: "Comic Sans MS";? -->
  <span>Foo</span>
</svelte:element>
```

I don't know how the SvelteJS compiler would handle this. Is the fact that `.button` was set in `Parent` enough to make the style for `.button span` work?

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
  <svelte:element slot="name" whatto:callit={ {val} }/>
</Child>
```

```svelte
<!-- Child.svelte -->
<svelte:element targeted:name let:val>content {val}</svelte:element>
```

But it is not necessary, because it is possible to pass data through `Child` parameters.

A i tak nie mam pomysłu `whatto:callit`.

---

Also for the reason that in this proposal there is no fallback for the element, only for the content of the element, you can do it.


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