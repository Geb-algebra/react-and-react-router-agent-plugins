---
name: route-module-patterns
description: How to implement Route Modules for React Router. Use when you implement any page or component using React Router's features (loader, action, clientLoader, clientAction, fetcher, Form, etc.).
---

# React Router Route Module Patterns

This guide strictly defines how to and not to implement applications under React Router.

## Terms

This section defines terms we use in the remainder of this document.

### Route Module

A Route Module is a `.tsx` file which is rendered as a page by React Router.
A `.tsx` file is technically specified as a Route Module by adding it to `routes.ts`.

### Route Component

A Route Component is the default exported component from a Route Module.

## Principles

This section defines rules you MUST follow.

### Clear separation between data manipulation and UI

- Route Components must not manipulate data (no `localStorage`, external APIs, Factories, or Repositories).  
- All data loading/mutation must be in  `loader`, `action`, `clientLoader`, or `clientAction`.  
- Components may only:  
  - Load via `loaderData`, `clientLoaderData`, or `fetcher.load()`  
  - Mutate via `<Form>` or `fetcher.submit()`  
  - Get results via `actionData`, `clientActionData`, or `fetcher.data`  
- Following this rule minimizes `useEffect` usage.

### No self implementation around types of loaderData and actionData

- React Router auto-generates types for `loader`, `action`, `clientLoader`, and `clientAction` in each Route Module.  
- **Do not** define their return types manually—always use the generated ones.  
- Import them from `+types/{route module file name without .tsx}`.  
  - These are alias paths—**do not** try to inspect them directly (e.g., `cat +types/route.tsx`).  
- **Exception**: `<Form>` data types must be defined as a Zod schema in the Route Module and used for validation.

### Single Source of True Data

- Do not store the same data in multiple places (e.g., DB and Zustand).  
- Each piece of data must have a single source.  

- Temporary `useState` storage is allowed, but components must not rely on it.  
- States should be safe to lose without causing major issues.  

### Prefer `<Form>` or `<fetcher.Form>` over manual submission

- Use `<Form>` (with redirects) or `<fetcher.Form>` (without).  
- **Minimize** use `<div>` + `onSubmit`. If it is used, must be justified.
- Avoid `useState` for inputs—HTML handles storage and submission via `type="submit"`. If `useState` is used for inputs, it must be justified by needs other than submission.  

### Consistent error handling for form submissions

- `action` and `clientAction` must return an `error` string  
  - Empty if success  
  - Message if failed  

- Submitting components must display the error message.  
- Messages must be detailed enough for users to fix their submissions.  

## Route Module implementation patterns

This section introduces Route Module implementation patterns. You must select one of these to implement Route Module.

1. Page Pattern ... For pages
2. Resource Route Pattern ... For data loadings and/or data mutations required in other pages
3. Fullstack Component Pattern ... For components that can load and/or mutate data independently from other component

### Page Pattern

This pattern is for building a page.

In this pattern, the Route Module has a Route Component and a meta function for page title and description.
The Route Module also optionally has `loader` and/or `action` (for server data) and `clientLoader` and/or `clientAction` (for client data).

```tsx
// sample-page.tsx
import type { Route } from "./+types/sample-page";

export function meta({}: Route.MetaArgs) {
 return [
  { title: "Sample" },
  { name: "description", content: "sample" },
 ];
}

// omit if no server data
export async function loader({ request }: Route.LoaderArgs) {
    // load server data via domain API
}

// omit if no client data
export async function clientLoader({ request }: Route.ClientLoaderArgs) {
    // load client data via domain API
}

// omit if no server data updates
export async function action({ request }: Route.ActionArgs) {
    // parse and validate formData and update server data via domain API.
}

// omit if no client data updates
export async function clientAction({ request }: Route.ClientActionArgs) {
    // parse and validate formData and update client data via domain API.
}

export default function Page({ loaderData, actionData, clientLoaderData, clientActionData }: Route.ComponentProps) {
    // render pages using loaderData, actionData, clientLoaderData, clientActionData
    // Direct use of domain API are prohibited
}
```

### Resource Route Patterns

[Resource Routes - React Router Official Docs](https://reactrouter.com/how-to/resource-routes)

Resource Route is a Route Module without Route Component.

This pattern is useful when a page requires loading and mutating several kinds of data, especially across multiple domains.
By defining loading and mutation of each kind of data in separated Resource Route, you can keep these implementations simple.

This pattern also usable for serving non-html resources like txt, pdf or images, mainly for access from outside the app.

```tsx
// sample-resource-route.tsx
import type { Route } from "./+types/sample-resource-route";

// omit if no server data
export async function loader({ request }: Route.LoaderArgs) {
    // load server data via domain API
}

// omit if no client data
export async function clientLoader({ request }: Route.ClientLoaderArgs) {
    // load client data via domain API
}

// omit if no server data updates
export async function action({ request }: Route.ActionArgs) {
    // parse and validate formData and update server data via domain API.
}

// omit if no client data updates
export async function clientAction({ request }: Route.ClientActionArgs) {
    // parse and validate formData and update client data via domain API.
}
```

To work with data from resource routes:

```tsx
// sample-page.tsx
// ...

export default function Page({ loaderData, actionData, clientLoaderData, clientActionData }: Route.ComponentProps) {
    const fetcher = useFetcher()
    // to use resource route data with initial render:
    useEffect(() => {
        fetcher.load("/sample-resource-route")
        // then the data is stored in fetcher.data
    }, [fetcher.load])
    // to submit data to the resource route, submit data to "/sample-resource-route" action with fetcher.submit or <Form>
}
```

Keep in mind that you must specify a route to fetch ("/sample-resource-route" in above example) when you call fetcher.load or .submit.
Otherwise the fetcher fetches the route of the page it is defined ("/sample-page" in above example).

### Fullstack Component Patterns

[Full Stack Components - Epic Web Dev](https://www.epicweb.dev/full-stack-components)

Fullstack Component is a Route Module with normal (named export) components that use data from the Route Module.

This achieves good "separation of concerns" by encapsulating everything related to a resource within a single file.

```tsx
// sample-fs-component.tsx
import type { Route } from "./+types/sample-resource-route";

// omit if no server data
export async function loader({ request }: Route.LoaderArgs) {
    // load server data via domain API
}

// omit if no client data
export async function clientLoader({ request }: Route.ClientLoaderArgs) {
    // load client data via domain API
}

// omit if no server data updates
export async function action({ request }: Route.ActionArgs) {
    // parse and validate formData and update server data via domain API.
}

// omit if no client data updates
export async function clientAction({ request }: Route.ClientActionArgs) {
    // parse and validate formData and update client data via domain API.
}

export function Component(props) {
    const fetcher = useFetcher()
    // to use route data with initial render:
    useEffect(() => {
        fetcher.load("/sample-fs-component")
        // then the data is stored in fetcher.data
    }, [fetcher.load])
    // to submit data to the resource route, submit data to "/sample-fs-component" action with fetcher.submit or <Form>
}
```

Keep in mind that you must specify a route to fetch ("/sample-resource-route" in above example) when you call fetcher.load or .submit.
Otherwise the fetcher fetches the route of the page the component is used ("/sample-page" in above example).
