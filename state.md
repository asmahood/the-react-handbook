# State

We cannot use local variables to manage a components state for two reasons:

1. Local variables don't persist between renders. When React renders this component a second time, it renders it from scratch - it doesn't consider any changes to the local variables.
2. Changes to local variables won't trigger renders. React doesn't realize it needs to render the component again with the new data. 

To fix these, React has the `useState` hook which provides:

1. A state variable to retain the data between renders.
2. A state setter function to update the variable and trigger React to render the component again.

## Adding a State Variable

First, import the hook:
```javascript
import { useState } from "react";
```

Then use the hook as so:

```javascript
const [index, setIndex] = useState(0);
```

In this example, `index` is the state variable, and `setIndex` is the setter function. The argument passed to `useState` is the initial value of the state.

The convention is to the name the pair like `const [something, setSomething]`.

A hook can have any number of state variables. It is a good idea to have multiple state variables if their state is unrelated. But if you find that you foten change two state variables together, it might be easier to combine them into one. For example, if you have a form with many fields, it's more covenient to have a single state variable that holds an object than one state variable per field.

State is local to a component instance on the screen. If you render the same component twice, each copy will have completely isolated state.

### Use State in Depth
Here is how `useState` works in action:

1. The component renders the first time. Because `0` was passed to `useState` as the initial value for index, it will return `[0, setIndex]`
2. You update the state. Using the `setIndex(1)` will tell React to remember `index` is `1` now and triggers another render.
3. The component renders a second time. React still sees `useState(0)` but remembers that you set `index` to `1` so it returns `[1, setIndex]`

<details>
    <summary><h4>React In Depth</h4></summary>

    Hooks rely on a stable call order on every render of the same component. This enables hooks to not need any information such as an identifier for state variables. This is why hooks are required to be at the top-level of a component. Internally, React holds an array of state pairs for every component. It also maintains the current pair index. Each time `useState` is called, Reac tgives the next state pair and increments the index.
</details>

## State is a snapshot

State is not like a regular variable that disappears after the function returns. State actually lives in React itself. When React calls a component, it gives a snapshot of the state for that particular render. The component returns a snapshot of the UI with a fresh set of props and event handlers in its JSX, all calculated using the state values from that render.

To showcase this fact, look at the following example:
```javascript
import { useReact } from "react";

export default function Counter() {
    const [number, setNumber] = useState(0);

    return (
        <>
            <h1>{number}</h1>
            <button onClick={() => {
                setNumber(number + 1);
                setNumber(number + 1);
                setNumber(number + 1);
            }}>+3</button>
        </>
    );
}
```
If state was like a variable, you would expect that this will increment `number` 3 times since `setNumber` is called three times. However, clicking the button would result in `number` will only be incremented once.

This is because **setting state only changes it for the next render**. During the first render `number` was 0. In *that render's* `onClick` handler, the value of `number` is still `0`, event after `setNumber(number + 1)` was called. **A state variable's value never changes within a render, even if it's event handler is async**.

### Queueing A Series of State Updates

To get the intended functionality, we need to use a state updater function. Before going into this, we will first discuss how React batches state updates.

#### React Batches State Updates

React waits until all code in the event handlers has run before processing state updates. This lets you update multiple state variables without triggering too many re-renders. This behaviour is known as **batching**. React **does not batch** across multiple intentional events like clicks - each click is handled separately. React only does batching when it is safe to do so.

#### Updating the same state multiple times before the next render
If you want to update the same state variable mulitple times before the next render, itnstead of passing the next state value like `setNumber(number + 1)`, you can pass a function that calculates the next state based on the previous one in the queue, like `setNumber(n => n + 1)`. This is called a **updater function**. When you pass it to a state setter:

1. React queues this function to be processed after all the other code in the event handler has run.
2. During the next render, React goes through the queue and gives you the final updated state.

If you use both in the same handler, React will queue each either as *replace with X* or adding a function to the queue.

## Updating Objects in State

When you want to update an object, you need to create a new one or make a copy of an existing one, then set the state to use that copy. **DO NOT MODIFY THE STATE IN PLACE**. You should treat objects as if they were immutable. In other words, treat any JavaScript object that you put into state as read-only. Modifying and object in place will not trigger a re-render, thus no changes would appear. An example that will cause a re-render:

```javascript
const [position, setPosition] = useState({
    x: 0,
    y: 0,
});

onPointerMove={e => {
    setPosition({
        x: e.clientX,
        y: e.clientY
    })
}}
```

Or, another option when you need to include existing data as part of the new state, you can use object spread to copy the object. For example, when updating only one field in a form:

```javascript
import { useState } from "react";

export default function Form() {
    const [person, setPerson] = useState({
        firstName: "Barbara",
        lastName: "Hepworth",
        email: "bhepworth@sculpture.com"
    });

    function handleFirstNameChange(e) {
        setPerson({
            ...person,
            firstName: e.target.value
        })
    }

    // similar for other fields

    return (
        <form>
            <label>First name:</label>
            <input value={person.firstName} onChange={handleFirstNameChange} />
        </form>
    );
}
```
*Note: the `...` spread syntax is "shallow", it only copies things one level deep.*

You can also use a single event handler for multiple fields by using `[]` to specify a property with a dynamic name:
```javascript
setPerson({
    ...person,
    [e.target.name]: e.target.value,
});
```

### Updating a nested object

This takes a similar approach to updating a non-nested object. For example:
```javascript
const [person, setPerson] = useState({
    name: "Niki de Saint Phalle",
    artwork: {
        title: "Blue Nana",
        city: "Hamburg",
        image: "https://i.imgur.com/Sd1AgUOm.jpg",
    },
});
```

To update the `city` in `artwork`, you would first need to create a new `artwork` object and then produce a new person object:
```javascript
const nextArtwork = { ...person.artwork, city: "New Delhi" };
const nextPerson = { ...person, artwork: nextArtwork };

// OR

setPerson({
    ...person,
    artwork: {
        ...person.artwork,
        city: "New Delhi",
    },
});
```

### Using Immer
If your state is deeply nested, you might need to flatten it. But if you don't want to change your state structure, you might prefer a shortcut to nested spreads. **Immer** is a popular library that lets you wirte using the convenient but mutating syntax and takes care of producing the copies for you. To use it:

- Run `npm install use-immer` to add Immer as a dependency
- Then replace `import { useState } from "react"` with `import { useImmer } from "use-immer"`

Then use the hook as so:
```javscript
import { useImmer } from "use-immer";

export default function Form() {
    const [person, updatePerson] = useImmer({
        name: "Niki de Saint Phalle",
        artwork: {
            title: "Blue Nana",
            city: "Hamburg",
            image: "https://i.imgur.com/Sd1AgUOm.jpg",
        }, 
    });

    function handleNameChange(e) {
        updatePerson(draft => {
            draft.name = e.target.value;
        });
    }
}
```

## Updating Arrays in State

Similar to objects, arrays are mutable in JavaScript, but you should treat them as immutable when you store them in state. Just like objects, to update an array stored in state, you need to create a new one or make a copy of an existing one and then set state to use the new array.

To create a new array, call non-mutating methods like `filter()` and `map()`. Some other methods that you should use/avoid:

| | Avoid (mutatates the array) | Prefer (returns a new array) |
|-|-----------------------------|------------------------------|
| adding | `push`, `unshift` | `concat`, `[...arr]` spread syntax |
| removing | `pop`, `shift`, `splice` | `filter`, `slice` |
| replacing | `splice`, `arr[i] = ...` assignemnt | `map` |
| sorting | `reverse`, `sort` | copy the array first |

Alternatively you can use Immer which lets you use methods from both columns.

Some examples of using these methods:

```javascript
// Adding to an array
setArtists([
    ...artists,
    { id: nextId++, name: name },
]);

// Removing from an array
setArtists(
    artists.filter(a => a.id !== artist.id);
);

// Transforming
const nextShapes = shapes.map(shape => {
    if (shape.type === "square") {
        return shape;
    } else {
        return {
            ...shape,
            y: shape.y + 50;
        };
    }
});

setShapes(nextShapes);

// Replacing
const nextCounters = counters.map((c, i) => i === index ? c + 1 : c);
setCounters(nextCounters);

// Inserting into an array
const nextArtists = [
    ...artists.slice(0, 1),
    { id: nextId++, name: name },
    ...artists.slice(1)
];
setArtists(nextArtists);

// Reverse or Sort an Array
const nextList = [...list];
nextList.reverse();
nextList.sort();
setList(nextList);
```

*Note: even if you copy an array, you can't mutate existing items inside of it directly*.

This is because copying is shallow - the new array will contain the same items as the original one. So if you modify an object inside the copied array, you are mutating the existing state.

### Updating Objects inside Arrays

When updating nested state, you need to create copies from the point where you want to update, and all the way up to the top level. One solution is to use `map` to substitute an old item with its updated version without mutation:
```javascript
setMyList(myList.map(artwork => {
    if (artwork.id === artworkId) {
        // Create a new object with changes
        return { ...artwork, seen: nextSeen }
    } else {
        // No changes
        return artwork;
    }
}));
```

Or you could use Immer to mutate the objects.
