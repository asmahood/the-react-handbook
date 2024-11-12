# Event Handling

To respond to user interactions like clicking, hovering, and focusing form inputs React lets you add event handlers to the JSX.

## Adding event handlers
To add an event handler, define a function and pass that function to an appropriate JSX tag. For example, to handle the event when a user clicks on a button:
```javascript
export default function Button() {
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

Event handler functions are usually defined inside components. By convention it is common to name event handlers as `handle` followed by the event name. Alternatively you can define handler functions inline:
```javascript
<button onClick={() => {
  alert('You clicked me!');
}}>
```

## Passing event handlers as props

Often when moving state up the tree, you'll be required to pass event handlers as props. This can be done as any other prop.

### Naming event handler props

Built-in HTML elements like `<button>` and `<div>` only support browser event names like `onClick`. When building your own components, you can name their event handler props any way that you like. By convention event handler props should start with `on` followed by a captial letter and the name of the event. For example, a prop could be named `onSmash`.

*Note: make sure to use the appropriate HTML tags for event handlers. For example: use `<button onClick>` instead of `<div onClick>`. Using a browser `<button>` enalbes built-in browser behaviours like keyboard navigation.*

## Event propagation

Event handlers will also catch events from any children your component might have. This is called **propagation**. The event starts where the event happened and "propagates" up the tree. For example:

```javascript
export default function Toolbar() {
  return (
    <div className="Toolbar" onClick={() => {
      alert('You clicked on the toolbar!');
    }}>
      <button onClick={() => alert('Playing!')}>
        Play Movie
      </button>
      <button onClick={() => alert('Uploading!')}>
        Upload Image
      </button>
    </div>
  );
}
```

When clicking either button, first the `Playing!` or `Uploading!` alerts will appear, followed by the `You clicked on the toolbar!` alert. If you only click the toolbar, just that alert will appear.

*Note: all events propagate in React except `onScroll`, which only works on the JSX tag you attach to it.*

### Stopping propagation

Event handlers receive an **event object** as their only argument. By convention it's usually called `e`. This object can be used to read information about event. If you want to prevent an event from reaching parent components, you need to call `e.stopPropagation()`:

```javascript
function Button({ onClick, children }) {
  return (
    <button onClick={e => {
      e.stopPropagation();
      onClick();
    }}>
      {children}
    </button>
  );
}

export default function Toolbar() {
  return (
    <div className="Toolbar" onClick={() => {
      alert('You clicked on the toolbar!');
    }}>
      <Button onClick={() => alert('Playing!')}>
        Play Movie
      </Button>
      <Button onClick={() => alert('Uploading!')}>
        Upload Image
      </Button>
    </div>
  );
}
```

#### Capture Phase Events

In rare cases, you might need to catch all events on child elements, even if they stopped propagation. For example, logging clicks for analytics. To do this, add `Capture` to the end of the event name:

```javascript
<div onClickCapture={() => { /* this runs first */ }}>
  <button onClick={e => e.stopPropagation()} />
  <button onClick={e => e.stopPropagation()} />
</div>
```

Each event propagates in three phases:

1. It travels down, calling all `onClickCapture` handlers
2. It runs the clicked element's `onClick` handler
3. It travels upwards, calling all `onClick` handlers

## Preventing Default Behaviour

Some browser events have default behaviour associated with them. For example, a `<form>` submit event, which happens when a button inside of it is clicked, will reload the whole page by default. You can call `e.preventDefault()` to stop this behaviour.

```javascript
export default function Signup() {
  return (
    <form onSubmit={e => {
      e.preventDefault();
      alert('Submitting!');
    }}>
      <input />
      <button>Send</button>
    </form>
  );
}
```
