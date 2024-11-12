# Thinking in React

When building a user interface with React, you will first break it down into components. Then, describe the different visual states for each component. Finally connect the components so data flows through them. Start with a mockup of data and the UI, and follow these 5 steps:

## Step 1: Break the UI into a Component Hierarchy
Start by drawing boxes around every component and subcomponent in the mockup and name them. The design can be split in many ways including:
- **Programming**: use the same techniques for deciding if you should create a new function or object, such as the *single responsibility principle*
- **CSS**: Consider what you would make class selectors for
- **Design**: Consider how you would organize the design's layers

Once the components are identified, arrange them into a hierarchy. Components that appear within another component should appear as a child in the hierarchy.

## Step 2: Build a static version
The easiest and most straightforward approach is to build a version that renders the UI from the data model with no interactivity. To build a static version, you'll want to build components that reuse other components and pass data using props. Do not use any state at this stage. You can either build **top down** (start with the parent components and work your way down), or **bottom up** (start with child components and work your way up). In simple examples, top-down is easier. In larger projects, bottom-up is easier.

## Step 3: Find the minimal complete representation of UI state
The next step is to make the UI interactive using state. Think of state as the minimal set of changing data that the app needs to remember. The most important principle for structuring state is to keep it **DRY (Don't Repeat Yourself)**. Figure out the minimal representation of the state the application needs and compute everything else on demand. 

Some questions to ask to determine if something needs to be kept in state:

- Does it **remain unchanged** over time?
  - If so, it isn't state.
- Is it **passed in from a parent** via props? 
  - If so it isn't state.
- Can you **compute it** based on existing state or props in the component?
  - If so, it *definitely* isn't state.

## Step 4: Identify where state should live
Next you need to identify which component is responsible for changing this state (or owns it). React uses one-way data flow, passing data down the component hierarchy from parent to child component. To figure out where state should live, use these steps:

1. Identify every component that renders something based on that state
2. Find their closest common parent component
3. Decide where state should live:
  1. Often, you can put state directly into the common parent
  2. You can also put the state into some component above their common parent
  3. If you can't find a component where it makes sense, create a new component solely for holding the state and add it somewhere in the hierarchy above the common parent component

## Step 5: Add inverse data flow
To change state according to user input, we need to support data flow in the opposite direction. To do this, you need to pass functions tthat update the state from where the state lives to the components that change that state. For example:
```javascript
function FilterableProductTable({ products }) {
  const [filterText, setFilterText] = useState('');
  const [inStockOnly, setInStockOnly] = useState(false);

  return (
    <div>
      <SearchBar 
        filterText={filterText} 
        inStockOnly={inStockOnly}
        onFilterTextChange={setFilterText}
        onInStockOnlyChange={setInStockOnly} />
```
