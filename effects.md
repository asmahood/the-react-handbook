# Effects

Before getting to effects, we need to be familiar with two types of logic inside React components:

1. **Rendering code** lives at the top level of the component. This is where the component takes props and state and transform them and return the JSX you want to see on the screen. Rendering code **must** be pure.
2. **Event handlers** are nested functions inside components that do things rather than just calculate them. Event handlers contain side effects (different from effects in this context).

However, sometimes these two types are not enough. Consider a `ChatRoom` component that must connect to a chat server whenever it's visible on the screen. Connecting to a server is not a pure calculation (it's a side effect) so it can't happen during rendering. However, there is no single particular event like a click that causes `ChatRoom` to be displayed.

**Effects** let you specify side effects that are caused by rendering itself, rather than a particular event. Sending a message is an event because it is caused by a user hitting a button. Setting up a server connection is an effect because it should happen no matter which interaction caused the component to appear. **Effects run at the end of a commit after the screen updates**.

You might not need an effect. Effecs are typically used to step out of React code and synchronize with some external system. This includes browser APIs, third-party widgets, network, and so on. If you effect only adjusts some state based on other state, you might not need an effect.

## Writing an effect

To write an Effect, follow these steps:

1. **Declare an Effect**: By default, the Effect will run after every commit
2. **Specify the Effect dependencies**: Most Effects should only re-run when needed rather than after every render. You can control when the effect runs by specifying dependencies.
3. **Add cleanup if needed**: Some Effects need to specify how to stop, undo, or clean up whatever they wre doing. For example, "connect" needs "disconnect", "subscribe" needs "unsubscribe", "fetch" needs "cancel" or "ignore". You can do this by returning a cleanup function.

### Step 1: Declare an Effect
To declare an Effect, import the `useEffect` hook:
```javascript
import { useEffect } from "react";
```

Then call it at the top level of the component:
```javascript
function MyComponent() {
    useEffect(() => {
        // Code here will run after every render
    });
}
```
Everytime the component renders, React will update the screen **then** run code inside the `useEffect`.

### Step 2: Specify the Effect dependencies

As we've seen above, by default Effects run after every redner. Often this is not what we want:
- Sometimes the Effect can be slow so we might want to skip it if its unnecessary.
- Sometimes it's wrong such as playing an animation on every keystroke.

You can tell React to **skip unnecessarily re-running the Effect** by specifying an array of dependencies as the second argument to `useEffect`. Start by adding an empty `[]` array.
```javascript
useEffect(() => {
// ...
}, [])
```

With an empty `[]` array, the effect will only run when the component is mounted (component appears). If you place values inside the array (for example: `[a, b]`). The effect will run on mount and also if either a or b have changed since the last render.


React will error if there are any variables/state/props that are included inside the `useEffect` that are not in the dependencies array. Specifying dependencies tells React that it should skip re-running the Effect if **all** of the dependencies specified have exactly the same values as they had during the previous render. React compares the dependency values using `Object.is` comparison. Note that you can't choose your dependencies. If you don't want some code to re-run, edit the Effect code itself to not need that dependency.

<details>
<summary>React Deep Dive</summary>

Note that Effects do not need to specify refs in their dependency lists. This is because ref objects have a **stable identity**. React guarantees that you'll always get the same object from the same `useRef` call on every render. It never changes, so it will never by itself cause the Effect to re-run. Therefore, it does not matter whether you include it or not. The `set` functions returned by `useState` also have stable identity so they can also be omitted from the dependencies too. 

Omitting always-stable dependencies only works when the linter can "see" that the object is stable. For example, if `ref` was passed from a parent component, you would have to specify it in the dependency array. However, this is good because you can't know whether the parent component always passes the same ref, or passes on of several refs conditionally.
</details>

### Step 3: Add cleanup if needed

If you need to cleanup after an effect, such as disconnecting from a connection, return a function from the Effect:
```javascript
useEffect(() => {
    const connection = createConnection();
    connection.connect();
    return () => {
        connection.disconnect();
    };
}, []);
```
This code will print:
- Connecting...
- Disconnecting...
- Connecting...

React will call the cleanup function each time before the Effect runs again, and one final time when the component unmounts (gets removed).

*Note: Do not use refs to prevent Effects from firing twice. For example:*
```javascript
const connectionRef = useRef(null);
useEffect(() => {
    // 🚩 This wont fix the bug!!!
    if (!connectionRef.current) {
      connectionRef.current = createConnection();
      connectionRef.current.connect();
    }
}, []);
```
This still has the issue that when the user navigates away, the connection is never closed.

## How to handle the Effect firing twice in development
React intentionally remounts components in development to find bugs like in the last example. This is intended and you should not try to remove this behaviour. You should instead ask how to fix my Effect so that it works after remounting. Usually the answer is to implement a cleanup function. The cleanup function should stop or undo whatever the effect was doing. The rule of thumb is that the user shouldn't be able to distinguish between the Effect running once (as in production) and a `setup -> cleanup -> setup` sequence (as you'd see in development). Most effects will fit into one of the following common patterns:

### Controlling non-React widgets
Sometimes you need to add UI widgets that aren't written in React. For example, adding a map component to your page:
```javascript
useEffect(() => {
  const map = mapRef.current;
  map.setZoomLevel(zoomLevel);
}, [zoomLevel]);
```

There is no cleanup needed in this case because calling the Effect twice will set the zoom level to the same value, which doesn't matter. Another example where a cleanup function is needed is when you are calling the `showModal` method on a build in `<dialog>` element, which will throw an error if it is called twice without closing the dialog. To fix this:
```javascript
useEffect(() => {
  const dialog = dialogRef.current;
  dialog.showModal();
  return () => dialog.close();
}, []);
```

### Subscribing to events

If your Effect subscribes to something, the cleanup function should unsubscribe:
```javascript
useEffect(() => {
  function handleScroll(e) {
    console.log(window.scrollX, window.scrollY);
  }
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

### Triggering animations

If your effect animates something in, the cleanup function should reset the animation to the initial values:
```javascript
useEffect(() => {
  const node = ref.current;
  node.style.opacity = 1; // Trigger the animation
  return () => {
    node.style.opacity = 0; // Reset to the initial value
  };
}, []);
```

### Fetching data

If your effect fetches something, the cleanup function should either abort the fetch or ignore its result:
```javascript
useEffect(() => {
  let ignore = false;

  async function startFetching() {
    const json = await fetchTodos(userId);
    if (!ignore) {
      setTodos(json);
    }
  }

  startFetching();

  return () => {
    ignore = true;
  };
}, [userId]);
```
You can't "undo" a network request that already happened, but your cleanup function should ensure that the fetch does not keep affecting the application.

In development, you will see two fetches in the Network tab. In production, there will only be one request. 

It is recommended to use either a built-in data fetching mechanism from a framework or consider using or building a client-side cache library like React Query, useSWR, and React Router 6.4+. Writing `fetch` calls inside Effects is a very manual approach with significant downsides:
- Effects don't run on the server. This means that the initial server-rendered HTML will only include a loading state with no data. The client computer will have to download all JavaScript and render your app only to discover that now it needs to load the data. This is not very efficient.
- Fetching directly in Effects makes it easy to create “network waterfalls”. You render the parent component, it fetches some data, renders the child components, and then they start fetching their data. If the network is not very fast, this is significantly slower than fetching all data in parallel.
- Fetching directly in Effects usually means you don’t preload or cache data. For example, if the component unmounts and then mounts again, it would have to fetch the data again.
- It’s not very ergonomic. There’s quite a bit of boilerplate code involved when writing fetch calls in a way that doesn’t suffer from bugs like race conditions.

### Sending analytics
Consider this code that sends an analytics event on the page visit:
```javascript
useEffect(() => {
  logVisit(url); // Sends a POST request
}, [url]);
```

In development, `logVisit` will be called twice for every URL, so you might be tempted to try to fix that. It is recommended to keep this as is. There is no user-visible difference between running it once versus twice.

### Not an Effect: Initializing the application
Some logic should only run once when the application starts. You can put it outside your components:
```javascript
if (typeof window !== 'undefined') { // Check if we're running in the browser.
  checkAuthToken();
  loadDataFromLocalStorage();
}

function App() {
  // ...
}
```
This guarantees that such logic only runs once after the browser loads the page.

## Removing unnecessary effects
There are two common cases where you don't need Effects:

1. You don't need Effects to transform data for rendering. To avoid unnecessary redner passes, transform all the data at the top level of your components. That code will automatically re-run whenever your props or state change.
2. You don't need Effects to handle user events.

You do need Effects to synchronize with external systems such as syncing with jQuery widgets or fetching data. Let's look at some examples:

### Updating state based on props or state
Suppose you have two state variables `firstName` and `lastName` and you want to calculate a `fullName` from them and you want `fullName` to update whenever you change either state variable. **You do not want to add a `fullName` state variable and update it in an effect**:
```javascript
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');

  // 🔴 Avoid: redundant state and unnecessary Effect
  const [fullName, setFullName] = useState('');
  useEffect(() => {
    setFullName(firstName + ' ' + lastName);
  }, [firstName, lastName]);
  // ...
}
```
This is more complicated and inefficient. React will do an entire render pass with a stale value for `fullName` then immediately re-renders with the updated value. Remove the state variable and the Effect and replace it with a normal variable:
```javascript
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');
  // ✅ Good: calculated during rendering
  const fullName = firstName + ' ' + lastName;
  // ...
}
```
**Rule of thumb: When something can be calculated from the existing props or state, don't put it in state. Instead calculate it during rendering.**

### Caching expensive calculations
You might feel tempted to store the result of some filtering operation in state and updating it in an Effect:
```javascript
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // 🔴 Avoid: redundant state and unnecessary Effect
  const [visibleTodos, setVisibleTodos] = useState([]);
  useEffect(() => {
    setVisibleTodos(getFilteredTodos(todos, filter));
  }, [todos, filter]);

  // ...
}
```
Like above, this is unnecessary:
```javascript
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');
  // ✅ This is fine if getFilteredTodos() is not slow.
  const visibleTodos = getFilteredTodos(todos, filter);
  // ...
}
```
Usually this is fine. But if `getFilteredTodos()` or you have many `todos` you don't want to recalculate this on every re-render when some unrelated state changes. You can cache (or "memoize") an expensive calculation by wrapping it in a `useMemo` hook:

```javascript
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');
  const visibleTodos = useMemo(() => {
    // ✅ Does not re-run unless todos or filter change
    return getFilteredTodos(todos, filter);
  }, [todos, filter]);
  // ...
}
```
This tells React that you don't want the inner function to re-run unless either `todos` or `filter` have changed. The function you wrap in `useMemo` runs during **rendering** so this only works for pure calculations.

<details>
<summary>React Deep Dive</summary>

In general, unless you're creating or looping over thousands of objects, it's probably not expensive. If you want to get more confidence, you can add a `console.time` to measure the time spent in a piece of code. If the overall logged time adds up to a significant amount (say `1ms` or more), it might make sense to memoize that calculation. 
</details>

### Resetting all state when a prop changes
Suppose you want to reset some state when a value changes to prevent potential bugs of state being remembered. You might want to clear the state using an Effect:
```javascript
export default function ProfilePage({ userId }) {
  const [comment, setComment] = useState('');

  // 🔴 Avoid: Resetting state on prop change in an Effect
  useEffect(() => {
    setComment('');
  }, [userId]);
  // ...
}
```
This is inefficient because `ProfilePage` and its children will first render with the `stale` value, and then render again. It is also complicated because you'd need to this in every component that has some state inside `ProfilePage`. Instead, you can tell React that each user's profile is conceptually a different profile by giving it an explicit key. Split your component in two and pass a `key` attribute from the outer component to the inner one:
```javascript
export default function ProfilePage({ userId }) {
  return (
    <Profile
      userId={userId}
      key={userId}
    />
  );
}

function Profile({ userId }) {
  // ✅ This and any other state below will reset on key change automatically
  const [comment, setComment] = useState('');
  // ...
}
```
Normally, React preserves the state when the same component is rendered in the same spot. By passing `userId` as a `key`, React will treat two `Profile` components with different `userId` as two different components that should not share any state.

### Adjusting some state when a prop changes
Sometimes you might want to reset or adjust a part of the state on a prop change, but not all of it:
```javascript
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selection, setSelection] = useState(null);

  // 🔴 Avoid: Adjusting state on prop change in an Effect
  useEffect(() => {
    setSelection(null);
  }, [items]);
  // ...
}
```
Every time `items` change, the `List` and its child components will render with a `stale` selection value at first. Then React will update the DOM and run the Effects. Finally, the `setSelection` call will cause another re-render of the `List` and its child components, restarting this whole process again. Instead, adjust the state directly during rendering:
```javascript
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selection, setSelection] = useState(null);

  // Better: Adjust the state while rendering
  const [prevItems, setPrevItems] = useState(items);
  if (items !== prevItems) {
    setPrevItems(items);
    setSelection(null);
  }
  // ...
}
```
Although this pattern is more efficient than an Effect, most components shouldn’t need it either. No matter how you do it, adjusting state based on props or other state makes your data flow more difficult to understand and debug. Always check whether you can reset all state with a key or calculate everything during rendering instead. For example, instead of storing (and resetting) the selected item, you can store the selected item ID:
```javascript
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selectedId, setSelectedId] = useState(null);
  // ✅ Best: Calculate everything during rendering
  const selection = items.find(item => item.id === selectedId) ?? null;
  // ...
}
```

### Sharing logic between event handlers
Let’s say you have a product page with two buttons (Buy and Checkout) that both let you buy that product. You want to show a notification whenever the user puts the product in the cart. Calling showNotification() in both buttons’ click handlers feels repetitive so you might be tempted to place this logic in an Effect:
```javascript
function ProductPage({ product, addToCart }) {
  // 🔴 Avoid: Event-specific logic inside an Effect
  useEffect(() => {
    if (product.isInCart) {
      showNotification(`Added ${product.name} to the shopping cart!`);
    }
  }, [product]);

  function handleBuyClick() {
    addToCart(product);
  }

  function handleCheckoutClick() {
    addToCart(product);
    navigateTo('/checkout');
  }
  // ...
}
```
This Effect is unnecessary. It will also most likely cause bugs. For example, let’s say that your app “remembers” the shopping cart between the page reloads. If you add a product to the cart once and refresh the page, the notification will appear again. It will keep appearing every time you refresh that product’s page. This is because product.isInCart will already be true on the page load, so the Effect above will call showNotification().

**When you’re not sure whether some code should be in an Effect or in an event handler, ask yourself why this code needs to run. Use Effects only for code that should run because the component was displayed to the user.**

 In this example, the notification should appear because the user pressed the button, not because the page was displayed! Delete the Effect and put the shared logic into a function called from both event handlers:
 ```javascript
function ProductPage({ product, addToCart }) {
  // ✅ Good: Event-specific logic is called from event handlers
  function buyProduct() {
    addToCart(product);
    showNotification(`Added ${product.name} to the shopping cart!`);
  }

  function handleBuyClick() {
    buyProduct();
  }

  function handleCheckoutClick() {
    buyProduct();
    navigateTo('/checkout');
  }
  // ...
}
 ```

### Sending a POST request
Looking at the following example:
```javascript
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');

  // ✅ Good: This logic should run because the component was displayed
  useEffect(() => {
    post('/analytics/event', { eventName: 'visit_form' });
  }, []);

  // 🔴 Avoid: Event-specific logic inside an Effect
  const [jsonToSubmit, setJsonToSubmit] = useState(null);
  useEffect(() => {
    if (jsonToSubmit !== null) {
      post('/api/register', jsonToSubmit);
    }
  }, [jsonToSubmit]);

  function handleSubmit(e) {
    e.preventDefault();
    setJsonToSubmit({ firstName, lastName });
  }
  // ...
}
```
The analytics POST request should remain in an Effect. This is because the reason to send the analytics event is that the form was displayed. However, the `/api/register` POST request is not caused by the form being displayed. You only want to send the request at one specific moment in time: when the user presses the button. It should only ever happen on that particular interaction. Delete the second Effect and move that POST request into the event handler:
```javascript
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');

  // ✅ Good: This logic runs because the component was displayed
  useEffect(() => {
    post('/analytics/event', { eventName: 'visit_form' });
  }, []);

  function handleSubmit(e) {
    e.preventDefault();
    // ✅ Good: Event-specific logic is in the event handler
    post('/api/register', { firstName, lastName });
  }
  // ...
}
```

When you choose whether to put some logic into an event handler or an Effect, the main question you need to answer is what kind of logic it is from the user’s perspective. If this logic is caused by a particular interaction, keep it in the event handler. If it’s caused by the user seeing the component on the screen, keep it in the Effect.

### Chains of computations

Sometimes you might feel tempted to chain Effects that adjust a piece of state based on other state:
```javascript
function Game() {
  const [card, setCard] = useState(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);
  const [isGameOver, setIsGameOver] = useState(false);

  // 🔴 Avoid: Chains of Effects that adjust the state solely to trigger each other
  useEffect(() => {
    if (card !== null && card.gold) {
      setGoldCardCount(c => c + 1);
    }
  }, [card]);

  useEffect(() => {
    if (goldCardCount > 3) {
      setRound(r => r + 1)
      setGoldCardCount(0);
    }
  }, [goldCardCount]);

  useEffect(() => {
    if (round > 5) {
      setIsGameOver(true);
    }
  }, [round]);

  useEffect(() => {
    alert('Good game!');
  }, [isGameOver]);

  function handlePlaceCard(nextCard) {
    if (isGameOver) {
      throw Error('Game already ended.');
    } else {
      setCard(nextCard);
    }
  }

  // ...
```
There are two mains problem with this:

1. **Efficiency:** The component (and its children) have to re-render between each `set` call in the `chain`. In the example above, in the worst case (`setCard` -> render -> `setGoldCardCount` -> render -> `setRound` -> render -> `setIsGameOver` -> render) there are three unnecessary re-renders of the tree below.
2. **Complexity:** As the code evolves, you will run into cases where the chain you wrote doesn't fit the new requirements.

In this case, it's better to calculate what you can during rendering and adjust state in the event handler:
```javascript
function Game() {
  const [card, setCard] = useState(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);

  // ✅ Calculate what you can during rendering
  const isGameOver = round > 5;

  function handlePlaceCard(nextCard) {
    if (isGameOver) {
      throw Error('Game already ended.');
    }

    // ✅ Calculate all the next state in the event handler
    setCard(nextCard);
    if (nextCard.gold) {
      if (goldCardCount <= 3) {
        setGoldCardCount(goldCardCount + 1);
      } else {
        setGoldCardCount(0);
        setRound(round + 1);
        if (round === 5) {
          alert('Good game!');
        }
      }
    }
  }

  // ...
```

In some cases, the chain of effects is appropriate. For example, imagine a form with multiple dropdowns where the options of the next dropdown depend on the selected value of the previous dropdown. Since you are syncing with the network, the chain is appropriate.

### Initializing the application
Some logic should only run once when the app loads. You might be tempted to place it in an Effect in the top-level component:
```javascript
function App() {
  // 🔴 Avoid: Effects with logic that should only ever run once
  useEffect(() => {
    loadDataFromLocalStorage();
    checkAuthToken();
  }, []);
  // ...
}
```
However, this will run twice in development, which could cause some issues. If some logic must be run once per app load rather than once per component mount, add a top-level variable to track whether it has already executed:
```javascript
let didInit = false;

function App() {
  useEffect(() => {
    if (!didInit) {
      didInit = true;
      // ✅ Only runs once per app load
      loadDataFromLocalStorage();
      checkAuthToken();
    }
  }, []);
  // ...
}
```

You can also run it udring module initialization and before the app renders:
```java
if (typeof window !== 'undefined') { // Check if we're running in the browser.
   // ✅ Only runs once per app load
  checkAuthToken();
  loadDataFromLocalStorage();
}

function App() {
  // ...
}
```

### Notifying parent components about state changes
Say you want to notify a parent component whenever a toggles internal state changes, so you expose an `onChange` event and call it from an Effect:
```javascript
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);

  // 🔴 Avoid: The onChange handler runs too late
  useEffect(() => {
    onChange(isOn);
  }, [isOn, onChange])

  function handleClick() {
    setIsOn(!isOn);
  }

  function handleDragEnd(e) {
    if (isCloserToRightEdge(e)) {
      setIsOn(true);
    } else {
      setIsOn(false);
    }
  }

  // ...
}
```
Like earlier, this is not ideal. The `Toggle` updates its state first, and React updates the screen. Then React runs the Effect, which calls the `onChange` function passed from a parent component. Now the parent component will update its own state, starting another render pass. It would be better to do everything in a single pass:

```javascript
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);

  function updateToggle(nextIsOn) {
    // ✅ Good: Perform all updates during the event that caused them
    setIsOn(nextIsOn);
    onChange(nextIsOn);
  }

  function handleClick() {
    updateToggle(!isOn);
  }

  function handleDragEnd(e) {
    if (isCloserToRightEdge(e)) {
      updateToggle(true);
    } else {
      updateToggle(false);
    }
  }

  // ...
}
```

### Passing data to the parent
This `Child` component fetches some data and then passes it to the `Parent` component in an Effect:
```javascript
function Parent() {
  const [data, setData] = useState(null);
  // ...
  return <Child onFetched={setData} />;
}

function Child({ onFetched }) {
  const data = useSomeAPI();
  // 🔴 Avoid: Passing data to the parent in an Effect
  useEffect(() => {
    if (data) {
      onFetched(data);
    }
  }, [onFetched, data]);
  // ...
}
```
In React, data flows from the parent components to their children. When you see something wrong on the screen, you can trace where the information comes from by going up the component chain until you find which component passes the wrong prop or has the wrong state. When child components update the state of their parent components in Effects, the data flow becomes very difficult to trace. Since both the child and the parent need the same data, let the parent component fetch that data, and pass it down to the child instead:
```javascript
function Parent() {
  const data = useSomeAPI();
  // ...
  // ✅ Good: Passing data down to the child
  return <Child data={data} />;
}

function Child({ data }) {
  // ...
}
```

### Subscribing to an external store
Sometimes your components may need to subscribe to some data outside of the React state. This if often done with an Effect, for example:
```javascript
function useOnlineStatus() {
  // Not ideal: Manual store subscription in an Effect
  const [isOnline, setIsOnline] = useState(true);
  useEffect(() => {
    function updateState() {
      setIsOnline(navigator.onLine);
    }

    updateState();

    window.addEventListener('online', updateState);
    window.addEventListener('offline', updateState);
    return () => {
      window.removeEventListener('online', updateState);
      window.removeEventListener('offline', updateState);
    };
  }, []);
  return isOnline;
}

function ChatIndicator() {
  const isOnline = useOnlineStatus();
  // ...
}
```

Although it’s common to use Effects for this, React has a purpose-built Hook for subscribing to an external store that is preferred instead. Delete the Effect and replace it with a call to `useSyncExternalStore`:

```javascript
function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus() {
  // ✅ Good: Subscribing to an external store with a built-in Hook
  return useSyncExternalStore(
    subscribe, // React won't resubscribe for as long as you pass the same function
    () => navigator.onLine, // How to get the value on the client
    () => true // How to get the value on the server
  );
}

function ChatIndicator() {
  const isOnline = useOnlineStatus();
  // ...
}
```

### Fetching data

Many apps use Effects to kick off data fetching. It is quite common to write a data fetching Effect like this:
```javascript
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  const [page, setPage] = useState(1);

  useEffect(() => {
    // 🔴 Avoid: Fetching without cleanup logic
    fetchResults(query, page).then(json => {
      setResults(json);
    });
  }, [query, page]);

  function handleNextPageClick() {
    setPage(page + 1);
  }
  // ...
}
```

You don’t need to move this fetch to an event handler. This might seem like a contradiction with the earlier examples where you needed to put the logic into the event handlers! However, consider that it’s not the typing event that’s the main reason to fetch. Search inputs are often prepopulated from the URL, and the user might navigate Back and Forward without touching the input.

However, the code above has a bug. Imagine you type "hello" fast. Then the query will change from "h", to "he", "hel", "hell", and "hello". This will kick off separate fetches, but there is no guarantee about which order the responses will arrive in. For example, the "hell" response may arrive after the "hello" response. Since it will call `setResults()` last, you will be displaying the wrong search results. This is an example of a race condition. 

To fix this race condition, you need to add a cleanup function to ignore stale responses:
```javascript
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  const [page, setPage] = useState(1);
  useEffect(() => {
    let ignore = false;
    fetchResults(query, page).then(json => {
      if (!ignore) {
        setResults(json);
      }
    });
    return () => {
      ignore = true;
    };
  }, [query, page]);

  function handleNextPageClick() {
    setPage(page + 1);
  }
  // ...
}
```

This ensures that when your Effect fetches data, all responses except the last requested one will be ignored.

## Lifecycle of Effects

Every React component goes through the same lifecycle:

- A component mounts when it's added to the screen
- A component updates when it receives new props or state
- A component unmounts when it's removed from the screen

Effects are independent from the component's lifecycle. Effects specify how to start synchronizing and how to stop synchronizing. Sometimes it is necessary to start and stop synchronizing multiple times while a component is still mounted. Thus is it not good to thing that effects start syncing when a component mounts and stop syncing when a component unmounts. 

An Effect will start syncing when either:
- A component mounts for the first time
- On a re-render when the Effect's dependencies have changed

An Effect will stop syncing when either:
- An Effect's dependencies have changed on a re-render
- When a component unmounts

When an a component mounts, the Effect will start synchronzing with the values of its dependencies for that render (start synchronizing). When a component re-renders, if the dependencies of the effect have changed, React will call the cleanup function for the values of the dependencies from the previous render (stop synchronizing). Then React will run the Effect for the new value of the dependencies (start synchronzing). When a component unmounts, the cleanup for the last render is called (stop synchronizng).

With Effects, it's best to always focus on a single start/stop cycle at a time. It shouldn’t matter whether a component is mounting, updating, or unmounting. All you need to do is to describe how to start synchronization and how to stop it. If you do it well, your Effect will be resilient to being started and stopped as many times as it’s needed.

### How React verifies that your Effect can re-synchronize
React verifies that your Effect can re-synchronize by forcing it to do that immediately in development. React starts and stops your Effect one extra time in development to check that there is a cleanup function implemented.

### How React knows that it needs to re-synchronize the Effect
React knows when to re-synchronize the effect because of the list of dependencies specified in it. Every time after your component re-renders, React will look at the array of dependencies that you have passed. If any of the values in the array is different from the value at the same spot that you passed during the previous render, React will re-synchronize your Effect.

Resist adding unrelated logic to your Effect only because this logic needs to run at the same time as an Effect you already wrote. Each Effect in your code should represent a separate and independent synchronization process.

### Effects "react" to reactive values
Effect dependencies only need to be values that can change over time. For example, props, state, and other values declared inside the component are reactive because they’re calculated during rendering and participate in the React data flow. On the other hand, constant values that do not change do not have to be list as dependencies. 

An empty `[]` dependency array means the Effect synchronizes when the component mounts and stops synchronizing when the component unmounts. 

Props and state aren’t the only reactive values. Values that you calculate from them are also reactive. If the props or state change, your component will re-render, and the values calculated from them will also change. This is why all variables from the component body used by the Effect should be in the Effect dependency list. All values inside the component (including props, state, and variables in your component’s body) are reactive. Any reactive value can change on a re-render, so you need to include reactive values as Effect’s dependencies.

<detail>
<summary>React Deep Dive</summary>

Mutable values (including global variables) aren’t reactive.

A mutable value like `location.pathname` can’t be a dependency. It’s mutable, so it can change at any time completely outside of the React rendering data flow. Changing it wouldn’t trigger a re-render of your component. Therefore, even if you specified it in the dependencies, React wouldn’t know to re-synchronize the Effect when it changes. Instead, you should read and subscribe to an external mutable value with `useSyncExternalStore`.

A mutable value like ref.current or things you read from it also can’t be a dependency. The ref object returned by useRef itself can be a dependency, but its current property is intentionally mutable. It lets you keep track of something without triggering a re-render. But since changing it doesn’t trigger a re-render, it’s not a reactive value, and React won’t know to re-run your Effect when it changes.
</details>

### What if youi won't want to re-synchronize

If you don't want to re-synchronize, you must prove that the values aren't reactive. You could move the values outside the component, or move them inside the effect. You can’t “choose” your dependencies. Your dependencies must include every reactive value you read in the Effect. Here is some ways to fix errors with Effects:

- Check that your Effect represents an independent synchronization process. If your Effect doesn’t synchronize anything, it might be unnecessary. If it synchronizes several independent things, split it up.
- If you want to read the latest value of props or state without “reacting” to it and re-synchronizing the Effect, you can split your Effect into a reactive part (which you’ll keep in the Effect) and a non-reactive part (which you’ll extract into something called an Effect Event).
- Avoid relying on objects and functions as dependencies. If you create objects and functions during rendering and then read them from an Effect, they will be different on every render. This will cause your Effect to re-synchronize every time.

