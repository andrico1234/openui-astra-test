---
menu: Proposals
name: Pop Up API (Explainer)
path: /components/popup.research.explainer
pathToResearch: /components/popup.research
layout: ../../layouts/Layout.astro
---

- [@mfreed7](https://github.com/mfreed7), [@scottaohara](https://github.com/scottaohara), [@BoCupp-Microsoft](https://github.com/BoCupp-Microsoft), [@domenic](https://github.com/domenic), [@gregwhitworth](https://github.com/gregwhitworth), [@chrishtr](https://github.com/chrishtr), [@dandclark](https://github.com/dandclark), [@una](https://github.com/una), [@smhigley](https://github.com/smhigley), [@aleventhal](https://github.com/aleventhal)
- June 1, 2022

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

## Table of Contents

- [Background](#background)
  - [Goals](#goals)
  - [See Also](#see-also)
- [API Shape](#api-shape)
  - [HTML Content Attribute](#html-content-attribute)
  - [Showing and Hiding a Pop-up](#showing-and-hiding-a-pop-up)
    - [Declarative Triggers](#declarative-triggers)
    - [Javascript Trigger](#javascript-trigger)
    - [Page Load Trigger](#page-load-trigger)
    - [CSS Pseudo Class](#css-pseudo-class)
    - [Shown vs Hidden Pop-ups](#shown-vs-hidden-pop-ups)
  - [IDL Attribute and Feature Detection](#idl-attribute-and-feature-detection)
  - [Events](#events)
  - [Focus Management](#focus-management)
  - [Anchoring](#anchoring)
  - [Backdrop](#backdrop)
- [Behaviors](#behaviors)
  - [Automatic Dismiss Behavior](#automatic-dismiss-behavior)
    - [Light Dismiss](#light-dismiss)
    - [Nested Pop-ups](#nested-pop-ups)
    - [The Pop-up Stack](#the-pop-up-stack)
    - [Nearest Open Ancestral Pop-up](#nearest-open-ancestral-pop-up)
    - [Close signal](#close-signal)
  - [Classes of Top Layer UI](#classes-of-top-layer-ui)
    - [One at a time behavior summary](#one-at-a-time-behavior-summary)
    - [Detailed description of interactions among pop-up types](#detailed-description-of-interactions-among-pop-up-types)
  - [Accessibility / Semantics](#accessibility--semantics)
  - [Disallowed elements](#disallowed-elements)
- [Example Use Cases](#example-use-cases)
  - [Generic Pop-up (Date Picker)](#generic-pop-up-date-picker)
  - [Generic Pop-up (`<selectmenu>` listbox example)](#generic-pop-up-selectmenu-listbox-example)
  - [Hint/Tooltip](#hinttooltip)
  - [Manual](#manual)
- [Additional Considerations](#additional-considerations)
  - [Exceeding the Frame Bounds](#exceeding-the-frame-bounds)
  - [Shadow DOM](#shadow-dom)
  - [Eventual Single-Purpose Elements](#eventual-single-purpose-elements)
- [The Choices Made in this API](#the-choices-made-in-this-api)
  - [Alternatives Considered](#alternatives-considered)
  - [Design decisions](#design-decisions)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Background

A very common UI pattern on the Web, for which there is no native API, is "pop up UI" or "pop-ups". Pop-ups are a general class of UI that have three common behaviors:

1.  Pop-ups always appear **on top of other page content**.
2.  Pop-ups are **ephemeral**. When the user "moves on" to another part of the page (e.g. by clicking elsewhere, or hitting ESC), the pop-up closes.
3.  Pop-ups (of a particular type) are generally **"one at a time"** - opening one pop-up closes others.

This document proposes a set of APIs to make this type of UI easy to build.

## Goals

Here are the goals for this API:

- Allow any element and its (arbitrary) descendants to be rendered on top of **all other content** in the host web application.
- Include **“light dismiss” management functionality**, to remove the element/descendants from the top-layer upon certain actions such as hitting Esc (or any [close signal](https://wicg.github.io/close-watcher/#close-signal)) or clicking outside the element bounds.
- Allow this “top layer” content to be fully styled, including properties which require compositing with other layers of the host web application (e.g. the box-shadow or backdrop-filter CSS properties).
- Allow these top layer elements to reside at semantically-relevant positions in the DOM. I.e. it should not be required to re-parent a top layer element as the last child of the `document.body` simply to escape ancestor containment and transforms.
- Allow this “top layer” content to be sized and positioned to the author's discretion.
- Include an appropriate user input and focus management experience, with flexibility to modify behaviors such as initial focus.
- **Accessible by default**, with the ability to further extend semantics/behaviors as needed for the author's specific use case.
- Avoid developer footguns, such as improper stacking of dialogs and pop-ups, and incorrect accessibility mappings.
- Avoid the need for Javascript for the common cases.

## See Also

See the [original `<popup>` element explainer](https://open-ui.org/components/popup.research.explainer), and also the comments on [Issue 410](https://github.com/openui/open-ui/issues/410) and [Issue 417](https://github.com/openui/open-ui/issues/417). See also [this CSSWG discussion](https://github.com/w3c/csswg-drafts/issues/6965) which has mostly been about a CSS alternative for top layer.

This proposal was discussed on [Issue 455](https://github.com/openui/open-ui/issues/455), which was closed as [resolved](https://github.com/openui/open-ui/issues/455#issuecomment-1050172067).

# API Shape

This section lays out the full details of this proposal. If you'd prefer, you can **[skip to the examples section](#example-use-cases) to see the code**.

## HTML Content Attribute

A new content attribute, **`popup`**, controls both the top layer status and the dismiss behavior. There are several allowed values for this attribute:

- **`popup=auto`** - A top layer element following "Auto" dismiss behaviors (see below).
- **`popup=hint`** - A top layer element following “Hint” dismiss behaviors (see below).
- **`popup=manual`** - A top layer element following “Manual” dismiss behaviors (see below).

So this markup represents pop-up content:

```html
<div popup="auto">I am a pop-up</div>
```

As written above, the `<div>` will be rendered `display:none` by the UA stylesheet, meaning it will not be shown when the page is loaded. To show the pop-up, one of several methods can be used: [declarative triggering](#declarative-triggers), [Javascript triggering](#javascript-trigger), or [page load triggering](#page-load-trigger).

Additionally, the `popup` attribute can be used without a value (or with an empty string `""` value), and in that case it will behave identically to `popup=auto`:

```html
<div popup="auto">I am a pop-up</div>
<div popup>I am also an "auto" pop-up</div>
<div popup="">So am I</div>
```

For convenience and brevity, the remainder of this explainer will use this boolean-like syntax in most cases, e.g. `<div popup>`.

## Showing and Hiding a Pop-up

There are several ways to "show" a pop-up, and they are discussed in this section. When any of these methods are used to show a pop-up, it will be made visible and moved (by the UA) to the [top layer](https://fullscreen.spec.whatwg.org/#top-layer). The top layer is a layer that paints on top of all other page content, with the exception of other elements currently in the top layer. This allows, for example, a "stack" of pop-ups to exist.

### Declarative Triggers

A common design pattern is to have an activating element, such as a `<button>`, which makes a pop-up visible. To facilitate this pattern, and avoid the need for Javascript in this common case, three content attribute (`togglepopup`, `showpopup`, and `hidepopup`) allow the developer to declaratively toggle, show, or hide a pop-up. To do so, the attribute's value should be set to the idref of another element:

```html
<button togglepopup="foo">Toggle the pop-up</button>
<div id="foo" popup>Pop-up content</div>
```

When the button in this example is activated, the UA will call `.showPopUp()` on the `<div id=mypopup>` element if it is currently hidden, or `hidePopUp()` if it is showing. In this way, no Javascript will be necessary for this use case.

If the desire is to have a button that only shows or only hides a pop-up, the following markup can be used:

```html
<button togglepopup="foo">Toggle the pop-up</button>
<button showpopup="foo">This button only shows the pop-up</button>
<button hidepopup="foo">This button only hides the pop-up</button>
<div id="foo" popup>Pop-up content</div>
```

Note that all three attributes can be used together like this, pointing to the same element. However, using more than one triggering attribute on **a single button** is not recommended.

When the `togglepopup`, `showpopup`, or `hidepopup` attributes are applied to an activating element, the UA may automatically map this attribute appropriate `aria-*` attributes, such as `aria-haspopup`, `aria-describedby` and/or `aria-expanded`, in order to ensure accessibility. There will need to be further discussion with the ARIA working group to determine the exact ARIA semantics, if any, are necessary.

### Javascript Trigger

To show and hide the pop-up via Javascript, there are two methods on HTMLElement:

```javascript
const popUp = document.querySelector("[popup]");
popUp.showPopUp(); // Show the pop-up
popUp.hidePopUp(); // Hide a visible pop-up
```

Calling `showPopUp()` on an element that has a valid value for the `popup` attribute will cause the UA to remove the `display:none` rule from the element and move it to the top layer. Calling `hidePopUp()` on a showing pop-up will remove it from the top layer, and re-apply `display:none`.

There are several conditions that will cause `showPopUp()` and/or `hidePopUp()` to throw an exception:

1. Calling `showPopUp()` or `hidePopUp()` on an element that does not contain a [valid value](#html-content-attribute) of the `popup` attribute. This will throw a `NotSupportedError` `DOMException`.
2. Calling `showPopUp()` on a valid pop-up that is already in the showing state. This will throw an `InvalidStateError` `DOMException`.
3. Calling `showPopUp()` on a valid pop-up that is not connected to a document. This will throw an `InvalidStateError` `DOMException`.
4. Calling `hidePopUp()` on a valid pop-up that is not currently showing. This will throw an `InvalidStateError` `DOMException`.

### Page Load Trigger

As mentioned above, a `<div popup>` will be hidden by default. If it is desired that the pop-up should be shown automatically upon page load, the `defaultopen` attribute can be applied:

```html
<div popup defaultopen></div>
```

In this case, the UA will immediately call `showPopUp()` on the element, as it is parsed. If multiple such elements exist on the page, only the first such element (in DOM order) on the page will be shown.

Note that `hint` pop-ups cannot use the `defaultopen` attribute:

```html
<div popup="hint" defaultopen>I will not be shown on page load</div>
```

Note also that more than one `manual` pop-up can use `defaultopen` and all such pop-ups will be shown on load, not just the first one:

```html
<div popup="manual" defaultopen>Shown on page load</div>
<div popup="manual" defaultopen>Also shown on page load</div>
```

### CSS Pseudo Class

When a pop-up (or any element) is in the top layer, it will match the `:top-layer` pseudo class:

```javascript
const popUp = document.createElement("div");
popUp.popUp = "auto";
popUp.matches(":top-layer") === false;
popUp.showPopUp();
popUp.matches(":top-layer") === true;
```

### Shown vs Hidden Pop-ups

The styling for a pop-up is provided by **roughly** the following UA stylesheet rules:

```css
[popup]:not(:top-layer) {
  display: none;
}

[popup] {
  position: fixed;
}
```

The above rules mean that a pop-up, when not "shown", has `display:none` applied, and that style is removed when one of the methods above is used to show the pop-up. Note that the `display:none` UA stylesheet rule is **not** `!important`. In other words, developer style rules can be used to override this UA style to make a not-showing pop-up visible in the page. In this case, the pop-up will **not** be displayed in the top layer, but rather at it's ordinary `z-index` position within the document. This can be used, for example, to animate the show/hide behavior of the pop-up, or make pop-up content "return to the page" instead of becoming hidden.

## IDL Attribute and Feature Detection

The `popup` content attribute will be [reflected](https://html.spec.whatwg.org/#reflect) as a nullable IDL attribute:

```
[Exposed=Window]
partial interface Element {
  attribute DOMString? popUp;
```

This not only allows developer ease-of-use from Javascript, but also allows for a feature detection mechanism:

```javascript
function supportsPopUp() {
  return Element.prototype.hasOwnProperty("popUp");
}
```

Further, only [valid values](#html-content-attribute) of the content attribute will be reflected to the IDL property, with invalid values being reflected as the value `null`. For example:

```javascript
const div = document.createElement("div");
div.setAttribute("popup", "hint");
div.popUp === "hint"; // true
div.setAttribute("popup", "invalid!");
div.popUp === null; // true
```

This allows feature detection of the values, for forward compatibility:

```javascript
function supportsPopUpType(type) {
  if !Element.prototype.hasOwnProperty("popUp")
    return false; // Pop-up API not supported
  // If the assignment fails, it will return null:
  return !!(document.createElement('div').popUp = type);
}
supportsPopUpType('manual') === true;
supportsPopUpType('invalid!') === false;
```

## Events

Events are fired (asynchronously) when a pop-up is shown (`show` event) and hidden (`hide` event). These events can be used, for example, to populate content for the pop-up just in time before it is shown, or update server data when it closes. The events are:

```javascript
const popUp = Object.assign(document.createElement("div"), { popUp: "auto" });
popUp.addEventListener("show", () => console.log("Pop-up is being shown!"));
popUp.addEventListener("hide", () => console.log("Pop-up is being hidden!"));
```

Neither of these events are cancellable, and both are fired asynchronously.

## Focus Management

Elements that move into the top layer may require focus to be moved to that element, or a descendant element. However, not all elements in the top layer will require focus. For example, a modal `<dialog>` will have focus set to its first interactive element, if not the dialog element itself, because a modal dialog is something that requires immediate attention. On the other hand, a `<div popup=hint>` (which will more often than not represent a "tooltip") does not receive focus at all (nor is it expected to contain focusable elements). Similarly, a `<div popup=manual>`, which may represent a dynamic notification message (commonly referred to as a toast), or potentially a persistent chat widget, should not immediately receive focus (even if it contains focusable elements). This is because such pop-ups are meant for out-of-band communication of state, and are not meant to interrupt a user's current action. Additionally, if the top layer element **should** receive immediate focus, there is a question about **which** part of the element gets that initial focus. For example, the element itself could receive focus, or one of its focusable descendants could receive focus.

To provide control over these behaviors, the `autofocus` attribute can be used on or within pop-ups. When present on a pop-up or one of its descendants, it will result in focus being moved to the pop-up or the specified element when the pop-up is rendered. Note that `autofocus` is [already a global attribute](https://html.spec.whatwg.org/multipage/interaction.html#the-autofocus-attribute), but the existing behavior applies to element focus on **page load**. This proposal extends that definition to be used within pop-ups, and the focus behavior happens **when they are shown**. Note that adding `autofocus` to a pop-up descendant does **not** cause the pop-up to be shown on page load, and therefore it does not cause focus to be moved into the pop-up **on page load**, unless the `defaultopen` attribute is also used.

The `autofocus` attribute allows control over the focus behavior when the pop-up is shown. When the pop-up is hidden, often the most user friendly thing to do is to return focus to the previously-focused element. The `<dialog>` element currently behaves this way. However, for pop-ups, there are some nuances. For example, if the pop-up is being hidden via light dismiss, because the user clicked on another element outside the pop-up, the focus should **not** be returned to another element, it should go to the clicked element (if focusable, or `<body>` if not). There are a number of other such considerations. The behavior on hiding the pop-up is:

- A pop-up element has a **previously focused element**, initially `null`, which is set equal to `document.activeElement` when the pop-up is shown, if a) the pop-up is a `auto` or `hint` pop-up, and b) if the [pop-up stack](#the-pop-up-stack) is currently empty. The **previously focused element** is set back to `null` when a pop-up is hidden.

- When a pop-up is hidden, focus is set back to the **previously focused element**, if it is non-`null`, in the following cases:

  1. Light dismiss via [close signal](https://wicg.github.io/close-watcher/#close-signal) (e.g. Escape key pressed).
  2. Hide pop-up from Javascript via `hidePopUp()`.
  3. Hide pop-up via a **pop-up-contained**\* triggering element with `hidepopup=pop_up_id` or `togglepopup=pop_up_id`. The triggering element must be pop-up-contained, otherwise the keyboard or mouse activation of the triggering element should have already moved focus to that element.
  4. Hide pop-up because its `popup` type changes (e.g. via `myPopUp.popUp='something else';`).

- Any other actions which hide the pop-up will **not** cause the focus to be changed when the pop-up hides. In these cases, the "normal" behavior happens for each action. Some examples include:
  1. Click outside the pop-up (focus the clicked thing).
  2. Click on a **non-pop-up-contained**\* triggering element to close pop-up (focus the triggering element). This is a special case of the point just above.
  3. Tab-navigate (focus the next tabbable element in the document's focus order).
  4. Hide pop-up because it was removed from the document (event handlers not allowed while removing elements).
  5. Hide pop-up when a modal dialog or fullscreen element is shown (follow dialog/fullscreen focusing steps).
  6. Hide pop-up via `showPopUp()` on another pop-up that hides this one (follow new pop-up focusing steps).

The intention of the above set of behaviors is to return focus to the previously focused element when that makes sense, and not do so when the user's intention is to move focus elsewhere or when it would be confusing.

\* In the above, "a pop-up contained triggering element" means the triggering element is contained within **the pop-up being hidden**, not any arbitrary pop-up.

## Anchoring

A new attribute, `anchor`, can be used on a pop-up element to refer to the pop-up's "anchor". The value of the attribute is a string idref corresponding to the `id` of another element (the "anchor element"). This anchoring relationship is used for two things:

1. Establish the provided anchor element as an [“ancestor”](#nearest-open-ancestral-pop-up) of this pop-up, for light-dismiss behaviors. In other words, the `anchor` attribute can be used to form nested pop-ups.
2. The referenced anchor element could be used by the [Anchor Positioning feature](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/CSSAnchoredPositioning/explainer.md).

## Backdrop

Akin to modal `<dialog>` and fullscreen elements, pop-ups allow access to a `::backdrop` pseudo element, which is a full-screen element placed directly behind the pop-up in the top layer. This allows the developer to do things like blur out the background when a pop-up is showing:

```html
<div popup>I'm a pop-up</div>
<style>
  [popup]::backdrop {
    backdrop-filter: blur(3px);
  }
</style>
```

Note that in contrast to the `::backdrop` pseudo element for modal dialogs and fullscreen elements, the `::backdrop` for a pop-up is styled by a UA stylesheet rule `pointer-events: none !important`, which means it cannot trap clicks outside the pop-up. This ensures the "click outside" light dismiss behavior continues to function.

# Behaviors

## Automatic Dismiss Behavior

For elements that are displayed on the top layer via this API, there are a number of things that can cause the element to be removed from the top-layer, besides the ones [described above](#showing-and-hiding-a-pop-up). These fall into three main categories:

- **One at a Time.** Another element being added to the top-layer causes existing top layer elements to be removed. This is typically used for “one at a time” type elements: when one pop-up is shown, other pop-ups should be hidden, so that only one is on-screen at a time. This is also used when “more important” top layer elements get added. For example, fullscreen elements should close all open pop-ups.
- **Light Dismiss.** User action such as clicking outside the element, hitting Escape, or causing keyboard focus to leave the element should all cause a displayed pop-up to be hidden. This is typically called “light dismiss”, and is discussed in more detail in [this section](#light-dismiss).
- **Other Reasons.** Because the top layer is a UA-managed resource, it may have other reasons (for example a user preference) to forcibly remove elements from the top layer.

In all such cases, the UA is allowed to forcibly remove an element from the top layer and re-apply the `display:none` pop-up UA rule. The rules the UA uses to manage these interactions depends on the element types, and this is described in more detail in [this section](#classes-of-top-layer-ui).

### Light Dismiss

The term "light dismiss" for a pop-up is used to describe the user "moving on" to another part of the page, or generally being done interacting with the pop-up. When this happens, the pop-up should be hidden. Several actions trigger light dismiss of a pop-up:

1. Clicking or tapping outside the bounds of the pop-up. Any `mousedown` event will trigger **all open pop-ups** to be hidden, starting from the top of [the pop-up stack](#the-pop-up-stack), and ending with the [nearest open ancestral pop-up](#nearest-open-ancestral-pop-up) of the `mousedown` event's `target` `Node`. This means clicking on a pop-up or its trigger or anchor elements will not hide that pop-up.
2. Hitting the Escape key, or generally any ["close signal"](#close-signal). This will cause the topmost pop-up on [the pop-up stack](#the-pop-up-stack) to be hidden.
3. Using keyboard or other navigation sources to move the focus off of the pop-up. When the focus changes, all pop-ups up to the ["nearest open ancestral pop-up"](#nearest-open-ancestral-pop-up) for the newly-focused element will be hidden. Note that this includes subframes and the user changing windows (a window `blur`) - both will cause all open pop-ups to be closed.

### Nested Pop-ups

For at least `popup=auto`, it is possible to have "nested" pop-ups. I.e. two pop-ups that are allowed to both be open at the same time, due to their relationship with each other. A simple example where this would be desired is a pop-up menu that contains sub-menus: it is commonly expected to support this pattern, and keep the main menu showing while the sub-menu is shown.

Pop-up nesting is not possible/applicable to the other pop-up types, such as `popup=hint` and `popup=manual`.

### The Pop-up Stack

The `Document` contains a "stack of open pop-ups", which is initially empty. When a `popup=auto` element is shown, that pop-up is pushed onto the top of the stack, and when a `popup=auto` element is hidden, it is popped from the top of the stack.

### Nearest Open Ancestral Pop-up

The "nearest open ancestral pop-up" `P` to a given `Node` N is defined in this way:

> `P` is the topmost pop-up in [the pop-up stack](#the-pop-up-stack) for which any one of the following is true:
>
> 1. `P` is a flat-tree DOM ancestor of `N`.
> 2. `P` has an `anchor` attribute whose value is equal to `N`'s `id` or any of `N`'s flat-tree descendent's `id`s.\*
> 3. `N` has an ancestor [triggering element](#declarative-triggers) whose target is `P`.
>    If none of the pop-ups in [the pop-up stack](#the-pop-up-stack) match the above conditions, then `P` is `null`.

The above description needs to be more crisply defined. The [implementation from Chromium](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/dom/element.cc;l=2577;drc=d164a7600a5f094d73a135c1f66f76139b525b48) should be a good starting point to describe the algorithm.

### Close signal

The "close signal" [proposal](https://wicg.github.io/close-watcher/#close-signal) attempts to unify the concept of "closing" something. Most typically, the Escape key is the standard close signal, but there are others, including the Android back button, Accessibility Tools dismiss gestures, and the Playstation square button. Any of these close signals is a light dismiss trigger for the topmost pop-up.

## Classes of Top Layer UI

As described in this section, the three pop-up types (`auto`, `hint`, and `manual`) each have slightly different interactions with each other. For example, `auto`s hide other `hint`s, but the reverse is not true. Additionally, there are other (non-pop-up) elements that participate in the top layer. This section describes the general interactions between the various top layer element types, including the various flavors of pop-up:

- Pop-up (**`popup=auto`**)
  - When opened, force-closes other pop-ups and hints, except for [ancestor pop-ups](#nearest-open-ancestral-pop-up).
  - It would generally be expected that a pop-up of this type would either receive focus, or a descendant element would receive focus when invoked.
  - Dismisses on [close signal](https://wicg.github.io/close-watcher/#close-signal), click outside, or blur.
- Hint/Tooltip (**`popup=hint`**)
  - When opened, force-closes only other hints, but leaves all other pop-up types open.
  - Dismisses on [close signal](https://wicg.github.io/close-watcher/#close-signal), click outside, when no longer hovered (after a timeout), or when the anchor element loses focus.
- Manual (**`popup=manual`**)
  - Does not force-close any other element type.
  - Does not light-dismiss - closes via timer or explicit close action.
- Dialog (**`<dialog>.showModal()`**)
  - When opened, force-closes auto and hint.
  - Dismisses on [close signal](https://wicg.github.io/close-watcher/#close-signal)
- Fullscreen (**`<div>.requestFullscreen()`**)
  - When opened, force-closes auto, hint, and (with spec changes) dialog
  - Dismisses on [close signal](https://wicg.github.io/close-watcher/#close-signal)

### One at a time behavior summary

This table summarizes the interactions between a first top layer element (rows) and a second top layer element (columns), as the second element is shown:

<table>
  <tr><td></td><td></td><td colspan="5">Second element</td></tr>
  <tr><td></td><td></td><td>Fullscreen</td><td>Modal Dialog</td><td>Pop-up</td><td>Hint</td><td>Manual</td></tr>
  <tr><td rowspan="5" >First Element</td><td>Fullscreen</td><td>Hide</td><td>Leave</td><td>Leave</td><td>Leave</td><td>Leave</td></tr>
  <tr><td>Modal Dialog</td><td>Hide*</td><td>Leave</td><td>Leave</td><td>Leave</td><td>Leave</td></tr>
  <tr><td>Pop-up</td><td>Hide</td><td>Hide</td><td>Hide</td><td>Leave</td><td>Leave</td></tr>
  <tr><td>Hint</td><td>Hide</td><td>Hide</td><td>Hide</td><td>Hide</td><td>Leave</td></tr>
  <tr><td>Manual</td><td>Hide</td><td>Hide</td><td>Leave</td><td>Leave</td><td>Leave</td></tr>
</table>

\*Not current behavior

In the table, "hide" means that when the second element is shown (enters the top layer), the first element is removed from the top layer. In contrast, "leave" means both elements will remain in the top layer together.

### Detailed description of interactions among pop-up types

This section details the interactions between the three pop-up types:

1. If a `popup=hint` is shown, it should hide **any** other open `popup=hint`s, including ancestral `popup=hint`s. (**"You can't nest `popup=hint`s".**)
2. If a `popup=auto` is shown, it should hide **any** open `popup=hint`s, including if the `popup=hint` is an ancestral pop-up of the `popup=auto`. (**"You can't nest a pop-up inside a `popup=hint`".**)
3. If you: **a)** show a `popup=auto` (call it D), then **b)** show an **ancestral** `popup=hint` of D (call it T) , then **c)** hide D, the `popup=hint` T should be hidden. (**"A `popup=hint` can be nested inside a pop-up."**)
4. If you: **a)** show a `popup=auto` (call it D), then **b)** show an **non-ancestral** `popup=hint` (call it T) , then **c)** hide D, the `popup=hint` T should be left showing. (**"Non-nested `popup=hint`s can stay open when unrelated pop-ups are hidden."**)
5. The `defaultopen` attribute should have no effect on `popup=hint`s. I.e. this attribute cannot be used to cause a `popup=hint` to be shown upon page load.
6. The `defaultopen` attribute can be used on as many `popup=manual`s as desired, and all of them will be shown upon page load.
7. Only the first `popup=auto` (in DOM order) containing the `defaultopen` attribute will be shown upon page load.

## Accessibility / Semantics

Since the `popup` content attribute can be applied to any element, and this only impacts the element's presentation (top layer vs not top layer), this addition does not have any direct semantic or accessibility impact. The element with the `popup` attribute will keep its existing semantics and AOM representation. For example, `<article popup>...</article>` will continue to be exposed as an implicit `role=article`, but will be able to be displayed on top of other content. Similarly, ARIA can be used to modify accessibility mappings in [the normal way](https://w3c.github.io/html-aria/), for example `<div popup role=note>...</div>`.

As mentioned in the [Declarative Triggers](#declarative-triggers) section, accessibility mappings will be automatically configured to associate the pop-up with its trigger element, as needed.

## Disallowed elements

While the pop-up API can be used on most elements, there are some limitations. For example, calling `showPopUp()` on a modal (via `.showModal()`) `<dialog>` element will result in an exception being thrown, as will calling it on an [active fullscreen element](https://developer.mozilla.org/en-US/docs/Web/API/Element/requestFullscreen). All other element types are valid.

# Example Use Cases

This section contains several HTML examples, showing how various UI elements might be constructed using this API.

**Note:** these examples are for demonstrative purposes of how to use the `togglepopup` and `popup` attributes. They may not represent all necessary HTML, ARIA or JavaScript features needed to fully create such components.

## Generic Pop-up (Date Picker)

```html
<button togglepopup="datepicker">Pick a date</button>
<my-date-picker role="dialog" id="datepicker" popup>
  ...date picker implementation...
</my-date-picker>

<!-- No script - the togglepopup attribute takes care of activation -->
```

## Generic Pop-up (`<selectmenu>` listbox example)

```html
<selectmenu>
  <template shadowroot="closed">
    <button togglepopup="listbox">Icon</button>
    <div role="listbox" id="listbox" popup>
      <slot></slot>
    </div>
  </template>
  <option>Option 1</option>
  <option>Option 2</option>
</selectmenu>

<!-- No script - the togglepopup attribute takes care of activation -->
```

## Hint/Tooltip

```html
<div id=hint-trigger aria-describedby=hint>
  Hover me
</div>
<my-hint id=hint role=tooltip popup=hint anchor=hint-trigger>
  Hint text
</my-tooltip>

<script>
  const trigger = document.getElementById('hint-trigger');
  const hint = document.querySelector('my-hint');
  trigger.addEventListener('mouseover',() => {
    // This behavior could potentially be built into a new activation
    // content attribute, like <div trigger-on-hover=my-hint>.
    hint.showPopUp();
  });
</script>
```

## Manual

```html
<div role="alert">
  <my-manual-container popup="manual"></my-manual-container>
</div>

<script>
  window.addEventListener("message", (e) => {
    const container = document.querySelector("my-manual-container");
    container.appendChild(document.createTextNode("Msg: " + e.data));
    container.showPopUp();
  });
</script>
```

# Additional Considerations

## Exceeding the Frame Bounds

Allowing a pop-up/top-layer element to exceed the bounds of its containing frame poses a serious security risk: such an element could spoof browser UI or containing-page content. While the [original `<popup>` proposal](https://open-ui.org/components/popup.research.explainer-v1) did not discuss this issue, the [`<selectmenu>` proposal](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ControlUICustomization/explainer.md#security) does have a specific section at least mentioning this issue. Some top-layer APIs (e.g. the fullscreen API) make it possible for an element to exceed the frame bounds in some cases, great care must be taken in these cases to ensure user safety. Given the complete flexibility offered by the pop-up API (any element, arbitrary content, etc.), there would be no way to ensure the safety of this feature if it were allowed to exceed frame bounds.

For completeness, several use counters were added to Chromium to measure how often this type of behavior (content exceeding the frame bounds) might be needed. These are approximations, as they merely measure the total number of times one of the built-in “pop-up” windows, which can exceed frame bounds because of their carefully-controlled content, is shown. The pop-ups included in this count include the `<select>` pop-up, the `<input type=color>` color picker, and the `<input type=date/etc>` date/time picker. Data can be found here:

- Total pop-ups shown: [0.7% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/3298)
- Pop-ups appeared outside frame bounds: [0.08% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/3299)

So about 11% of all pop-ups currently exceed their owner frame bounds. That should be considered a rough upper bound, as it is possible that some of those pop-ups **could** have fit within their frame if an attempt was made to do so, but they just happened to exceed the bounds anyway.

In any case, it is important to note that this API cannot be used to render content outside the containing frame.

## Shadow DOM

Note that using the API described in this explainer, it is possible for elements contained within a shadow root to be pop-ups. For example, it is possible to construct a custom element that wraps a pop-up type UI element, such as a `<my-tooltip>`, with this DOM structure:

```html
<my-tooltip>
  <template shadowroot="closed">
    <div popup="hint">This is a tooltip: <slot></slot></div>
  </template>
  Tooltip text here!
</my-tooltip>
```

In this case, the (closed) shadow root contains a `<div>` that has `popup=hint` and that element will be shown on the top layer when the custom element calls `div.showPopUp()`.

This is "normal", and the only point of this section is to point out that even shadow dom children can be promoted to the top layer, in the same way that a shadow root can contain a `<dialog>` that can be `showModal()`'d, or a `<div>` that can be `requestFullscreen()`'d.

## Eventual Single-Purpose Elements

There might come a time, sooner or later, where a new pop-up-type HTML element is desired which combines strong semantics and purpose-built behaviors. For example, a `<tooltip>` or `<listbox>` element. Those elements could be relatively easily built via the APIs proposed in this document. For example, a `<tooltip>` element could be defined to have `role=tooltip` and `popup=hint`, and therefore re-use this pop-up API for always-on-top rendering, one-at-a-time management, and light dismiss. In other words, these new elements could be _explained_ in terms of the lower-level primitives being proposed for this API.

# The Choices Made in this API

Many decisions and choices were made in the design of this API, and those decisions were made via numerous discussions (live and on issues) in [OpenUI](https://open-ui.org/), a WHATWG [Community Group](https://open-ui.org/working-mode).

## Alternatives Considered

To achieve the [goals](#goals) of this project, a number of approaches could have been used:

- An HTML content attribute (this proposal).
- A dedicated `<popup>` element.
- A CSS property.
- A Javascript API.

Each of these options is significantly different from the others. To properly evaluate them, each option was fleshed out in some detail. Please see this document for the details of that effort:

- [Other Alternatives Considered](/components/popup.proposal.alternatives)

That document discusses the pros and cons for each alternative. After exploring these options, the [HTML content attribute](#html-content-attribute) approach [was resolved by OpenUI](https://github.com/openui/open-ui/issues/455#issuecomment-1050172067) to be the best overall.

## Design decisions

Many small (and large!) behavior questions were answered via discussions at OpenUI. This section contains links to some of those:

- [Overall discussion of the content attribute based approach](https://github.com/openui/open-ui/issues/455)
- [Why `show`/`hide` and not `open`/`close`, and why not parallel to `<dialog>`](https://github.com/openui/open-ui/issues/322)
- [Rules for multiple triggering elements on a single button.](https://github.com/openui/open-ui/issues/523#issuecomment-1106686358)
- [Should there be a `::backdrop` pseudo element](https://github.com/openui/open-ui/issues/519)
- [Should `showPopUp()` and `hidePopUp()` throw](https://github.com/openui/open-ui/issues/511)
- [Should `blur()` be a light dismiss trigger](https://github.com/openui/open-ui/issues/415)
- [IDL reflects only valid values](https://github.com/openui/open-ui/issues/491#issuecomment-1118927375)
- [Why `togglepopup` (bikeshed)](https://github.com/openui/open-ui/issues/508)
- [Why `defaultopen` (bikeshed)](https://github.com/openui/open-ui/issues/500)
- [Why `display:none` for hidden pop-ups](https://github.com/openui/open-ui/issues/492)
- [Why Close Signals and not just ESC](https://github.com/openui/open-ui/issues/320)
- [Naming of the `:top-layer` pseudo class](https://github.com/openui/open-ui/issues/470)
- [Support for "boolean-like" behavior for `popup` attribute](https://github.com/openui/open-ui/issues/533)
- [Returning focus to previously-focused element](https://github.com/openui/open-ui/issues/327)
- [The `show` and `hide` events should not be cancelable](https://github.com/openui/open-ui/issues/321)
- [The `popup` attribute confers behavior and not semantics](https://github.com/openui/open-ui/issues/495#issuecomment-1164827851)
- [Mouse down vs mouse up for light dismiss](https://github.com/openui/open-ui/issues/529)
- [Imperative API for content attributes](https://github.com/openui/open-ui/issues/382)
- [`.popup` vs `.popUp`](https://github.com/openui/open-ui/issues/546#issuecomment-1158190204)
- [Interactions between auto, hint, and manual](https://github.com/openui/open-ui/issues/525)
