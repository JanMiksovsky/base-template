# A possible solution for web component subclasses

This repository presents a web component that attempts to solve a long-standing puzzle: is it possible to [define HTML custom element subclasses that can fill in base class insertion points](http://blog.quickui.org/2013/06/11/puzzle-define-html-custom-element-subclasses-that-can-fill-in-base-class-insertion-points/)? That article describes an early web component library that evolved into the [Basic Web Components project](https://github.com/basic-web-components/components-dev/wiki), but the same issue remains today. A [web component spec bug](https://www.w3.org/Bugs/Public/show_bug.cgi?id=22344) tried to resolve this question by fixing the platform, but an initial [Blink implementation](https://code.google.com/p/chromium/issues/detail?id=263701) ran into problems and had to be backed out. This problem is unlikely to be resolved at the platform level anytime soon, hence the interest in finding a workaround.

This seemingly arcane point may make the difference between whether a well-factored web component library is possible or impossible.

# Success criteria

From that puzzle description, the criteria for a valid solution to the problem are:

1. An instance of a subclass is a proper instance of its base class. All the normal JavaScript stuff should work: property/method access should go up the prototype chain, and a subclass instance should report that it is an “instanceof” the base class.
2. A subclass can put stuff into an insertion point defined by the base class. That is, the subclass can fill in a slot (or slots) defined by a base class. In turn, the subclass should be able to redefine such an insertion point so that the subclass itself can be subclassed.
3. Unless overridden, all base class behavior should function properly in an instance of the subclass. E.g., if the base class wires up an event handler, then this works as expected for subclass instances too.
4. Base class properties/methods can be overridden by the subclass. A subclass’ property/method implementation should be able to invoke the base class’ implementation by whatever language means are necessary.
5. The base class can be any HTML custom element class; the base class author shouldn’t have to do special work a priori to enable this kind of subclassing. This ensures a component author can always use someone else’s element class as a base class.

As an example, we want to be able to define a button base class and subclass as follows. A plain-button base class contains the template:

    <template>
      <button>
        <content></content>
      </button>
    </template>

And a component called icon-button wants to subclass plain-button, while filling in the base class' `<content>` insertion point:

    <template>
      <shadow>
        <img src="icon.png">
        <content></content>
      </shadow>
    </template>

With this, an instance

    <icon-button>Hello</icon-button>

Should produce this effective DOM result:

    <button>
      <img src="icon.png">
      Hello
    </button>

But since the platform won't allow icon-button to project content into the `<shadow>` element, this example cannot currently be implemented.

# Proposed solution

The solution takes the form of a web component called base-template. An instance of base-template can be used inside a subclass in order to populate insertion points defined in the base class' template.

This component performs some DOM magic to produce the desired results. The base-template component *folds* its own content into the base class' Shadow DOM subtree — and then replaces itself with that subtree.

When base-instance is first attached to the DOM, the debugger will show that an instance of icon-button has two Shadow DOM subtrees: the first is from the subclass, and the second from the base class.

    <icon-button>
      #shadow-root
        <base-template>
          <img src="icon.png">
          <content></content>
        </base-template>
      #shadow-root
        <button>
          <content></content>
        </button>
    </icon-button>

The base-template component now does its DOM folding. The first step is to fold the component's own content into the older (base class') shadow subtree at the location of its `<content>` element:

    <icon-button>
      #shadow-root
        <base-template></base-template>
      #shadow-root
        <button>
          <img src="icon.png">
          <content></content>
        </button>
    </icon-button>

The second step is for the base-template component to replace *itself* with the contents of the older shadow subtree:

    <icon-button>
      #shadow-root
        <button>
          <img src="icon.png">
          <content></content>
        </button>
      #shadow-root
    </icon-button>

This produces the desired result: the plain-button base class has provided the outer `<button>`, and the icon-button subclass has partially filled in this button with an icon.

The component's name, base-template, hints at the fact that it's effectively copying the base class' template into that location.

# Demo

You can view a solution to this icon-button example in this
[demo](http://JanMiksovsky.github.io/base-template).

# Assessment

This base-template solution hasn't been used in a production environment yet, but initial results are promising. It appears to solve all the criteria described above. In the icon-button example:

* An instances of icon-button will successfully report that it is an `instanceof` a plain-button.
* The icon-button component can fill in the `<content>` insertion point defined by plain-button.
* A buttonClick() method defined by plain-button is inherited by icon-button.
* A showMessage() method defined by plain-button is overridden by icon-button.
* A `disabled` attribute defined by plain-button is inherited by icon-button.
* Generally speaking, the author of plain-button didn't have to do special work to make it subclassable or usable with base-template. That said, there are some restrictions (below).

So this solution appears to pass our test. As a bonus, the solution provides some nice additional features:

* Polymer node annotations are automatically retained. The plain-button component internally has a `<button id="button">`, which can be referred to programmatically as `this.$.button`. This reference will work even in instances of icon-button.
* Styling defined by the base class is inherited by the subclass.
* Significantly, the subclass can *override* the base class' styling without resorting to use of the CSS `!important` directive. Overriding styles is something subclasses often need to do. The base-template component solves this problem by ensuring that `<style>` elements defined by the base class are placed *before* `<style>` elements defined by the subclass. This natural document ordering results in the expected ability to override styles.

This solution has some limitations:
* Every subclass instance must perform non-trivial DOM modifications at run-time, which impairs performance. In practice, this solution is probably fine in small degrees, but would noticeably degrade performance on a page creating many instances of a subclass.
* Because the base class' Shadow DOM subtree is destructively modified, base class code assumptions about the DOM hierarchy might be violated in a subclass. It remains to be seen whether such problems ever come up in practice. It's also likely that, with a little care, the base class code could be fixed to avoid these problems.
* As written, this technique cannot be used to subclass standard HTML elements. For this reason, plain-button composes a regular button, rather than extending HTMLButtonElement with `is="plain-button"`.
