- Start Date: 2019-11-01
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This RFC introduces a new API, proposed here as a Hook, that enables a component
to remain mounted for a period of time before it is unmounted.

The purpose of this feature is to simplify the process of adding exit animations
to components.

# Basic example

In the following example, `MyComponent` will be kept in the React component tree, and the DOM,
for 5 additional seconds after it would normally be unmounted.

```jsx
import { useDeferredUnmount } from 'react';

export default function MyComponent() {
  useDeferredUnmount(() => {
    return new Promise(resolve => {
      setTimeout(resolve, 5000);
    });
  });

  return (
    <div>Hello</div>
  );
}
```

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.

# Detailed design

### The Unmounting Phase

If a component uses the hook `useDeferredUnmount`, and a Promise is returned from the function
passed as the first argument to the hook, then React will behave differently when that component
would normally be unmounted.

Instead of immediately unmounting the component, React will keep the component mounted in the component tree
and in the DOM. The component will remain mounted until the promise resolves.

This RFC will refer to the period of time between when the component begins unmounting and when it is actually
unmounted as the "unmounting phase." To say that a component is "unmounting" is equivalent to saying that it is
in the unmounting phase.

```jsx
import { useDeferredUnmount } from 'react';

export default function MyComponent() {
  useDeferredUnmount(() => {
    // The component will enter the unmounting phase whenever a Promise is returned from this new Hook.
    return new Promise(resolve => {
      setTimeout(() => {
        // The component will now exit the unmounting phase and be removed from
        // the React's component tree.
        resolve();
      }, 5000);
    });
  });

  return (
    <div>Hello</div>
  );
}
```

A component's lifecycle can be visualized as follows:

```
mounted ---> updated ---> ... ---> updated ---> unmounting phase begins ---> unmounting phase ends
```

### "Frozen" Components

During the unmounting phase, React will no longer make changes to the DOM tree described by a component. This is called being
"frozen."

Unmounting components are frozen as a consequence of the following three features of this RFC:

1. Props will no longer be updated during the unmounting phase. This is a consequence of the parent no longer
  returning the component's element, so there is no API available to update the props.
 
2. At the start of the unmounting phase, the component will be unsubscribed from any Context values that it is
  subscribed to. Accordingly, Context updates during the unmounting phase will be ignored by the component.

3. Attempts to update the component's state will be ignored by React. If a developer attempts to update a component's state
  during the unmounting phase, a warning will be logged to the console when in development mode.

Additionally, the entire component tree within a frozen component is frozen as well.

Because of freezing, it is up to the developer to imperatively update the component's DOM to perform the exit animation.

> This section explains _what_ being frozen means, but it does not explain _why_ this is a feature of this
> proposal. For more on the why, continue reading.

### Animations Dependent on State

It's common for animations to be dependent on application state. Sometimes, this state does not change, so props can be used. For instance,
to stagger an exit animation based on a child's index, developers can pass the index in as a prop:

```js
function ListItem({ index }) {
  useDeferredUnmount(() => new Promise(resolve => {
    setTimeout(() => {
      animateOut.finally(resolve);
    }, index * 50);
  }));

  return (<div>Hello</div>);
}
```

The fact the prop is frozen during the unmounting phase will not affect the animation.

Other times, a developer may need to use a value that has changed after the unmounting phase has begun. In these situations,
refs must be used. For instance, this component has a semi-complex exit animation that can be configured _during_ the unmounting
phase using a passed-in ref:

```js
function ListItem({ exitToRef }) {
  useDeferredUnmount(() => {
    // First, we slide the list item up or down depending on the value of
    // `exitToRef`.
    const start = exitToRef.current === 'down' : animateDown() : animateUp();

    return start.then(() => {
      // Once that is done, we slide it left or right depending on the
      // value of the ref (which may have changed).
      const end = exitToRef.current === 'left' : animateLeft() : animateRight();
      return end;
    });
  })
}
```

### The Child and its Parent

During the unmounting phase, the component will remain a child of its parent. This has a number of implications.

#### Internal Children Iterations

If React iterates the children of a component internally, then the unmounting components will be a
part of that loop, unless they are filtered out.

> Note: the impact of this is lost on me, as I do not have that deep an understanding of React internals.

#### Positioning New Sibling Elements

A developer may return new sibling elements in the spot where the unmounting child used to be. This raises the question of
where the new elements should be rendered within the DOM.

Consider a parent that renders three children: [`A`, `B`, `C`]. It then unmounts `B`, but `B` uses `useDeferredUnmount` and
enters the unmounting phase. Shortly after, the parent renders a new element `D` in the spot where `B` used to be.

In our parent's return value, we then have [`A`, `D`, `C`], but in the DOM we have the three of those, in addition to `B` which
is still unmounting. What is the appropriate order for these elements? There are two choices: [`A`, `B`, `D`, `C`] or [`A`, `B`, `C`, `D`].

In this proposal, I suggest that all additional children are rendered _after_ the unmounting nodes. This leaves it up to the developer
to ensure that the elements are _displayed_ in the correct order (i.e.; using `flex-order`) and/or perceived by assistive technology in
the correct order (i.e.; `aria-flowto`).

> Note: I am unfamiliar with the support of `aria-flowto`, so this suggestion may be insufficient for accessibility.

#### Keys

Keys are used in React to give elements a stable identity. If a child element is rendered that has the same key as an unmounting component,
then the unmounting component will be immediately unmounted. Developers will need to make sure that they do not reuse keys when components
are unmounting.

#### React.Children

Today, there is a guarantee that the size of `React.Children.count` will match what appears in the DOM. `useDeferredUnmount` breaks
this contract, which could cause problems in existing components. Let's explore this in more detail.

A common pattern in React is to provide a component that passes its children through. For instance:

```js
function MyComponent({ children }) {
  // Do something useful...

  return children;
}
```

Occasionally, a component like this will require that `children` has a length of 1 (it may use `React.Children.only` to enforce
this). One of the most popular examples of this is React Router's [`Router` component](https://reacttraining.com/react-router/web/api/Router/children-node),
which accepts a single node as a child.

There is a small chance that these libraries will break if there are two elements in the tree. Here is a contrived example component that could break when its
children defer unmounting:

```js
function MyComponent({ children }) {
  const elRef = useRef();
  const child = React.Children.only(children);

  useEffect(() => {
    if (elRef.current && typeof elRef.current.querySelector === 'function') {
      const firstAndOnlyChild = elRef.current.querySelector('> *');
      // Do something with `firstAndOnlyChild`...
    }
  });

  return (
    <div ref={elRef}>
      {child}
    </div>
  );
}
```

This component may go against React best practices, and there may be better ways to do this that would not break, but this is just to show a component that
_might_ break if this feature is introduced.

I anticipate the impact of this to be small, but this is worth investigating further because I could be misjudging it.

If there are times when the existence of a single child is truly significant, then the introduction of a new top-level API might help. Consider:

```js
export default React.only(function MyComponent({ children }) {
  return children;
});
```

This component would throw an error if `MyComponent` ever has two children.

At this time, `React.only` is not a feature of this RFC. It is presented here for discussion.

#### Interactions With `useImperativeRef`

This could cause problems, but I am not sure. I have added it to the unresolved questions section below.

### Interactions with `useEffect`

`useEffect` can be used to perform side effects in React applications. One use case of `useEffect` that is
salient to this RFC is setting up a subscription when the component mounts, and then tearing it down when
the component unmounts:

```js
useEffect(() => {
  const unsubscribe = store.listen(onUpdate);

  return () => {
    unsubscribe();
  }
}, []);
```

`onUpdate` in this example may perform an action such as updating the component's state. This hook functions similarly to a
Class component's `componentDidMount` and `componentWillUnmount` hook.

In today's React, there is a clearly-defined mounting and unmounting "_moment_". What this means is that there is no ambiguity about when this
subscription will be set up, and when it will be torn down. Deferred unmounting introduces two "moments" to unmounting: one when the unmounting
phase begins, and one when it ends, which leads to the question: is the teardown at the start of end of the unmounting?

In this proposal, `useEffect`'s return function will be executed at the _start_ of the unmounting phase, and not at the end. This has a number
of benefits.

First, it maintains the symmetry between `componentWillUnmount` and `useEffect`'s return function. If `useEffect`'s return value were called at the
end of the unmounting phase, then it would no longer be called at the same moment as `componentWillUnmount`. This inconsistency could make it more
difficult to learn the Hooks API, and it could make it more difficult to refactor Class components into function components.

It also plays nicely with the fact that state is frozen during the unmounting phase. Developers often update state as a side effect of a
subscription. If `useEffect`'s return function was called at the end of the unmounting phase then console warnings would be emitted when the
subscription attempted to update state in the middle of the unmounting phase.

Thirdly, this, along with freezing components during the unmounting phase, avoids the problem of developers needing to make a choice between cleaning up
subscriptions in `useEffect`'s return value, or in `useDeferredUpdates`. Imagine that state _could_ be updated during the unmounting phase. Developers
might then rely on subscriptions to update state to help with the exit transition. They would then need to clean up the subscription in `useDeferredUnmount`.
This would make `useEffect` no longer as self-contained, which I consider to be a problem.

### What if we don't freeze updates?

There is more to be said about the possibility of not freezing components. Let's consider a world where unmounting components aren't frozen. In this world,
the component still cannot receive prop updates because its element isn't returned by the parent, so there is no system that exists to update its props.

It can, however, still receive updates through Context or state. Because of this, a developer may want to use an effect to manage the exit transition.
Unfortunately, the exit transition occurs _after_ the start of the function returned from `useEffect` is called.

### Uncaught Errors

When there are uncaught errors during rendering, the unmounting phase will be interrupted and the component will immediately unmount. If an
Error Boundary exists, and it continues to render `children` (instead of an error message), then the unmounting phase will continue
uninterrupted.

#### React DevTools

Unmounting components will continue to be displayed in React DevTools, because they are still in the component tree. An indication of some kind could
be introduced that lets developers know which components are in the unmounting phase.

# Drawbacks

I know little about the internals of React, so I am not able to speak on the implementation cost in terms of code size and complexity. My naïve impression
is that this wouldn't be an enormous effort, but I am likely underestimating that.

Although the proposed feature can be implemented in user space, there are a few reasons why this is not ideal.

First, it is really hard to implement it without 

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

Another API that could enable this functionality involves adding additional
functionality to `useEffect`. Consider:

```jsx
import { useEffect } from 'react';

export default function MyComponent() {
  useEffect(() => {
    return () => {
      return new Promise(resolve => {
        setTimeout(resolve, 5000);
      });
    };
  });

  return (
    <div>Hello</div>
  );
}
```

A benefit of this approach is that it does not increase the surface area of the top-level React API.
A downside is that it increases the complexity of `useEffect`, which already has a discernible learning
curve. Another downside is that it may be a breaking change if somebody is currently returning a Promise
from `useEffect`, which today has no effect.

It would also be possible to implement this feature, either with a new Hook or with `useEffect`, by
passing a `done` callback that the developer must call once they are finished with their animation.

```js
import { useDeferredUnmount } from 'react';

export default function MyComponent() {
  useDeferredUnmount(done => {
    setTimeout(done, 5000);
  });

  return (
    <div>Hello</div>
  );
}
```

Another detail of this API could be changed is to not freeze components.

# Adoption strategy

This API will not* be a breaking change, which will allow for React developers to incrementally
adopt it into existing codebases.

Developers may choose to rewrite existing code that uses libraries to defer unmounting. I am
skeptical that a codemod could be written to assist with this refactoring.

It would be wise for React to reach out to maintainers of popular animation libraries, such as
`framer-motion` and `react-spring`, to ensure that this API is compatible with their
approach to motion.

> _*Most likely_. See the "Detailed design" section for examples of where it may break existing code.

# How we teach this

A new section should be added to the ["Hooks API Reference"](https://reactjs.org/docs/hooks-reference.html)
page that describes the usage of this hook. Additionally, i may be appropriate to introduce a
guide under the "Advanced Guides" section of the React website dedicated entirely to animation, which would
include animating components out.

Conceptually, it likely makes the most sense to communicate this as an extension of the behavior of unmounting. Right
now, developers are taught that a component is immediately removed from the DOM when it is not returned as an element
from its parent. This feature tweaks that idea slightly by allowing the child to allow itself to remain mounted for
a period of time.

I suspect that an _understanding_ of this idea will come quickly to developers who already understand mounting and unmounting, but
that the nuances of working with frozen components may require a bit more effort to understand. In particular, I suspect that _a lot_
of people will ask "why."

On the subject of terminology, Concurrent React is already using the word `deferred`, as in the hook
[`useDeferredValue`](https://reactjs.org/docs/concurrent-mode-reference.html#usedeferredvalue). This creates the
opportunity for confusion between these two similarly-named hooks which are used for
entirely different features of React. It may be better to use another name for this hook, like `useDelayedUnmount`, to
avoid confusion.

# Unresolved questions

Should unmounting components be frozen or not?

Should there be a fallback system in place that unmounts nodes after some period of time if the Promise is
never resolved?

Should there be a way to tell React whether to place new child nodes before or after an unmounting node in the DOM?

Should a new lifecycle method be added to Class components to enable this functionality
there as well? `componentWillDeferUnmount` could be introduced, and it could behave the
same way as `useDeferredUnmount` when a Promise is returned from it.

How does this interact with `useImperativeRef`? Consider a parent that passes a ref down to a single child element.
When it stops returning that element, the child could enter the unmounting phase. Shortly after, the parent could mount a
new child passing that same ref. I suspect that we would want the first child's ref to continue to equal its own DOM node,
but I am not sure if this would be the behavior that we get automatically.