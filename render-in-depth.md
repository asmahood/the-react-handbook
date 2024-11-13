# Render in Depth
The process of requesting and serving UI has three steps:

1. Triggering a render
2. Rendering the component
3. Committing to the DOM

## Step 1: Trigger a render

There are two reasons for a component to render:

1. It's the component's initial render
2. The component (or one of its ancestors') state has been updated

### Initial Render

The initial render is started by calling the `createRoot` method with the target DOM node, then calling the `render` method with the component you want to render.

### Re-renders when state updates

Updating a components state automatically queues a render. This is done by calling a `setX` function for a state variable.

## Step 2: React renders the components
After a render is triggered, React calls the components to figure out what to display on screen.

- On the initial render, React will call the root component
- On subsequent renders, React will call the component whose state update triggered the render.

This process is recursive. This means that if the updated component returns some other component, React will render that component next. The process will continue until there are no more nested components and React knows exactly what should be displayed on screen.

<details>
<summary><h4>React Deep Dive</h4></summary>

The default behaviour of rendering all components nested within the updated component is not optimal for performance if the updated component is very high in the tree. There are several ways to optimize this if performance is a bottleneck. Do not optimize prematurely.
</details>

## Step 3: React commits changes to the DOM

After rendering all components, React will modify the DOM.

- For the initial render, React will sue the `appendChild` DOM API to put all DOM nodes it has created on the screen.
- For re-renders React will apply the minimal necessary operations (calculated while rendering) to make the DOM match the latest output.

**React only changes the DOM nodes if there's a difference between renders.**
