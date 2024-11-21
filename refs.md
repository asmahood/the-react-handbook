# Refs

When you want a component to remember some information but don't want that information to trigger new renders, you can use a ref.

## Adding a ref to a component

You can add a ref to a component by importing the `useRef` hook from React:
```javascript
import { useRef } from "react";
```
Inside the component, call the `useRef` hook and provide an initial value as the only argument:
```javascript
const ref = useRef(0);
```
It will return an object that looks liek the following:
```javascript
{
    current: 0 // The value you passed to useRef
}
```

You can access the current value of the ref using the `ref.current` property. This value is mutable. Note that the component **doesn't** re-render with every increment. Like state, refs are retained by React between re-renders.

### Example: Building a Stopwatch
In order to display how much time has passed since the user pressed "Start", you will need to keep track of when the `Start` button was pressed and what the current state is. This information is used for **rendering**, so we'll keep it in state.

```javascript
const [startTime, setStartTime] = useState(null);
const [now, setNow] = useState(null);
```

When the user presses `Start`, we'll use the `setInterval` function to update the time every 10 milliseconds.

```javascript
import { useState } from "react";

export default function Stopwatch() {
    const [startTime, setStartTime] = useState(null);
    const [now, setNow] = useState(null);

    function handleStart() {
        // Start counting
        setStartTime(Date.now());
        setNow(Date.now());

        setInterval(() => {
            // Update the current time every 10ms
            setNow(Date.now());
        }, 10);
    }

    let secondsPassed = 0;
    if (startTime != null && now != null) {
        secondsPassed = (now - startTime) / 1000;
    }

    return (
        <>
            <h1>Time passed: {secondsPassed.toFixed(3)}</h1>
            <button onClick={handleStart}>
                Start
            </button>
        </>
    );
}
```

When the `Stop` button is pressed we need to cancel the existing interval so that it stops updating the `now` state variable. We can do this by calling `clearInterval`, but you need to give it the interval ID that was previously returned by `setInterval` call when the user pressed `Start`. Since the interval ID is **not used for rendering** we can keep it in a ref:

```javascript
import { useState, useRef } from "react";

export default function Stopwatch() {
    const [startTime, setStartTime] = useState(null);
    const [now, setNow] = useState(null);
    const intervalRef = useRef(null);

    function handleStart() {
        // Start counting
        setStartTime(Date.now());
        setNow(Date.now());

        clearInterval(intervalRef.current);
        intervalRef = setInterval(() => {
            // Update the current time every 10ms
            setNow(Date.now());
        }, 10);
    }

    function handleStop() {
        clearInterval(intervalRef.current);
    }

    let secondsPassed = 0;
    if (startTime != null && now != null) {
        secondsPassed = (now - startTime) / 1000;
    }

    return (
        <>
            <h1>Time passed: {secondsPassed.toFixed(3)}</h1>
            <button onClick={handleStart}>
                Start
            </button>
            <button onClick={handleStop}>
                Stop
            </button>
        </>
    );
}
```

**When a piece of information is used for rendering, keep it in state if necessary. When a piece of information is only needed by event handlers and changing it doesn't require a re-render, using a ref may be more efficient.**

## Differences between refs and state
Here's how state and refs compare:

| refs | state |
| ---- | ----- |
| `useRef(initialValue)` returns `{ current: initialValue }` | `useState(initialValue)` returns the current value of a state variable and a state setter function (`[value, setValue]`) |
| Doesn't trigger a re-render when you change it | Triggers a re-render when you change it |
| Mutable - you can modify and update `current`'s value outside of the rendering process | "Immutable" - you must use the state setting function to modify state variables to queue a re-render |
| You shouldn't read (or write) the current value during rendering | You can read state at any time. However, each redner has its own snapshot of state which does not change |

## When to use refs

Typically, you will use a ref when your component needs to "step outside" React and communicate with external APIs - often a browser API that won't impact the appearance of the component. Here are a few of these rare situations:

- Storing timeout IDs
- Storing and manipulating DOM elements
- Storing other objects that aren't necessary to calculate the JSX.

## Best practices for refs

- Treat refs as an escape hatch. Refs are useful when you work with external systems or browser APIs. If much of your application logic and data flow relies on refs, you might want to rethink your approach
- Don't read or write `ref.current` durring rendering. If some information is needed during rendering, use `state` instead.

## Refs and the DOM
You can point a ref to any value. However, the most common use case for a ref is to access a DOM element. WHen you pass a ref to a `ref` attribute in JSX, React will put the corresponding DOM element into `myRef.current`. Once the element is remvoed from the DOM, React will update `myRef.current` to be `null`.

### Manipulating the DOM with Refs
To access a DOM node managed by React, import the `useRef` hook and declare a ref inside the component:
```javascript
import { useRef } from "react";

export default function App() {
    const ref = useRef(null);
}
```
Finally, pass your ref as the `ref` attribute to the JSX tag for which you want to get the DOM node:
```javascript
<div ref={ref}>
```
Initially, `ref.current` will be null. When React creates a DOM node for this `<div>` React will put a reference to this node into `ref`. Now you can access this DOM node from event handlers and use built-in browser APIs defined on it:
```javascript
ref.current.scrollIntoView();
```

### Managing a list of refs
If you have an unknown number of refs you need to manage, you can pass a function to the `ref` attribute. This is called a `ref` callback. React will call your ref callback with the DOM node when it's time to set the ref, and with `null` when its time to clear it. This lets you maintain your own array or a Map and access any ref by its index or some kind of id. For example:
```javascript
// ...

const itemsRef = useRef(null);

function scrollToCat(cat) {
    const map = getMap();
    const node = map.get(cat);
    node.scrollIntoView({
        behaviour: "smooth",
        block: "nearest",
        inline: "center",
    });
}

function getMap() {
    if (!itemsRef.current) {
        itemsRef.current = new Map();
    }

    return itemsRef.current;
}

// ...

return (
    // ...
    <ul>
        <li
            key={cat}
            ref={(node) => {
                const map = getMap();
                if (node) {
                    map.set(cat, node);
                } else {
                    map.delete(node);
                }
            }}
        >
            // ...
        </li>
    // ...
);
```

### Accesing another component's DOM nodes

If you try to put a ref on your own component you will get `null`. This is because React does not let a component access the DOM nodes of other components. Not even its own children. Instead, components that want to expose their DOM nodes have to opt in by specifying that it "forwards" its ref to one of its children using the `forwardRef` API:

```javascript
const MyInput = forwardRef((props, ref) => {
    return <input {...props} ref={ref} />;
})
```

Then you can use it by doing:

```javascript
<MyInput ref={inputRef} />
```

To restrict what can be altered by a `forwardRef`, you can utilize the `useImperativeHandle` hook:

```javascript
import {
  forwardRef, 
  useRef, 
  useImperativeHandle
} from 'react';

const MyInput = forwardRef((props, ref) => {
  const realInputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    // Only expose focus and nothing else
    focus() {
      realInputRef.current.focus();
    },
  }));
  return <input {...props} ref={realInputRef} />;
});

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput ref={inputRef} />
      <button onClick={handleClick}>
        Focus the input
      </button>
    </>
  );
}
```

Here `useImperativeHandle` instructs React to provide your own special object as the value of a ref to the parent component. So `inputRef.current` will only have the focus method.

## When React attaches the refs
Every update is split into two phases:

1. During render, when React calls components to figure out what goes on the screen
2. During commit, when React applies changes to the DOM

In general, you don't want to access refs during rendering. During render, the DOM nodes have not yet been created so `ref.current` will be `null`. React sets `ref.current` during the commit. Because of this, you'll usually want to access refs from event handlers. If there is no event to do it in, you might need an effect.

<details>
<summary>React Deep Dive</summary>

You can force React to update ("flush") the DOM synchronously. To do this import `flushSync` from `react-dom` and wrap the state update into a `flushSync` call:

```javascript
flushSync(() => {
    setTodos([...todos, newTodo]);
});
listRef.current.lastChild.scrollIntoView();
```
</details>

## Best practices for DOM manipulation with refs

You should only use refs when you have to step outside of React. Common examples include managing focus, scroll position, or calling browser APIs that React does not expose. Avoid changing DOM nodes managed by React. You can safely modify parts of the DOM that React has no reason to update.
