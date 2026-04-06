# React Application Development Rules

---

## 1. Project Structure and Organization

### 1.1 Directory Structure
- Place all custom components in a dedicated `src/components/` subdirectory.
- Place all page-level components in a separate `src/pages/` subdirectory.
- Place all React contexts in a `src/contexts/` subdirectory.
- Keep the API client class at the `src/` root level (e.g., `src/ApiClient.js`).

### 1.2 File Naming
- A component must be written in a source file of the same name (e.g., the `App` component lives in `App.js`, the `Header` component in `Header.js`).
- Test files use a `.test.js` suffix alongside the file they test (e.g., `Post.test.js` next to `Post.js`).

### 1.3 Cleaning Up Starter Projects
- Remove the cruft added by Create React App (e.g., `logo.svg`, `App.css`) before you begin coding your own application.
- Maintain all custom CSS styles in a single `index.css` file rather than per-component CSS files, to avoid redundancy.

---

## 2. Component Design

### 2.1 Use Function Components
- Write components as functions, not classes. Functional components use a more concise syntax and support hooks.
- Component names must begin with a capital letter and are written in CamelCase.
- Component functions must be exported as the default export: `export default function MyComponent() { ... }`.

### 2.2 Single Responsibility
- Do not add too much work to a single component. Instead, split work across several components, each having a single purpose.
- Always try to partition the application into many components, each with only one purpose.
- When a component is doing too much, refactor by moving logic down into new subcomponents.

### 2.3 Reusable Components via Props
- Use props to create components that are reusable and generic.
- Accept input arguments (props) via destructuring assignments in the function signature: `function Body({ sidebar, children })`.
- Use the `children` prop when a component needs to wrap arbitrary content provided by the parent.
- For boolean props, omitting the value defaults to `true` (e.g., `<Body sidebar>` is equivalent to `<Body sidebar={true}>`).
- When a child component needs to trigger an action the parent owns, pass a callback function as a prop.

### 2.4 Component Naming Conventions
- In JSX, native HTML elements use lowercase letters; React components use CamelCase. This is how you tell them apart.
- Give each component a `className` matching the component name (e.g., `className="Header"`) to facilitate CSS styling.

---

## 3. JSX Rules

### 3.1 Single Root Node
- A component must return a JSX tree with a single root node. Use fragment tags `<>...</>` to group multiple nodes without rendering an unnecessary DOM element.

### 3.2 Strict XML Syntax
- JSX requires strict XML syntax: all elements must be properly closed (e.g., `<br />` not `<br>`).

### 3.3 HTML Attribute Differences
- Use `className` instead of `class` to avoid conflict with the JavaScript `class` keyword.

### 3.4 Return Statement Parentheses
- Enclose JSX returned from a component in parentheses, and place the opening parenthesis on the same line as `return` to prevent automatic semicolon insertion issues.

### 3.5 Dynamic Expressions
- Insert JavaScript expressions in JSX by enclosing them in curly braces `{ }`.
- When a prop needs an object as a value, use double braces: `value={{ key: value }}` — the outer pair is the JSX expression delimiter, the inner pair is the object literal.

---

## 4. Rendering Patterns

### 4.1 Rendering Lists
- Render lists of elements with `Array.map()`, transforming each element into a JSX expression.
- Always include a `key` attribute with a unique value per element on the top-level JSX node of each list item. Objects from a server often have an `id` attribute that works perfectly.
- The `key` attribute must remain in the source file that has the loop — React will not see it if moved into the child component.

### 4.2 Conditional Rendering
- Use `&&` for if-then rendering: `{condition && <Component />}`.
- Use the ternary operator `? :` for if-then-else rendering: `{condition ? <ComponentA /> : <ComponentB />}`.

### 4.3 Three-State Rendering for Async Data
- For state variables associated with data loaded from the network, define three specific values:
  - `undefined` → data is being retrieved (show a spinner).
  - `null` → data retrieval failed (show an error message).
  - Actual data → render the content.

---

## 5. State Management

### 5.1 useState Hook
- Use `useState()` to create state variables that store data that needs to be retrieved asynchronously or that can change throughout the life of the component.
- The hook returns an array of `[value, setter]`; use a destructuring assignment: `const [posts, setPosts] = useState();`.
- If an initial value is passed to `useState()`, it becomes the state variable's initial value. If omitted, the variable is `undefined`.
- Never mutate state directly. Always use the setter function to update state so React can trigger re-renders.
- The setter function accepts either a new value or an updater function: `setUpdate(prev => prev + 1)`. Use the function form when the current value is unknown or out of scope.

### 5.2 Dummy State Variables
- A dummy write-only state variable can be used to force a component to re-render when none of its inputs have changed. Only store the setter: `const [, setUpdate] = useState(0);`.

---

## 6. Side Effects

### 6.1 useEffect Hook
- Use `useEffect()` to perform network requests or other asynchronous operations that update state variables.
- The first argument is a function with the side effect logic; the second argument is a dependency array.
- Set the dependency array to `[]` to run the effect only once on initial render.
- A common mistake is to forget the second argument entirely — this makes the effect run on every render, which is rarely necessary.
- Include all variables from outside the effect function that are used inside it as dependencies. The React build process detects and warns about missing dependencies.

### 6.2 Async Effects
- React requires the function given to `useEffect()` to NOT be `async`. Use an Immediately Invoked Function Expression (IIFE) to enable `async`/`await` inside: `(async () => { ... })();`.

### 6.3 Effect Cleanup
- When a side effect allocates resources (e.g., timers, subscriptions), return a cleanup function from the effect. React calls this function when the component unmounts or before re-running the effect.

---

## 7. Refs and Uncontrolled Components

### 7.1 useRef Hook
- Use `useRef()` to create references to DOM elements. Assign the reference via the `ref` attribute on the element.
- Access the underlying DOM element via `refObject.current`.

### 7.2 Uncontrolled Form Components
- For forms, prefer uncontrolled components — use refs and DOM APIs to read field values on submission instead of tracking every change with state variables and event handlers. This significantly reduces boilerplate.

### 7.3 Passing Refs to Child Components
- Since `ref` is a reserved prop name in React, use a different prop name (e.g., `fieldRef`) when passing a reference to a child component, which then assigns it internally to its `ref` attribute.

### 7.4 Auto-Focus Pattern
- Use a side effect function with an empty dependency array to auto-focus the first field in a form on initial render: `useEffect(() => { fieldRef.current.focus(); }, []);`.

---

## 8. Context and Custom Hooks

### 8.1 React Contexts for Shared State
- To share a data item with a subcomponent, pass it as a prop. To share a data item with many subcomponents across several levels, use a context.
- Create contexts with `createContext()` and provide them via `<MyContext.Provider value={...}>`.
- Insert context providers high enough in the component hierarchy that all consumers are children.

### 8.2 Custom Hooks
- Create a custom hook function (named `useXxx()`) that encapsulates the `useContext()` call. This makes consuming code more readable by hiding the context object.
- Export the custom hook as a named export alongside the provider component's default export.
- Hook functions can only be called from component functions or from other hooks; calling a hook in any other context is not allowed.

### 8.3 Context Patterns
- A context is not only useful for sharing data downward. It can enable children components to pass information between themselves with the parent as intermediary (e.g., a flash message system).

---

## 9. API Client Architecture

### 9.1 Centralize API Logic
- Write all back-end calling logic in a single place: an API client class. Do not have each component call `fetch()` directly.
- The API client class should encapsulate: server URL, common path prefixes, query string handling, JSON parsing, error handling, and authentication.
- Provide shortcut methods for each HTTP verb (`get()`, `post()`, `put()`, `delete()`) that delegate to a generic `request()` method.

### 9.2 Sharing the API Client
- Share a single API client instance via a React context and a `useApi()` custom hook, rather than creating instances in every component.

### 9.3 Error Handling in the API Client
- Wrap `fetch()` in a `try/catch` block. When the server is unreachable, construct a synthetic response object with a 500 status code so callers handle network errors the same way as server errors.
- Return a simplified response object with `ok`, `status`, and `body` properties to callers.

### 9.4 Environment Variables for Configuration
- Store the back-end base URL in an environment variable (prefixed with `REACT_APP_`) in a `.env` file.
- Use `.env.production` and `.env.development` files to override values per environment.

---

## 10. Routing

### 10.1 React Router Setup
- Use `BrowserRouter` from `react-router-dom` high in the component hierarchy — it must be a parent to all routing logic.
- Define routes with `<Routes>` and `<Route>`. Use a catch-all `<Route path="*" element={<Navigate to="/" />} />` to redirect unknown URLs.

### 10.2 Page Components
- Create each logical page of the application as a separate page-level component that maps to a route.

### 10.3 SPA-Friendly Links
- Use `Link` or `NavLink` from React-Router instead of standard `<a>` tags to prevent full page reloads.
- Integrate React-Router's `NavLink` with React-Bootstrap's `Nav.Link` using the `as` attribute: `<Nav.Link as={NavLink} to="/">`.
- `NavLink` automatically adds an `active` class when the current URL matches, enabling visual highlighting.

### 10.4 Dynamic Route Parameters
- Use colon-prefixed path segments for dynamic routes: `path="/user/:username"`.
- Access dynamic parameters via the `useParams()` hook.

### 10.5 Programmatic Navigation
- Use the `useNavigate()` hook to navigate programmatically from event handlers (e.g., after form submission).
- Use `useLocation()` to access the current URL and its state (useful for redirect-after-login patterns).

---

## 11. Forms and Validation

### 11.1 Form Structure
- Use a reusable `InputField` component that wraps `Form.Group`, `Form.Label`, `Form.Control`, and `Form.Text` to avoid repeating boilerplate for every field.
- Always call `event.preventDefault()` in the `onSubmit` handler to prevent the browser's default form submission.

### 11.2 Client-Side Validation
- Use client-side validation as a complement, not as a replacement for server-side validation.
- Store validation errors in a `formErrors` state variable (initialized as `{}`). Pass error messages to input fields via props.
- If errors are found, set `formErrors` and return early from the submit handler.

### 11.3 Server-Side Validation
- When the server returns validation errors (e.g., `response.body.errors.json`), set them directly on the `formErrors` state variable if the format matches.

### 11.4 Flash Messages
- Implement a flash message system using a context (`FlashProvider`) that shares a `flash()` function.
- The `flash()` function accepts a message, a type (e.g., `'success'`, `'danger'`), and an optional duration.
- Automatically hide flash messages after a timeout; provide a dismiss button for manual closing.
- Any component can flash a message by calling the `useFlash()` hook — no need to know how or where the alert renders.

---

## 12. Authentication

### 12.1 Token Management
- Store access tokens in the browser's `localStorage` so the user stays logged in across page refreshes, tabs, and browser restarts.
- Be aware that storing tokens in `localStorage` is a risk if the application is vulnerable to XSS attacks. Mitigate by relying on JSX for all rendering (which auto-escapes content) and never rendering content directly via DOM APIs.

### 12.2 API Client Authentication Methods
- Implement `login()`, `logout()`, and `isAuthenticated()` methods on the API client class.
- In `login()`, send credentials with HTTP Basic Authentication, store the returned access token.
- In `logout()`, revoke the token on the server and remove it from `localStorage`.

### 12.3 User Context
- Create a `UserProvider` context that shares the current user object, a `setUser` setter, and `login`/`logout` helper functions.
- On initial render, check for an existing access token and load user info via the API. Set user to `null` if not authenticated.

### 12.4 Private and Public Routes
- Create a `PrivateRoute` wrapper component that renders its children only when a user is logged in; otherwise redirects to `/login` (saving the original URL in route state for redirect-after-login).
- Create a `PublicRoute` wrapper that renders its children only when no user is logged in; otherwise redirects to `/`.
- Group all private routes under a single `PrivateRoute` wrapper with an inner `<Routes>` to reduce boilerplate.

### 12.5 Token Refresh
- Implement transparent token refresh in the API client's `request()` method: if a request returns 401, attempt to refresh the token, then retry the original request.
- Only send cookies (`credentials: 'include'`) for the token endpoint; use `credentials: 'omit'` for all other requests.

---

## 13. Memoization and Performance

### 13.1 Component Memoization with `memo()`
- Wrap components that are frequently re-rendered with the same inputs in `memo()` to skip unnecessary renders: `export default memo(function Post({ post }) { ... });`.
- Do not blindly memoize every component — the logic added by `memo()` has a cost. Use the React DevTools profiler to measure whether memoization improves performance.

### 13.2 Function Memoization with `useCallback()`
- Memoize functions defined inside component render functions with `useCallback()` so they don't change on every render. This prevents child components that depend on them from re-rendering unnecessarily.
- Always declare proper dependencies in the second argument array.

### 13.3 Value Memoization with `useMemo()`
- Use `useMemo()` to memoize non-function values (objects, class instances) created inside render functions. The first argument is a factory function; the second is the dependency array.

### 13.4 Preventing Render Loops
- Render loops occur when circular dependencies between components cause endless re-render cycles (e.g., context provider re-creates functions → child re-renders → triggers another state change → provider re-renders).
- Break loops by memoizing functions and objects defined in context providers with `useCallback()` and `useMemo()`, so they remain stable across re-renders unless their actual dependencies change.

---

## 14. Security

### 14.1 XSS Prevention
- React provides protection against XSS attacks by escaping all text rendered through JSX. This escaping is always applied automatically.
- Never bypass JSX to render content directly through DOM APIs, as this removes XSS protection.
- If you need to store HTML in a variable intended for rendering, use a JSX expression (e.g., `<>You are following <b>{username}</b>.</>`) instead of a plain string with HTML tags.

### 14.2 Access Token Security
- Use short-lived access tokens (e.g., 15 minutes).
- Store refresh tokens in secure, HTTP-only cookies that are inaccessible from JavaScript.
- Only include cookies (`credentials: 'include'`) in requests to the token endpoint.

---

## 15. UI Library Integration (React-Bootstrap)

### 15.1 Importing Components
- Import React-Bootstrap components individually: `import Container from 'react-bootstrap/Container'`. Do not import the entire library, as this inflates application size.

### 15.2 Layout Patterns
- Use `Container` (with or without `fluid`) for responsive margins and alignment.
- Use `Stack` with `direction="horizontal"` for side-by-side layouts.
- Embed layout primitives recursively: for example, a fluid container at root, a navbar inside it, and a non-fluid container inside the navbar for content alignment.

### 15.3 Integrating with Third-Party Components
- Use the `as` attribute on React-Bootstrap components to render them as components from other libraries (e.g., `<Nav.Link as={NavLink}>`).

---

## 16. CSS Styling Practices

### 16.1 Component Class Names
- Assign a `className` matching the component name to the top-level element of each component for CSS targeting.

### 16.2 Overriding Third-Party Styles
- Inspect rendered elements in the browser's developer console to find Bootstrap class names and override them in your custom CSS.
- Import Bootstrap CSS before your application CSS in `index.js` so your styles can override library defaults.

---

## 17. Testing

### 17.1 Test Structure
- Tests follow a three-phase structure: **Render** a component → **Query** for elements of interest → **Assert** that elements have the expected attributes.

### 17.2 Rendering in Tests
- Use `render()` from React Testing Library.
- When rendering individual components, create a minimal wrapper environment (e.g., add `BrowserRouter` if the component uses links, add context providers if the component uses hooks).

### 17.3 Querying Elements
- Use `getBy...()` when the element must exist; `queryBy...()` when it might not exist; `findBy...()` (async) when the element appears after an asynchronous operation.
- Look at the rendered HTML in a browser's developer console to find text, alt text, roles, or titles suitable for queries.
- Add `data-testid` attributes only as a last resort.

### 17.4 Mocking
- Mock `fetch()` with `jest.fn()` and `mockImplementationOnce()` to prevent tests from making real network requests.
- Save the original function before mocking and restore it in `afterEach()`.
- Clear `localStorage` in `afterEach()` to prevent leakage between tests.

### 17.5 Fake Timers
- Use `jest.useFakeTimers()` in `beforeEach()` and `jest.useRealTimers()` in `afterEach()` when testing timer-based behavior.
- Use `jest.runAllTimers()` wrapped in `act()` to advance timers without real waiting.

### 17.6 Testing Async State Changes
- Use `screen.findByText()` (async, returns a promise) to wait for elements that appear after asynchronous operations.
- Wrap actions that trigger state changes (e.g., advancing timers, simulating clicks) in `act()` to let React process all updates.

---

## 18. Production Builds and Deployment

### 18.1 Development vs. Production
- Development builds include extra logging and instrumentation for debugging, at the cost of performance.
- Production builds are optimized for performance and file size.

### 18.2 Generating Production Builds
- Run `npm run build` to create an optimized production build in the `build/` subdirectory.
- Test a production build locally with `npx serve -s build`.

### 18.3 Deployment Requirements
- The web server must serve `index.html` for any non-existent path to support client-side routing (e.g., the `-s` flag for `serve`, or `try_files` in Nginx).
- Cache the contents of `build/static/` for a long time (files have content hashes in their names). Serve all other files with caching disabled.

### 18.4 Environment-Specific Configuration
- Use `.env.production` to override environment variables for production builds. For example, set `REACT_APP_BASE_API_URL=` (empty) when the front end and API share the same origin via a reverse proxy.

### 18.5 Docker Deployment
- Use Nginx as the base image for the front-end container to serve static files and proxy API requests.
- Use Docker Compose to orchestrate front-end and back-end containers with private networking.
- Create a custom `npm run deploy` script that chains build and deployment commands.

---

## 19. Modern JavaScript Conventions

### 19.1 Variables and Constants
- Use `let` for variables, `const` for constants. Never use `var`.
- Use `const` by default; only use `let` when reassignment is needed.

### 19.2 Comparisons
- Always use strict equality `===` and strict inequality `!==`. Never use `==` or `!=`.

### 19.3 Arrow Functions
- Prefer arrow function syntax, especially for callbacks and short functions.
- Omit parentheses for single parameters; omit braces and `return` for single-expression bodies.

### 19.4 Destructuring
- Use destructuring assignments for props, function arguments, array returns from hooks, and object extractions.
- Rename during destructuring to avoid collisions: `const { user: loggedInUser } = useUser();`.

### 19.5 Spread Operator
- Use the spread operator `...` for shallow copies of arrays/objects and for merging: `[...oldArray, ...newArray]`, `{...oldObject, newProp: value}`.

### 19.6 Template Literals
- Use template literals (backtick strings) for string interpolation: `` `Hello, ${name}!` ``.

### 19.7 Async/Await
- Prefer `async`/`await` over `.then()` chains for asynchronous operations.
- Use `try`/`catch` for error handling in async functions.

### 19.8 Modules
- Use ES6 `import`/`export` syntax. A module can have one `default` export and multiple named exports.
- Import default exports without braces; import named exports with braces: `import DefaultExport, { namedExport } from './module'`.
