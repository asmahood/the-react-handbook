# Getting Started

## Components
React apps are made out of components. **Functional Components** are JavaScript functions that return JSX:
```javascript
function MyButton() {
  return (
    <button>I'm a button</button>
  );
}
```

React is designed around the concept that components must be **pure**. This means that React assumes that every component you write is a pure function. This means that React components must always return the same JSX given the same inputs. Components should only return their JSX and not change any objects or variables that existed before rendering. However, it is completely fine to change variables that are created while rendering. 

To use a component, nest it inside another component:
```javascript
export default function MyApp() {
  return (
    <div>
      <h1>Welcome to my app</h1>
      <MyButton />
    </div>
  );
}
```

*Note: React components must always start with capital letter. This distingushes them from HTML elements.*

*Note: It is convient to keep tightly related small components in the same file. However **do not nest definitions of components**. When a child component needs some data from a parent, pass it by props.*

## JSX Syntax
JSX is stricter than HTML. You must close tags like `<br />`. A component **cannot** return multiple JSX tags, they must be wrapped in a shared parent or in a React fragment:
```javascript
function AboutPage() {
  return (
    <>
      <h1>About</h1>
      <p>Hello there.<br />How do you do?</p>
    </>
  );
}
```
Additionally, in React many HTML and SVG attributes are written in camelCase. For historical reasons, `aria-*` and `data-*` attributes are written as in HTML with dashes.

### Styling
To add a CSS class, use the `className` attribute:
```javascript
<img className="avatar" />
```
To add inline CSS styles, use the `styles` attribute and pass an object representing the CSS rules:
```javascript
<img
    style={{
        width: user.imageSize,
        height: user.imageSize
    }}
/>
```

### Displaying Data
To embed some variable from code into the HTML to be displayed to the user, wrap it in `{}`:
```javascript
return (
  <h1>
    {user.name}
  </h1>
);
```
In attributes, use `{}` to set the value of an attribute to the value of an expression:
```javascript
return (
  <img
    className="avatar"
    src={user.imageUrl}
  />
);
```

### Conditional Rendering
You can use the same techniques from Vanilla JavaScript to conditionally include different JSX:
```javascript
let content;
if (isLoggedIn) {
  content = <AdminPanel />;
} else {
  content = <LoginForm />;
}
return (
  <div>
    {content}
  </div>
);

// OR

<div>
  {isLoggedIn ? (
    <AdminPanel />
  ) : (
    <LoginForm />
  )}
</div>

// If else branch not needed
<div>
  {isLoggedIn && <AdminPanel />}
</div>
```
This also works for specifying attributes.

### Rendering Lists
To render lists of items, use a `for` loop or the array `map` method:
```javascript
const products = [
  { title: 'Cabbage', id: 1 },
  { title: 'Garlic', id: 2 },
  { title: 'Apple', id: 3 },
];

const listItems = products.map(product =>
  <li key={product.id}>
    {product.title}
  </li>
);

return (
  <ul>{listItems}</ul>
);
```
Each item in the rendered list needs a unique `key` attribute among their siblings and the key cannot change. Usually this key should come from the data. React uses the keys to know what happened if you later insert, delete, or re-order items.

### Responding to Events
To respond to different events, define **event handler** functions inside a component and pass them as attributes to element that needs them:
```javascript
function MyButton() {
  function handleClick() {
    alert('You clicked me!');
  }

  return (
    <button onClick={handleClick}>
      Click me
    </button>
  );
}
```

### Updating the Screen
To "remember" information across re-renders, use the `useState` hook. For example:
```javascript
function MyButton() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <button onClick={handleClick}>
      Clicked {count} times
    </button>
  );
}
```
In this example, the `useState` hook is used. It returns two things, an object that holds the current state (`count`), and a function that updates the current state (`setCount`). These can be named anything, but common convention is to name them `[something, setSomething]`. When the page is first rendered, the count displayed will be 0, since we passed it into the hook as the initial value. When the button is clicked, the state will update, and the page will render with the updated state. Each component will gets its own state, so if you render a component multiple times, each instance will have a separate state. When you call a set function in a component, React automatically updates the child components inside too.

#### Aside on Hooks
Functions starting with `use` are called hooks. You can find other built-in hooks in the [React API reference](https://react.dev/reference/react). You can also write your own custom hooks by combining multiple hooks.

Hooks are more restrictive then functions. They can only be called at the top of components (or other hooks). If you want to use `useState` in a condition or loop, extract a new component and put it there.

#### Aside on Mutability
There is a benefit to keeping state immutable. By default, all child components re-render automatically when the state of a parent component changes. This includes even the child components that weren’t affected by the change. Although re-rendering is not by itself noticeable to the user (you shouldn’t actively try to avoid it!), you might want to skip re-rendering a part of the tree that clearly wasn’t affected by it for performance reasons. Immutability makes it very cheap for components to compare whether their data has changed or not. You can learn more about how React chooses when to re-render a component in the [memo API reference](https://react.dev/reference/react/memo).

### Sharing Data Between Components
To make two components share the same state, move the state "upwards" from the individual components to the nearest parent component that contains all of them. For example, if we wanted two buttons to share the same state and increment the count if either button is pressed:
```javascript
export default function MyApp() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <div>
      <h1>Counters that update separately</h1>
      <MyButton count={count} onClick={handleClick} />
      <MyButton count={count} onClick={handleClick} />
    </div>
  );
}
```
The information passed to a component is called **props**. Then update the `MyButton` component to accept and read these props:
```javascript
function MyButton({ count, onClick }) {
  return (
    <button onClick={onClick}>
      Clicked {count} times
    </button>
  );
}
```

When you want to nest React componets inside other React components, use the `children` prop. For example:

```javascript
import Avatar from './Avatar.js';

function Card({ children }) {
  return (
    <div className="card">
      {children}
    </div>
  );
}

export default function Profile() {
  return (
    <Card>
      <Avatar
        size={100}
        person={{ 
          name: 'Katsuko Saruhashi',
          imageId: 'YfeOqp2'
        }}
      />
    </Card>
  );
}
```

*To collect data from multiple children, or to have two child components communicate with each other, declare the shared state in their parent component instead. The parent component can pass that state back down to the children via props. This keeps the child components in sync with each other and with their parent.*

