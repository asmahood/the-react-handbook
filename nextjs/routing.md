# Routing

## Creating Routes

Next.js uses a file-system based router where folders are used to define routes. Each folder represents a route segment that maps to a URL segment. To create a nested route, you nest folders inside each other.

```
/project
    /app => maps to /
        /dashboard => maps to /dashboard
            /settings => maps to /dashboard/settings
```

To make a route accessible, you need to include a `page.js` file at that route. Any directory that does not include a `page.js` file will not be publicly accessible.

## Pages
A page is a UI that is unique to a route. You can define a page by default exporting a component from a `page.js` file:

```javascript
export default function Page() {
  return <h1>Hello, Next.js!</h1>
}
```

Pages are server components by default, but can be set to a client component. Pages can also fetch data.

## Layouts and Templates
There are two more special files `layout.js` and `template.js` that allow you to create UI that is shared between routes.

### Layouts
A layout is UI that is shared between multiple routes. On navigation, layouts preserve state, remain interactive, and do not re-render. Layouts can also be nested.

You can define a layout be default exporting a React component from a `layout.js` file. The component should accept a `children` prop that will be populated with a child layout or a page during rendering. For example:

```javascript
export default function DashboardLayout({
  children, // will be a page or nested layout
}: {
  children: React.ReactNode
}) {
  return (
    <section>
      {/* Include shared UI here e.g. a header or sidebar */}
      <nav></nav>
 
      {children}
    </section>
  )
}
```

#### Root Layout (Required)

The root layout is defiend at the top level of the `app` directory and applies to all routes. This layout is requried and must contain `html` and `body` tags, allowing you to modify the initial HTML returned from the server.

```javascript
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        {/* Layout UI */}
        <main>{children}</main>
      </body>
    </html>
  )
}
```

#### Nested Layouts

Be default, layouts in the folder hierarchy are nested, which means they wrap child layouts via their `children` prop.

<details>
<summary>Next.js Deep Dive</summary>

- Layouts are server components by default but can be set to a client component.
- Layouts can fetch data
- Passing data between a parent layout and its children is not possible. However you can fetch the same data in a route more than once, and React will automatically dedupe the requests.
- Layouts do not have access to the `pathname`. Imported client components can access the pathname using the `usePathname` hook.
- Layouts do not have access to the route segments below itself. To access all route segments, you can use the `useSelectedLayoutSegment` or `useSelectedLayoutSegments` in a client component.
- You can use route groups to opt specific route segments in and out of shared layouts.
- You can use route groups to create multiple root layouts
</details>

### Templates
Templates are similar to layouts in that they wrap a child layout or page, however they create a new instance for each of their children on navigation. This means that when a user navigates between routes that share a template, a new instance of the child is mounted, DOM elements are recreated, state is not preserved in client components, and effects are re-synchronized.

Some examples of when this behaviour is desired:
- To re-sync `useEffect` on navigation
- To reset the state of a child client component on navigation.

A template can be defined by default exporting aa React component from `template.js` file. The component should accept a children pop.

```javascript
export default function Template({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>
}
```

In terms of nesting, a template is rendered between a layout and its children.

### Examples

#### Metadata

You can modify the `<head>` HTML elements such as `title` and `meta` using the Metadata APIs. Metadata can be defined by exporting a `metadata` object or a `generateMetadata` function in a layout or page:

```javascript
import type { Metadata } from 'next'
 
export const metadata: Metadata = {
  title: 'Next.js',
}
 
export default function Page() {
  return '...'
}
```

#### Active Nav Links

You can use the `usePathname` hook to determine if a nav link is active. Since `usePathname` is a client hook, you need to extract the nav links into a client component, whihc can be imported into your layout or template:

```javascript
'use client'
 
import { usePathname } from 'next/navigation'
import Link from 'next/link'
 
export function NavLinks() {
  const pathname = usePathname()
 
  return (
    <nav>
      <Link className={`link ${pathname === '/' ? 'active' : ''}`} href="/">
        Home
      </Link>
 
      <Link
        className={`link ${pathname === '/about' ? 'active' : ''}`}
        href="/about"
      >
        About
      </Link>
    </nav>
  )
}
```

## Linking and Navigating

There are four ways to navigate between routes:

- Using the `<Link>` component
- Using the `useRouter` client component hook
- Using the `redirect` function in server components
- Using the native History API

### `<Link>` Component

The `<Link>` component is a built-in component that extends the HTML `<a>` tag to provide prefetching and client-side navigation between routes. It is the primary and reocmmended way to navigate between routes:

```javascript
import Link from 'next/link'
 
export default function Page() {
  return <Link href="/dashboard">Dashboard</Link>
}
```

### `useRouter()` Hook

The `useRouter` hook allows you to change routes from client components:

```javascript
'use client'
 
import { useRouter } from 'next/navigation'
 
export default function Page() {
  const router = useRouter()
 
  return (
    <button type="button" onClick={() => router.push('/dashboard')}>
      Dashboard
    </button>
  )
}
```

### `redirect` Function

For server components, use the `redirect` function:

```typescript
import { redirect } from 'next/navigation'
 
async function fetchTeam(id: string) {
  const res = await fetch('https://...')
  if (!res.ok) return undefined
  return res.json()
}
 
export default async function Profile({ params }: { params: { id: string } }) {
  const team = await fetchTeam(params.id)
  if (!team) {
    redirect('/login')
  }
 
  // ...
}
```

*Note: `redirect` can be called in client components during the rendering process but not in event handlers. Use the `useRouter` hook instead.*

## Error Handling

Errors can be divided into two categories: expected errors and uncaught exceptions:

- Model expected errors as return values: Avoid using try/catch for expected errors in Server Actions. Use useFormState to manage these errors and return them to the client.
- Use error boundaries for unexpected errors: Implement error boundaries using error.tsx and global-error.tsx files to handle unexpected errors and provide a fallback UI.

### Handling Expected Errors
Expected errors are those that can occur during the normal operation of the application, such as those from server-side form validation or failed requests. These errors should be handled explicitly and returned to the client.

#### Handling Expected Errors from Server Actions

Use the `useActionState` hook to manage the state of Server Actions, including handling errors. This approach avoids try/catch blocks for expected errors, which should be modeled as return values rather than thrown exceptions.

```typescript
'use server'
 
import { redirect } from 'next/navigation'
 
export async function createUser(prevState: any, formData: FormData) {
  const res = await fetch('https://...')
  const json = await res.json()
 
  if (!res.ok) {
    return { message: 'Please enter a valid email' }
  }
 
  redirect('/dashboard')
}
```
Then you can pass your action to the `useActionState` hook and use the returned `state` to display an error message:

```typescript
'use client'
 
import { useFormState } from 'react-dom'
import { createUser } from '@/app/actions'
 
const initialState = {
  message: '',
}
 
export function Signup() {
  const [state, formAction] = useFormState(createUser, initialState)
 
  return (
    <form action={formAction}>
      <label htmlFor="email">Email</label>
      <input type="text" id="email" name="email" required />
      {/* ... */}
      <p aria-live="polite">{state?.message}</p>
      <button>Sign up</button>
    </form>
  )
} 
```

#### Handling Expected Errors from Server Components

When fetching data inside of a server component, you can use the response to conditionally render an error message or redirect:

```typescript
export default async function Page() {
  const res = await fetch(`https://...`)
  const data = await res.json()
 
  if (!res.ok) {
    return 'There was an error.'
  }
 
  return '...'
}
```

### Uncaught Exceptions
Uncaught exceptions should be handled by throwing errors which will be caught by error boundaries:

- **Common**: Handle uncaught errors below the root layout with `error.js`
- **Optional**: Handle granular uncaught errors with nested `errors.js` files (e.g. `dashboard/error.js`)
- **Uncommon**: Handle uncaught errors in the root layout with `global-error.js`.

#### Using Error Boundaries

Next.js uses error boundaries to handle uncaught exceptions. Error boundaries catch errors in their child components and display a fallback UI instead of the component tree that crashed.

Create an error boundary by adding an error.tsx file inside a route segment and exporting a React component:

```typescript
'use client' // Error boundaries must be Client Components
 
import { useEffect } from 'react'
 
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Log the error to an error reporting service
    console.error(error)
  }, [error])
 
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button
        onClick={
          // Attempt to recover by trying to re-render the segment
          () => reset()
        }
      >
        Try again
      </button>
    </div>
  )
}
```

If you want errors to bubble up to the parent error boundary, you can `throw` when rendering the `error` component.

#### Handling Errors in Nested Routes

Errors will bubble up to the nearest parent error boundary. THis allows for granular error handling by placing `error.js` files at different levels in the route hierarchy.

#### Handling Global Errors
While less common, you can handle errors in the root layout using `app/global-error.js` located in the root app directory, even when leveraging internationalization. Global error UI must define its own `<html>` and `<body>` tags, since it is replacing the root layout or template when active.
