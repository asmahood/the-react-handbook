# Managing State
This section will cover techniques on how to structure state well, how to keep state update logic maintainable, and how to share state between distant components.

## Reacting to Input with State
React provides a declarative way to manipulate the UI. Instead of manipulating individual pieces of the UI directly, you describe the different states that the component can be in, and switch between them in response to user input.

In React, you don't directly manipulate the UI. Instead you declare what you want to show, and React figures out how to update the UI.

### Thikning about UI Declaratively
To start thinking in React, use these steps when creating/refactoring a component:

1. **Identify** your components different visual states
2. **Determine** what triggers those state changes
3. **Represent** the state in memory using `useState`
4. **Remove** any non-essential state variables
5. **Connect** the event handlers to set the state

#### Step 1: Identify your component's different visual states

First visualize all the different "states" of the UI the user might see. Just like a designer, you'll want to mock up or create mocks for the different states before you add logic. To control the mock you can add a prop to select between the different states. Then, using this prop, you can place all states of the UI beside each other during implementation.

#### Step 2: Determine what triggers those state changes
State updates can be triggered in response to two kinds of inputs:

- Human inputs: like clicking a button, typing in a field, navigating a link.
- Computer inputs: like a network response arriving, a timeout completing, an image loading.

In both cases, you need state variables to update the UI.


#### Step 3: Represent the state in memory with `useState`
Next, you'll need to represent the visual sates of your component in memory with `useState`. Start with the state that absolutely must be there. You'll also want a state variable representing which one of the visual states that you want to display. There is usually more than one way to do this, so experimentation is needed.

#### Step 4: Remove any non-essential state variables

You want to avoid duplication in the state content so you're only tracking what is essential. The goal is to prevent the cases where the state in meory doesn't represent any valid IO that you'd want a user to see. Some questions to ask:

- Does this state cause a paradox? For example, `isTyping` and `isSubmitting` can't both be true at the same time. To remove the "impossible" state, you can combine these into a `status` state that must be one of a few different values
- Is the same information available in another state variable already? Another paradox: `isEmpty` and `isTyping` can't be `true` at the same time. You can remove `isEmpty` by checking if `answer.length === 0`
- Can you get the same information from the inverse of another state variable? `isError` is not beeded because you can check `error !== null` instead.

You will know state variables are essential when you can't remove any of them without breaking functionality.

#### Step 5: Connect the event handlers to set state
Lastly, create event handlers that update the state. Below is the final form, with all event handlers wired up.

## Choosing the State Structure
Below is some tips when structuring state.

### Principles for structuring state

1. **Group related state**. If you always update two or more state variables at the same time, consider merging them into a single state variable.
2. **Avoid contradictions in state**. When the state is structured in a way that several pieces of state may contradict and "disagree" with each other, you leave room for mistakes. Avoid this.
3. **Avoid redundant state**. If you can calculate some information from the components props or it's existing state variables during rendering, you should not put that information into that component's state.
4. **Avoid duplication in state**. When the same data is duplicated between multiple state variables, or within nested objects, it is difficult to keep them in sync. Reduce duplication when possible.
5. **Avoid deeply nested state**. Deeply hierarchical state is not very convenient to update. When possible, prefer to structure state in a flat way.

## Sharing State Between Components
When you want the state of two components to update together, remove the state from both of them and move it to their closest common parent, and then pass it down via props. This is known as **lifting state up**.

### An Example
In this example, a parent `Accordion` component renders two separate `Panel`s:

- `Accordion`
  - `Panel`
  - `Panel`

Each `Panel` component has a boolean `isActive` state that determines whether its content is visible.

```javascript
import { useState } from "react";

function Panel({ title, children }) {
    const [isActive, setIsActive] = useState(false);
    return (
        <section>
            <h3>{title}</h3>
            {isActive ? (
                <p>{children}</p>
            ) : (
                <button onClick={() => setIsActive(true)}>
                    Show
                </button>
            )}
        </section>
    );
}

export default function Accordion() {
    return (
        <>
            <h2>Almaty, Kazahkstan</h2>
            <Panel title="About">Lorem Ipsum</Panel>
            <Panel titel="Etymology">Lorem Ipsum</Panel>
        </>
    );
}
```

Right now, the the panel's buttons are independent, they do not affect each other.

Now let's say we want to change it so that only one panel is expanded at any given time. To coordinate these two panels, you need to "lift their state up" to a parent component in three steps:

1. Remove state from the child components
2. Pass hardocded data from the common parent
3. Add state to the common parent and pass it down together with the event handlers

#### Step 1: Remove state from the child components
You will give control of the `Panel`'s `isActive` to its parent component. This means that the parent component will pass `isActive` to `Panel` as a prop instead. So remove:
```javascript
const [isActive, setIsActive] = useState(false);
```
And instead add `isActive` to the `Panel`'s list of props
```javascript
function Panel({ title, children, isActive }) {
```

#### Step 2: Pass hardcoded data from the common parent
In this example, `Accordion` is the common parent of both `Panel`s. Make the `Accordion` component pass a hardcoded value of `isActive` (for example `true`) to both panels.
```javascript
<Panel title="About" isActive={true}>Lorem Ipsum</Panel>
```

#### Step 3: Add state to the common parent
Now, the `Accordion` needs to keep track of which panel is the active one. Instead of a `boolean` value, it could use a number as the index of the active panel for the state variable:
```javascript
const [activeIndex, setActiveIndex] = useState(0);
```

When the `activeIndex` is `0`, the first panel is active and when the it's `1`, its the second one.

Clicking the "Show" button on either panel needs to change the active index in `Accordion`. A panel can't set the `activeIndex` state directly because its defined inside `Accordion`. The `Accordion` component needs to explicitly allow the `Panel` component to change its state by passing an event handler down as a prop:
```javascript
<Panel isActive={activeIndex === 0} onShow(() => setActiveIndex(0))>...</Panel>
```
The button inside `Panel` will now use the `onShow` prop as its click event handler:
```javascript
function Panel({ title, children isActive, onShow }) {
    return (
        <section>
            <h3>{title}</h3>
            {isActive ? (
                <p>{children}</p>
            ) : (
                <button onClick={onShow}>
                    Show
                </button>
            )}
        </section>
    );
}
```

## Preserving and Resetting State

State is tied to a position in the render tree. React builds render trees for the component structure in your UI. When you give a component state, it does not "live" inside the component, but in React. React associates each piece of state it's holding with the correct component by where that component sits in the render tree.

React will keep the state around for as long as you render the same component at the same position in the tree. If a component gets removed or a different component gets rendered in the same position, React discards its state.

If React renders the same component at the same position, state is preserved. Note that this is the position in the UI tree, not the JSX markup.

Different components at the same position reset state. If you switch between different component types at the same position, React will destroy the state of the original component. Additionally, when you render a different component in the same position, it resets the staet of its entire subtree.

As a rule of thumb, if you want to preserve the state between re-renders, the structure of your tree needs to "match up" from one render to another.

When you want to reset state at the same position, there are two ways to do so:

1. Render components in different positions
2. Give each component an explicit identity with `key`

### Option 1: rendering a component in different positions

Suppose you wanted a one counter to display the points for two players in a game, and a button to switch between each players counter. To achieve this, we can render the component in different positions:

```javascript
import { useState } from 'react';

export default function Scoreboard() {
  const [isPlayerA, setIsPlayerA] = useState(true);
  return (
    <div>
      {isPlayerA &&
        <Counter person="Taylor" />
      }
      {!isPlayerA &&
        <Counter person="Sarah" />
      }
      <button onClick={() => {
        setIsPlayerA(!isPlayerA);
      }}>
        Next player!
      </button>
    </div>
  );
}

function Counter({ person }) {
  const [score, setScore] = useState(0);
  const [hover, setHover] = useState(false);

  let className = 'counter';
  if (hover) {
    className += ' hover';
  }

  return (
    <div
      className={className}
      onPointerEnter={() => setHover(true)}
      onPointerLeave={() => setHover(false)}
    >
      <h1>{person}'s score: {score}</h1>
      <button onClick={() => setScore(score + 1)}>
        Add one
      </button>
    </div>
  );
}
```
Initially, `isPlayerA` is `true`, so the first position contains a `Counter` state, and the second one is empty. When you click "Next player", the first position clears but the second one now contains a `Counter`. Each `Counter`'s state gets destroyed each time it's removed from the DOM. This is why they reset every time the button is clicked.

### Option 2: Resetting state with a key

A more generic way to reset a component's state by using a `key`. Keys are used to make React distinguish between any components. In the example above, we can use a `key` to tell React that this isn't just the first counter, but Taylors counter.

```javascript
import { useState } from 'react';

export default function Scoreboard() {
  const [isPlayerA, setIsPlayerA] = useState(true);
  return (
    <div>
      {isPlayerA ? (
        <Counter key="Taylor" person="Taylor" />
      ) : (
        <Counter key="Sarah" person="Sarah" />
      )}
      <button onClick={() => {
        setIsPlayerA(!isPlayerA);
      }}>
        Next player!
      </button>
    </div>
  );
}

function Counter({ person }) {
  const [score, setScore] = useState(0);
  const [hover, setHover] = useState(false);

  let className = 'counter';
  if (hover) {
    className += ' hover';
  }

  return (
    <div
      className={className}
      onPointerEnter={() => setHover(true)}
      onPointerLeave={() => setHover(false)}
    >
      <h1>{person}'s score: {score}</h1>
      <button onClick={() => setScore(score + 1)}>
        Add one
      </button>
    </div>
  );
}
```

In this example, the two `Counter`s don't share state even though they appear in the same place in JSX. Switching between Taylor and Sarah does not preserve the state. This is because they have different `key`s.

### Resetting a form with a key
Resetting state with a key is particularly useful when dealing with forms. Add a key to a component with a form, so that if state above it changes, the form will be rendered because it has a different key. For example, for a chat app:
```javascript
import { useState } from 'react';

export default function Messenger() {
  const [to, setTo] = useState(contacts[0]);
  return (
    <div>
      <ContactList
        contacts={contacts}
        selectedContact={to}
        onSelect={contact => setTo(contact)}
      />
      <Chat key={to.id} contact={to} />
    </div>
  )
}

function ContactList({
  selectedContact,
  contacts,
  onSelect
}) {
  return (
    <section className="contact-list">
      <ul>
        {contacts.map(contact =>
          <li key={contact.id}>
            <button onClick={() => {
              onSelect(contact);
            }}>
              {contact.name}
            </button>
          </li>
        )}
      </ul>
    </section>
  );
}

function Chat({ contact }) {
  const [text, setText] = useState('');
  return (
    <section className="chat">
      <textarea
        value={text}
        placeholder={'Chat to ' + contact.name}
        onChange={e => setText(e.target.value)}
      />
      <br />
      <button>Send to {contact.email}</button>
    </section>
  );
}
```

In this example, if you click on a new recipient, the `Chat` component will be recreated from scratch including any state in the tree below it.

<details>
<summary><h4>React Deep Dive</h4></summary>

If you want to preserve state for removed components, there are a few options:

- You could render all components instead of the current one, but hide others with CSS.
  - Components would not get removed, so local state is preserved.
  - Can be very slow if the hidden trees are large and contain a lot of DOM nodes.
- You could lift state up to a parent component. Thus when child components get removed the parent still holds the state information
  - This is the most common solution
- You could also use a different source, such as reading from `localStorage`.
</details>

## Extracting State Logic into a Reducer

Components with many state updates spread across many event handlers can get overwhelming. For these cases, you can consolidate all the state update logic outside your component in a single function, called a **reducer**.

For example, consider the `TaskApp` component below which holds an array of `tasks` in state and uses three different event handlers to add, remove, and edit tasks:

```javascript
import { useState } from 'react';
import AddTask from './AddTask.js';
import TaskList from './TaskList.js';

export default function TaskApp() {
  const [tasks, setTasks] = useState(initialTasks);

  function handleAddTask(text) {
    setTasks([
      ...tasks,
      {
        id: nextId++,
        text: text,
        done: false,
      },
    ]);
  }

  function handleChangeTask(task) {
    setTasks(
      tasks.map((t) => {
        if (t.id === task.id) {
          return task;
        } else {
          return t;
        }
      })
    );
  }

  function handleDeleteTask(taskId) {
    setTasks(tasks.filter((t) => t.id !== taskId));
  }

  return (
    <>
      <h1>Prague itinerary</h1>
      <AddTask onAddTask={handleAddTask} />
      <TaskList
        tasks={tasks}
        onChangeTask={handleChangeTask}
        onDeleteTask={handleDeleteTask}
      />
    </>
  );
}

let nextId = 3;
const initialTasks = [
  {id: 0, text: 'Visit Kafka Museum', done: true},
  {id: 1, text: 'Watch a puppet show', done: false},
  {id: 2, text: 'Lennon Wall pic', done: false},
];
```

To reduce complexity and to keep all logic in one place, the state logic can be moved into a reducer.

Reducers are a different way to handle state. You can migrate from `useState` to `useReducer` in three steps:

1. Move from setting state to dispatching actions
2. Write a reducer function
3. Use the reducer from the component

### Step 1: Move from setting state to dispatching actions

Managing state with reducers is slightly different from directly setting state. Instead of telling React "what to do" by setting state, you specify "what the user just did" by dispatching "actions" from your event handlers. The state update logic will live elsewhere. So instead of setting tasks via an event handler, you will dispatch an "added/changed/deleted a task" action.

```javascript
function handleAddTask(text) {
  dispatch({
    type: 'added',
    id: nextId++,
    text: text,
  });
}

function handleChangeTask(task) {
  dispatch({
    type: 'changed',
    task: task,
  });
}

function handleDeleteTask(taskId) {
  dispatch({
    type: 'deleted',
    id: taskId,
  });
}
```

The object you pass to `dispatch` is called an "action". Actions should generally contain the minimal information about what happened. 

### Step 2: Write a reducer function

A reducer function is where all of the state logic will go. It takes two arguments, the current state and the action object, and it returns the next state:

```javascript
function yourReducer(state, action) {
  // return next state for React to set
}
```

React will set the state to what is returned from the reducer.

To move the state setting logic from event handlers to a reducer function in the example:

- Declare the current state `tasks` as the first argument
- Declare the `action` object as the second argument
- Return the next state from the Reducer

```javascript
function tasksReducer(tasks, action) {
  if (action.type === 'added') {
    return [
      ...tasks,
      {
        id: action.id,
        text: action.text,
        done: false,
      },
    ];
  } else if (action.type === 'changed') {
    return tasks.map((t) => {
      if (t.id === action.task.id) {
        return action.task;
      } else {
        return t;
      }
    });
  } else if (action.type === 'deleted') {
    return tasks.filter((t) => t.id !== action.id);
  } else {
    throw Error('Unknown action: ' + action.type);
  }
}
```

*Note: it is convention to use `switch` statements in reducers.*

### Step 3: Use the reducer in the component

Finally, you need to hook up the `tasksReducer` to the component. Import the `useReducer` hook from React:
```javascript
import { useReducer } from "react";
```
Then replace `useState` with:
```
const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);
```

The `useReducer` hook is similar to `useState` in the fact that it takes an initial state and it returns a stateful value and a way to set state.

### Writing reducers well
Keep these in mind when writing reducers:

- Reducers must be pure. Similar to state updater functions, reducers run during rendering. They should not send requests, schedule timeouts, or perform any side effects.
- Each action describes a single user interaction, even if that leads to multiple changes in the data.

## Passing Data Deeply with Context
Passing props can become verbose and inconvenient when you need to pass some prop deeply through the tree or if many components need the same prop. React contexts allow you to "teleport" data to compoents in the tree without passing props. Contexts let a parent component provide data to the entire tree below it.

### Step 1: Creating a context
The first step is toc reate a context and export it from a file so that other components can use it:

```javascript
export const LevelContext = createContext(1);
```
The only argument to create context is the default value. 

### Step 2: Using the context
Import the `useContext` hook from React and your context
```javascript
import { useContext } from "react";
import { LevelContext } from "./LevelContext.js";
```
Then use the context with the `useContext` hook:
```javascript
export default function Heading({ children }) {
    const level = useContext(LevelContext);
}
```
Without a provider, React will use the default value. So in this case `level` will be 1.

### Step 3: Provide the context
To provide a context, wrap the component that you want to provide the context with a **context provider**:
```javascript
import { LevelContext } from "./LevelContext.js";

export default function Section({ level, children }) {
    return (
        <section className="section">
            <LevelContext.Provider value={level}>
                {children}
            </LevelContext.Provider>
        </section>
    )
}
```
The value passed to `value` will be passed to all child components that use the context. Components will use the value of the nearest matching provider.

### Before using context
Do not use a context just because you need to pass some props several levels deep. A few alternatives to use before using context:

1. **Start by passing props**. If the component is not trivial, it's not unusual to pass a dozen props down throigh a dozen components. This makes the data flow explicit.

2. **Extract components and pass JSX as `children` to them**. If you pass some data through many layers of intermediate componets that dpn't use that data (and only pass it further down), this often means that you forgot to extract some components along the way. For example, you may pass data props like `posts` to visual components that don't use them directly, like `<Layout posts={posts} />`. Instead make `Layout` take `children` as a prop and render `<Layout><Posts posts={posts} /><Layout>`

If neither of these approaches work well, then consider contexts.

### Some use cases for context
- **Themeing**: If your app lets the user change its appearance (e.g. dark mode) you can put a context provider at the top of the app and use that context in components to adjust their visual look.
- **Current Account**: Many components might need to know the currently logged in user. Putting it in context makes it convenient to read it anywhere in the tree.
- **Routing**: Most routing solutions use context internally to hold the current route. This is how every link "knows" whether it's active or not.
- **Managing state**: As the app grows, you might end up with a lot of state closer to the top of the app. Many distant components below may want to change it. It is common to use a reducer together with a context to manage complex state and pass it down to distant components.

## Using Reducers and Contexts Together
A reducer helps keep event handlers short and concise. However you might run into another difficulty. With only reducers, the state and dispatch functions are only available at the top level of the component they are used in. To let other components read the list of tasks or change it, you have to explicitly pass down the current state and the event handlers that change it as props.

As an alternative to this, you can put both the state and siaptch functions into context. This way, any components below the provider in the tree can read the state and dispatch actions without repetitive "prop drilling".

### Step 1: Create the context
The `useReducer` hook returns the current state and the `dispatch` function that updates the state. For example:
```javascript
const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);
```
To pass them down the tree, you will need to create two separate contexts:
- `TasksContext` provides the current list of tasks
- `TasksDispatchContext` provides the function that lets components dispatch actions

Export them from a separate file so that they can be imported later:
```javascript
import { createContext } from "react";

export const TasksContext = createContext(null);
export const TasksDispatchContext = createContext(null);
```

### Step 2: Put state and dispatch into context
Now import both contexts and provide the state and `dispatch` function to the contexts:
```javascript
import { TasksContext, TasksDispatchContext } from "./TasksContext.js";

export default function TaskApp() {
    const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

    return (
        <TasksContext.Provider value={tasks}>
            <TasksDispatchContext.Provider value={dispatch}>
                ...
            </TasksDispatchContext.Provider>
        </TasksContext.Provider>
    );
}
```

### Step 3: Use context anywhere in the tree
Instead of passing props, now any component that needs the task list can read it from the `TaskContext`:
```javascript
export default function TaskList() {
    const tasks = useContext(TasksContext);
    // ...
}
```
To update the task list, any component can read the `dispatch` function from context and call it:
```javascipt
export default function AddTask() {
    const [text, setText] = useState('');
    const dispatch = useContext(TasksDispatchContext);
    // ...
    return (
        // ...
        <button onClick={() => {
            setText('');
            dispatch({
                type: 'added',
                id: nextId++,
                text: text,
            });
        }}>Add</button>
        // ...
    );
}
```

### Moving everything into a single file
To further declutter components, you can move both the reducer and context into a single file. First move the reducer into the same file. Then declare a new provider component in the same file that will tie all the pieces together:

1. It will manage the state with a reducer
2. It will provide both contexts to components below
3. It will take `children` as a prop so you can pass JSX to it

```javascript
export function TasksProvider({ children }) {
    const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

    return (
        <TasksContext.Provider value={tasks}>
            <TasksDispatchContext.Provider value={dispatch}>
                {children}
            </TasksDispatchContext.Provider>
        </TasksContext.Provider>
    );
}
```
You can also export functions that use the context from `TasksContext.js`:
```javascript
export function useTasks() {
    return useContext(TasksContext);
}

export function useTasksDispatch() {
    return useContext(TasksDispatchContext);
}
```
