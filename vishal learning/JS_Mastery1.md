Deepanshu-Deepanshu_fisglbl: consider yourself a master of JS and web development and give me a sheet with answers to below questions in depth the answers should be good enough and detailed and easy to understand witout being unecessarily long for me to help crack tech lead level interviews also prepare a quiz to solve on these topics and questions and other questions that you think might be related
JS,HTML,CSS,REACT,NODEJS,OPTIMIZATION,SECURITY & OUTPUT/ CODING QUESTION
1. Let, var and const difference in terms of hoisting and scoping
2. Explain passed by value and passed by reference.
3. What is the purpose of context or this keyword and how it behaves in a function declaration and arrow function
4. How to achieve inheritance in JS without classes
5. What is callback hell and how it got resolved
6. Async await vs promise where to use one
7. Difference between normal function and arrow function
8. What is closure
9. How async action are handled in JS (event loop concept)
10. Synthetic events vs HTML Events
11. What is event delegation
12. Difference between Html 4 and Html 5
13. What is semantic and non-semantic tag in html
14. Difference between cookies, local storage and session storage and which to use for which purpose
15. What are meta tags
16. SSR vs CSR pros and cons and how to decide which one to choose
17. How the Browser is able to render a html page complete cycle from html parsing to paint
18. What is box model in CSS and how to change its behavior
19. Ways of rendering a div in the center of page
20. Display flex vs grid
21. Different types of positions and how they behave like static, relative, absolute, fixed etc
22. Block elements vs inline vs inline-block elements
23. What is mobile first approach and how will you use media queries to achieve that
24. Difference between Class based and function based component and when to use one
25. Ways of optimizing your react application
26. Real life use case or example of both useMemo and useState
27. useEffect vs useLayoutEffect
28. Redux architecture and redux middleware explanations and basic code if you want
29. Redux vs context api difference and how to decide which to use
30. What is state batching in react and how it is possible
31. Reconciliation process in react and virtual dom concept
32. How to use forwardRef and it’s real life example
33. How to render React as SSR
34. How to do SEO in a react app
35. What are HOC 
36. What is Node JS and is it single threaded
37. For what kind of app will you use node js for 
38. Explain concurrency in node(it should include threadpool, libuv and eventloop)
39. How is node event driven and what are the methods for a eventemitter class
40. What are middlewares
41. Different Verbs of Rest APIsand how to use each


GitHub Copilot: # 🚀 Tech Lead Interview Prep Sheet

---

## JAVASCRIPT

---

### 1. Let, Var, Const — Hoisting & Scoping

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Yes (initialized as `undefined`) | Yes (TDZ — not initialized) | Yes (TDZ — not initialized) |
| Re-declare | ✅ | ❌ | ❌ |
| Re-assign | ✅ | ✅ | ❌ |

**Temporal Dead Zone (TDZ):** The period between hoisting and initialization where accessing `let`/`const` throws `ReferenceError`.

```js
console.log(a); // undefined (var hoisted)
console.log(b); // ReferenceError (TDZ)
var a = 1;
let b = 2;
```

> **Rule of thumb:** Always use `const` by default, `let` when reassignment needed, never `var`.

---

### 2. Passed by Value vs Passed by Reference

- **Primitives** (string, number, boolean, null, undefined, symbol) → **by value** (copy)
- **Objects/Arrays/Functions** → **by reference** (pointer to same memory)

```js
// By Value
let a = 10;
let b = a;
b = 20;
console.log(a); // 10 — unaffected

// By Reference
let obj1 = { name: "Alice" };
let obj2 = obj1;
obj2.name = "Bob";
console.log(obj1.name); // "Bob" — same reference

// To avoid mutation — shallow copy
let obj3 = { ...obj1 };
// deep copy
let obj4 = JSON.parse(JSON.stringify(obj1));
// or structuredClone(obj1)
```

---

### 3. `this` Keyword / Context

`this` refers to the **execution context** — who called the function.

| Scenario | `this` refers to |
|---|---|
| Global scope | `window` (browser) / `global` (node) |
| Object method | The object |
| Regular function (strict mode) | `undefined` |
| Arrow function | Lexical `this` (inherits from surrounding scope) |
| `new` keyword | Newly created object |
| `call/apply/bind` | Explicitly set |

```js
const obj = {
  name: "Alice",
  greet: function() { console.log(this.name); }, // "Alice"
  greetArrow: () => { console.log(this.name); }  // undefined (lexical this = global)
};

// Arrow functions DO NOT have their own 'this'
// Critical in React: always use arrow functions or bind in class components
```

---

### 4. Inheritance in JS Without Classes (Prototypal Inheritance)

```js
// Method 1: Object.create
const animal = {
  speak() { console.log(`${this.name} makes a sound`); }
};

const dog = Object.create(animal);
dog.name = "Rex";
dog.bark = function() { console.log("Woof!"); };
dog.speak(); // Rex makes a sound

// Method 2: Constructor Functions
function Animal(name) { this.name = name; }
Animal.prototype.speak = function() { console.log(`${this.name} speaks`); };

function Dog(name, breed) {
  Animal.call(this, name); // inherit properties
  this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype); // inherit methods
Dog.prototype.constructor = Dog;

const rex = new Dog("Rex", "Lab");
rex.speak(); // Rex speaks
```

> **Key concept:** Every JS object has a `[[Prototype]]` chain. `Object.create` sets that chain explicitly.

---

### 5. Callback Hell & Resolution

**Callback Hell:** Nested callbacks that make code unreadable/unmaintainable.

```js
// Callback Hell (Pyramid of Doom)
getUser(id, function(user) {
  getOrders(user, function(orders) {
    getInvoice(orders[0], function(invoice) {
      sendEmail(invoice, function(result) {
        // ... keep going
      });
    });
  });
});
```

**Solutions:**

```js
// 1. Promises (ES6)
getUser(id)
  .then(user => getOrders(user))
  .then(orders => getInvoice(orders[0]))
  .then(invoice => sendEmail(invoice))
  .catch(err => console.error(err));

// 2. Async/Await (ES8) — cleanest
async function processUser(id) {
  try {
    const user = await getUser(id);
    const orders = await getOrders(user);
    const invoice = await getInvoice(orders[0]);
    await sendEmail(invoice);
  } catch(err) {
    console.error(err);
  }
}
```

---

### 6. Async/Await vs Promises — When to Use

| | Promises | Async/Await |
|---|---|---|
| Readability | Chained `.then()` | Synchronous-looking |
| Error handling | `.catch()` | `try/catch` |
| Parallel execution | `Promise.all()` | `await Promise.all()` |
| Debugging | Harder | Easier (stack traces) |

```js
// Use Promises when: chaining multiple independent ops
Promise.all([fetchUser(), fetchPosts(), fetchComments()])
  .then(([user, posts, comments]) => { /* all resolved */ });

// Use Async/Await when: sequential operations, better readability
async function loadDashboard() {
  const user = await fetchUser();
  const posts = await fetchPosts(user.id); // depends on user
  return { user, posts };
}

// Parallel with async/await
async function loadAll() {
  const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);
}
```

> **Rule:** Prefer `async/await` for readability. Use `Promise.all` for parallel execution.

---

### 7. Normal Function vs Arrow Function

| | Normal Function | Arrow Function |
|---|---|---|
| `this` | Dynamic (caller) | Lexical (enclosing scope) |
| `arguments` object | ✅ | ❌ |
| Used as constructor | ✅ | ❌ |
| `prototype` property | ✅ | ❌ |
| Hoisting | ✅ (declaration) | ❌ |

```js
// Normal function — own 'this'
function Timer() {
  this.seconds = 0;
  setInterval(function() {
    this.seconds++; // BUG: 'this' = window/global
  }, 1000);
}

// Arrow function — lexical 'this'
function Timer() {
  this.seconds = 0;
  setInterval(() => {
    this.seconds++; // CORRECT: 'this' = Timer instance
  }, 1000);
}
```

---

### 8. Closure

A **closure** is when an inner function retains access to variables from its outer function's scope even after the outer function has returned.

```js
function makeCounter() {
  let count = 0; // private variable
  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count
  };
}

const counter = makeCounter();
counter.increment(); // 1
counter.increment(); // 2
counter.getCount();  // 2
// 'count' is not accessible directly — data privacy via closure

// Real use case: function factory
function multiplier(factor) {
  return (num) => num * factor; // closes over 'factor'
}
const double = multiplier(2);
const triple = multiplier(3);
double(5); // 10
triple(5); // 15
```

---

### 9. Event Loop — How Async Works in JS

JS is **single-threaded** but handles async via the **Event Loop**.

```
Call Stack → Web APIs → Callback Queue (Macro) → Microtask Queue → Event Loop
```

**Priority Order:**
1. **Call Stack** (synchronous code)
2. **Microtask Queue** (Promises, `queueMicrotask`, `MutationObserver`)
3. **Macro Task Queue** (setTimeout, setInterval, I/O, UI events)

```js
console.log("1");                          // sync

setTimeout(() => console.log("2"), 0);    // macrotask

Promise.resolve().then(() => console.log("3")); // microtask

console.log("4");                          // sync

// Output: 1, 4, 3, 2
// Microtasks (Promise) run BEFORE macrotasks (setTimeout)
```

**Flow:**
- Sync code runs on Call Stack
- Async ops (fetch, setTimeout) go to Web APIs
- On completion, callbacks go to respective queues
- Event loop picks from microtask first, then macrotask

---

### 10. Synthetic Events vs HTML Events

| | HTML/Native Events | React Synthetic Events |
|---|---|---|
| Origin | Browser DOM | React wrapper |
| Cross-browser | Inconsistent | Normalized |
| Pooling (React <17) | N/A | Reused (nullified after) |
| Access after async | Direct | Need `e.persist()` (React <17) |

```js
// React Synthetic Event
function handleClick(e) {
  e.preventDefault(); // works same as native
  console.log(e.nativeEvent); // access native event
  // React 17+: no more event pooling, safe to use async
  setTimeout(() => console.log(e.target), 0); // works in React 17+
}
```

> React wraps native events for **cross-browser consistency** and **performance** (used to pool/reuse event objects in React <17).

---

### 11. Event Delegation

Instead of attaching event listeners to each child, attach **one listener to the parent** and use `event.target` to identify the source.

```js
// Without delegation — inefficient for 1000 items
document.querySelectorAll('li').forEach(li => {
  li.addEventListener('click', handleClick);
});

// With delegation — one listener
document.getElementById('list').addEventListener('click', (e) => {
  if (e.target.tagName === 'LI') {
    console.log('Clicked:', e.target.textContent);
  }
});
// Works even for dynamically added <li> elements!
```

**Benefits:** Performance, works for dynamic elements, less memory usage.

---

## HTML

---

### 12. HTML4 vs HTML5

| Feature | HTML4 | HTML5 |
|---|---|---|
| Doctype | Complex | `<!DOCTYPE html>` |
| Multimedia | Plugins (Flash) | `<audio>`, `<video>` native |
| Semantic tags | None | `<header>`, `<footer>`, `<article>`, etc. |
| Storage | Cookies only | localStorage, sessionStorage, IndexedDB |
| APIs | None | Geolocation, Canvas, Web Workers, WebSockets |
| Form inputs | Limited | `date`, `email`, `range`, `color`, etc. |
| SVG/MathML | External | Inline support |

---

### 13. Semantic vs Non-Semantic Tags

**Semantic:** Clearly describes meaning/content to browser and developer.
```html
<header>, <nav>, <main>, <article>, <section>, 
<aside>, <footer>, <figure>, <time>, <mark>
```

**Non-Semantic:** No meaning about content.
```html
<div>, <span>
```

**Why semantic matters:**
- **SEO** — search engines understand content structure
- **Accessibility** — screen readers navigate properly
- **Maintainability** — self-documenting code

---

### 14. Cookies vs localStorage vs sessionStorage

| | Cookies | localStorage | sessionStorage |
|---|---|---|---|
| Capacity | ~4KB | ~5-10MB | ~5-10MB |
| Expiry | Manual/set | Never (manual clear) | Tab close |
| Server access | Yes (sent in HTTP headers) | No | No |
| Scope | Domain/path | Origin | Tab + Origin |
| Security | `HttpOnly`, `Secure` flags | JS accessible | JS accessible |

**When to use:**
- **Cookies:** Auth tokens (with `HttpOnly`+`Secure`), server-needed data
- **localStorage:** User preferences, theme, non-sensitive persistent data
- **sessionStorage:** Wizard/multi-step form data, tab-specific temp data

> ⚠️ **Never store sensitive data** (passwords, tokens) in localStorage/sessionStorage — XSS vulnerable.

---

### 15. Meta Tags

Tags in `<head>` that provide **metadata** about the document.

```html
<!-- Character encoding -->
<meta charset="UTF-8">

<!-- Viewport for responsive design -->
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<!-- SEO -->
<meta name="description" content="Page description for search engines">
<meta name="keywords" content="react, javascript, tutorial">
<meta name="robots" content="index, follow">

<!-- Open Graph (Social sharing) -->
<meta property="og:title" content="My Page">
<meta property="og:image" content="https://example.com/image.jpg">

<!-- Twitter Cards -->
<meta name="twitter:card" content="summary_large_image">

<!-- Cache control -->
<meta http-equiv="Cache-Control" content="no-cache">
```

---

### 16. SSR vs CSR

| | CSR (Client-Side Rendering) | SSR (Server-Side Rendering) |
|---|---|---|
| Initial load | Slow (blank page until JS loads) | Fast (HTML ready) |
| SEO | Poor (bots see empty HTML) | Excellent |
| Subsequent nav | Fast (SPA) | Slower (new request) |
| Server load | Low | High |
| TTFB | Fast | Slower |
| Real-time apps | Better | Not ideal |

**When to choose:**
- **CSR:** Dashboards, admin panels, authenticated apps, real-time apps
- **SSR:** E-commerce, blogs, marketing pages, SEO-critical apps
- **Hybrid (Next.js):** Best of both — static pages SSR, dynamic parts CSR

---

### 17. Browser Rendering Pipeline

```
URL → DNS Lookup → TCP Connection → HTTP Request → Response
→ HTML Parsing → DOM Tree
→ CSS Parsing → CSSOM Tree
→ DOM + CSSOM = Render Tree
→ Layout (Reflow) — calculate positions/sizes
→ Paint — fill pixels
→ Composite — layer ordering, GPU
→ Display
```

**Key concepts:**
- **Render-blocking:** CSS and sync JS block rendering
- **Reflow:** Layout recalculation (expensive) — triggered by size/position changes
- **Repaint:** Visual change without layout (color change) — cheaper
- **`async`/`defer`** on scripts to avoid blocking

```html
<script defer src="app.js"></script>   <!-- After DOM parsed -->
<script async src="analytics.js"></script> <!-- Parallel, runs when ready -->
```

---

## CSS

---

### 18. Box Model & Changing Behavior

Every element = **Content + Padding + Border + Margin**

```css
/* Default: box-sizing: content-box */
/* width = content only, padding/border ADD to total size */
.box {
  width: 200px;
  padding: 20px;
  border: 5px solid;
  /* Total width = 200 + 40 + 10 = 250px */
}

/* border-box: width INCLUDES padding and border */
*, *::before, *::after {
  box-sizing: border-box; /* Recommended global reset */
}
.box {
  width: 200px;
  padding: 20px;
  border: 5px solid;
  /* Total width = 200px exactly */
}
```

---

### 19. Centering a Div

```css
/* 1. Flexbox (most common) */
.parent {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 2. Grid */
.parent {
  display: grid;
  place-items: center;
}

/* 3. Absolute + Transform */
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* 4. Margin auto (horizontal only) */
.child {
  width: 300px;
  margin: 0 auto;
}

/* 5. Absolute + all sides 0 */
.child {
  position: absolute;
  inset: 0; /* top/right/bottom/left: 0 */
  margin: auto;
  width: fit-content;
  height: fit-content;
}
```

---

### 20. Flexbox vs Grid

| | Flexbox | Grid |
|---|---|---|
| Dimension | 1D (row OR column) | 2D (rows AND columns) |
| Use case | Component layout, nav bars | Page layout, complex grids |
| Content-driven | Yes | No (structure-driven) |
| Alignment | Easy main/cross axis | Full 2D control |

```css
/* Flexbox — navigation */
nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

/* Grid — page layout */
.page {
  display: grid;
  grid-template-columns: 250px 1fr;
  grid-template-rows: 60px 1fr 50px;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
}
```

> **Rule:** Use Flexbox for 1D layouts/components, Grid for 2D page layouts.

---

### 21. CSS Positions

| Position | Normal Flow | Offset (top/left) | Relative to |
|---|---|---|---|
| `static` | ✅ | ❌ | N/A |
| `relative` | ✅ | ✅ | Itself |
| `absolute` | ❌ | ✅ | Nearest positioned ancestor |
| `fixed` | ❌ | ✅ | Viewport |
| `sticky` | ✅ | ✅ | Scroll container |

```css
/* sticky header */
header {
  position: sticky;
  top: 0;
  z-index: 100;
}

/* absolute inside relative — tooltip */
.container { position: relative; }
.tooltip {
  position: absolute;
  top: 100%;
  left: 0;
}
```

---

### 22. Block vs Inline vs Inline-Block

| | Block | Inline | Inline-Block |
|---|---|---|---|
| New line | ✅ | ❌ | ❌ |
| Width/Height | Settable | Ignored | Settable |
| Margin/Padding | All sides | Horizontal only | All sides |
| Examples | `div`, `p`, `h1` | `span`, `a`, `strong` | `img`, `button` |

```css
span {
  display: inline-block; /* Now accepts width/height */
  width: 100px;
  height: 50px;
}
```

---

### 23. Mobile-First & Media Queries

**Mobile-first:** Write base styles for mobile, use `min-width` media queries to scale up.

```css
/* BASE = Mobile styles */
.container {
  padding: 1rem;
  font-size: 14px;
}

/* Tablet — min-width 768px */
@media (min-width: 768px) {
  .container {
    padding: 2rem;
    font-size: 16px;
  }
}

/* Desktop — min-width 1024px */
@media (min-width: 1024px) {
  .container {
    max-width: 1200px;
    margin: 0 auto;
  }
}

/* vs Desktop-first uses max-width (avoid for new projects) */
@media (max-width: 768px) { /* targets mobile */ }
```

**Why mobile-first?** Progressive enhancement, better performance (mobile downloads less CSS), majority of traffic is mobile.

---

## REACT

---

### 24. Class vs Function Components

| | Class | Function |
|---|---|---|
| State | `this.state` | `useState` |
| Lifecycle | Lifecycle methods | `useEffect` |
| `this` | Required | Not needed |
| Boilerplate | More | Less |
| Performance | Slightly worse | Optimized |
| Hooks | ❌ | ✅ |

```jsx
// Class Component
class Counter extends React.Component {
  state = { count: 0 };
  render() {
    return <button onClick={() => this.setState({ count: this.state.count + 1 })}>
      {this.state.count}
    </button>;
  }
}

// Function Component (preferred)
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

> **When to use class?** Only when working with legacy code or Error Boundaries (no hook equivalent yet).

---

### 25. React Optimization Techniques

```jsx
// 1. React.memo — prevent re-render if props unchanged
const MyComponent = React.memo(({ data }) => <div>{data}</div>);

// 2. useMemo — memoize expensive calculations
const sortedList = useMemo(() => 
  data.sort((a, b) => a - b), [data]
);

// 3. useCallback — memoize functions passed as props
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);

// 4. Code splitting + Lazy loading
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));
<Suspense fallback={<Spinner />}>
  <HeavyComponent />
</Suspense>

// 5. Virtualization for large lists
import { FixedSizeList } from 'react-window';

// 6. Avoid anonymous functions in JSX
// Bad: <button onClick={() => handleClick(id)}>
// Good: <button onClick={handleClick}>

// 7. Key prop correctly in lists
// 8. Avoid unnecessary state — derive when possible
// 9. Bundle splitting — dynamic imports
// 10. Image optimization — lazy loading, WebP format
```

---

### 26. useMemo vs useCallback — Real Examples

```jsx
// useMemo — expensive computation
function ProductList({ products, filterTerm }) {
  const filteredProducts = useMemo(() => {
    console.log("Filtering..."); // only runs when deps change
    return products.filter(p => 
      p.name.toLowerCase().includes(filterTerm.toLowerCase())
    );
  }, [products, filterTerm]);
  
  return filteredProducts.map(p => <Product key={p.id} {...p} />);
}

// useCallback — stable function reference for child component
function Parent() {
  const [count, setCount] = useState(0);
  
  const handleDelete = useCallback((id) => {
    // Without useCallback, new function on every render
    // causing Child to re-render even if its props didn't change
    deleteItem(id);
  }, []); // stable reference
  
  return <Child onDelete={handleDelete} />;
}
const Child = React.memo(({ onDelete }) => { /* ... */ });
```

---

### 27. useEffect vs useLayoutEffect

| | `useEffect` | `useLayoutEffect` |
|---|---|---|
| Timing | After paint (async) | After DOM update, before paint (sync) |
| Blocks paint | No | Yes |
| Use case | Data fetching, subscriptions | DOM measurements, prevent flicker |

```jsx
// useEffect — standard side effects
useEffect(() => {
  fetchData().then(setData);
}, []);

// useLayoutEffect — DOM measurements before paint
useLayoutEffect(() => {
  // Measure DOM, adjust layout
  // Prevents visual flicker
  const { height } = ref.current.getBoundingClientRect();
  setHeight(height);
}, []);
// Example: tooltip positioning — calculate position before paint
```

> **Rule:** Use `useEffect` by default. Only use `useLayoutEffect` when you see flickering or need DOM measurements pre-paint.

---

### 28. Redux Architecture & Middleware

```
UI → Action → Middleware (Thunk/Saga) → Reducer → Store → UI
```

```js
// Store
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';

// Action
const fetchUsers = () => async (dispatch) => {
  dispatch({ type: 'FETCH_START' });
  try {
    const users = await api.getUsers();
    dispatch({ type: 'FETCH_SUCCESS', payload: users });
  } catch(e) {
    dispatch({ type: 'FETCH_ERROR', payload: e.message });
  }
};

// Reducer
function usersReducer(state = { data: [], loading: false }, action) {
  switch(action.type) {
    case 'FETCH_START': return { ...state, loading: true };
    case 'FETCH_SUCCESS': return { data: action.payload, loading: false };
    default: return state;
  }
}

// Modern — Redux Toolkit (RTK) preferred
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

const fetchUsers = createAsyncThunk('users/fetch', async () => {
  return await api.getUsers();
});

const usersSlice = createSlice({
  name: 'users',
  initialState: { data: [], loading: false },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => { state.loading = true; })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.loading = false;
        state.data = action.payload;
      });
  }
});
```

**Middleware:** Functions between action dispatch and reducer. Used for: async ops (thunk), logging, crash reporting.

---

### 29. Redux vs Context API

| | Context API | Redux |
|---|---|---|
| Built-in | ✅ | ❌ (external) |
| DevTools | ❌ | ✅ Excellent |
| Middleware | ❌ | ✅ |
| Performance | Re-renders all consumers | Selective re-renders |
| Complexity | Low | Higher |
| Best for | Theme, auth, locale | Complex shared state, large apps |

```
Use Context API for:
- Simple shared state (theme, user, language)
- Low-frequency updates
- Small-medium apps

Use Redux for:
- Complex state logic
- Many components sharing state
- Frequent updates
- Need time-travel debugging
- Large team
```

---

### 30. State Batching in React

React groups multiple state updates into a **single re-render** for performance.

```jsx
// React 18 — Automatic Batching everywhere
function handleClick() {
  setCount(c => c + 1);  // \
  setName("Alice");       //  > ONE re-render (batched)
  setLoading(false);      // /
}

// React 17 — only batched in event handlers
// In async/promises — NOT batched (React 17)
setTimeout(() => {
  setCount(c => c + 1); // re-render
  setName("Alice");      // re-render — 2 renders in React 17
  // React 18: still ONE render (automatic batching)
}, 0);

// Opt-out of batching (React 18)
import { flushSync } from 'react-dom';
flushSync(() => setCount(c => c + 1)); // immediate render
flushSync(() => setName("Alice"));     // immediate render
```

---

### 31. Reconciliation & Virtual DOM

**Virtual DOM:** Lightweight JS representation of the real DOM.

**Process:**
1. State/props change → New Virtual DOM tree created
2. **Diffing algorithm** compares old vs new VDOM (O(n) complexity)
3. Only **changed nodes** updated in real DOM (**Reconciliation**)

**Diffing rules:**
- Different element types → Destroy & rebuild subtree
- Same element type → Update attributes only
- `key` prop → Identify list items efficiently

```jsx
// Keys help reconciliation
// Bad — index as key (causes bugs on reorder)
{items.map((item, i) => <Item key={i} {...item} />)}

// Good — stable unique id
{items.map(item => <Item key={item.id} {...item} />)}
```

**React Fiber (React 16+):** Reimplementation of reconciler. Allows:
- Splitting rendering work into chunks
- Pause/resume/abort work
- Priority-based updates (Concurrent Mode)

---

### 32. forwardRef — Real Example

Used when parent needs to access child's DOM element or methods.

```jsx
// Child — forward the ref to DOM element
const CustomInput = React.forwardRef((props, ref) => (
  <input
    ref={ref}
    className="custom-input"
    {...props}
  />
));

// Parent — access child's input DOM directly
function SearchBar() {
  const inputRef = useRef(null);
  
  const focusInput = () => inputRef.current.focus();
  
  return (
    <>
      <CustomInput ref={inputRef} placeholder="Search..." />
      <button onClick={focusInput}>Focus</button>
    </>
  );
}

// With useImperativeHandle — expose specific methods
const VideoPlayer = React.forwardRef((props, ref) => {
  const videoRef = useRef();
  
  useImperativeHandle(ref, () => ({
    play: () => videoRef.current.play(),
    pause: () => videoRef.current.pause(),
  }));
  
  return <video ref={videoRef} src={props.src} />;
});
```

---

### 33. React SSR

```jsx
// Next.js (most common approach)

// getServerSideProps — SSR (every request)
export async function getServerSideProps(context) {
  const data = await fetchData(context.params.id);
  return { props: { data } };
}

// getStaticProps — SSG (build time)
export async function getStaticProps() {
  const data = await fetchData();
  return { props: { data }, revalidate: 60 }; // ISR
}

// Pure React SSR (without Next.js)
// Server:
import { renderToString } from 'react-dom/server';
const html = renderToString(<App />);
res.send(`<html><body><div id="root">${html}</div></body></html>`);

// Client:
import { hydrateRoot } from 'react-dom/client';
hydrateRoot(document.getElementById('root'), <App />);
// hydrateRoot attaches event listeners to existing HTML
```

---

### 34. SEO in React

```jsx
// 1. Use SSR/SSG (Next.js) — crawlers see content

// 2. React Helmet / Next.js Head
import Head from 'next/head';
function Page() {
  return (
    <>
      <Head>
        <title>Product Name | Brand</title>
        <meta name="description" content="..." />
        <meta property="og:title" content="..." />
        <link rel="canonical" href="https://..." />
      </Head>
      <main>...</main>
    </>
  );
}

// 3. Semantic HTML in JSX
// 4. Structured data (JSON-LD)
<script type="application/ld+json">
  {JSON.stringify({ "@context": "https://schema.org", "@type": "Product" })}
</script>

// 5. Sitemap generation
// 6. robots.txt
// 7. Core Web Vitals optimization (LCP, FID, CLS)
// 8. Image alt tags, proper heading hierarchy
```

---

### 35. Higher-Order Components (HOC)

A function that takes a component and returns an enhanced component.

```jsx
// HOC for authentication
function withAuth(WrappedComponent) {
  return function AuthComponent(props) {
    const { isAuthenticated } = useAuth();
    
    if (!isAuthenticated) {
      return <Redirect to="/login" />;
    }
    
    return <WrappedComponent {...props} />;
  };
}

const ProtectedDashboard = withAuth(Dashboard);

// HOC for loading state
function withLoading(WrappedComponent) {
  return function({ isLoading, ...props }) {
    if (isLoading) return <Spinner />;
    return <WrappedComponent {...props} />;
  };
}

// Modern alternative: Custom Hooks (preferred over HOC)
function useAuth() {
  const [user] = useContext(AuthContext);
  return { isAuthenticated: !!user, user };
}
```

---

## NODE.JS

---

### 36. What is Node.js — Single Threaded?

**Node.js:** JavaScript runtime built on Chrome's **V8 engine** for server-side JS.

**Yes and No:**
- **Single-threaded** for JS execution (one call stack)
- **Multi-threaded** internally via **libuv** (thread pool for I/O)

```
Your JS code → Single Thread
File I/O, DNS, Crypto → libuv Thread Pool (4 threads default)
Network I/O → OS (non-blocking, no threads needed)
```

---

### 37. When to Use Node.js

**Best for:**
- Real-time applications (chat, gaming, live updates)
- REST/GraphQL APIs
- Microservices
- Streaming applications
- Proxy servers
- I/O heavy applications

**Avoid for:**
- CPU-intensive tasks (video encoding, ML, heavy computations)
- Because CPU tasks block the single JS thread

---

### 38. Concurrency in Node — Event Loop, libuv, Thread Pool

```
┌──────────────────────────────┐
│         Node.js              │
│   ┌──────────────────────┐   │
│   │   Your JS Code       │   │
│   │   (Single Thread)    │   │
│   └──────────┬───────────┘   │
│              │               │
│   ┌──────────▼───────────┐   │
│   │      libuv           │   │
│   │  ┌───────────────┐   │   │
│   │  │  Event Loop   │   │   │
│   │  └───────────────┘   │   │
│   │  ┌───────────────┐   │   │
│   │  │  Thread Pool  │   │   │
│   │  │  (4 threads)  │   │   │
│   │  └───────────────┘   │   │
│   └──────────────────────┘   │
└──────────────────────────────┘
```

**Event Loop Phases (in order):**
1. **Timers** — `setTimeout`, `setInterval`
2. **Pending callbacks** — I/O callbacks deferred
3. **Idle/Prepare** — internal use
4. **Poll** — retrieve new I/O events
5. **Check** — `setImmediate`
6. **Close callbacks** — `socket.on('close')`

Between each phase: **process.nextTick()** and **Promises** (microtasks) run first.

```js
// libuv Thread Pool handles:
// - File system (fs)
// - DNS lookup
// - Crypto
// - zlib

// Network I/O uses OS async mechanisms (epoll/kqueue) — no threads!

// Increase thread pool size
process.env.UV_THREADPOOL_SIZE = 8; // default 4, max 1024
```

---

### 39. Node Event-Driven Architecture & EventEmitter

```js
const EventEmitter = require('events');

class OrderService extends EventEmitter {
  placeOrder(order) {
    // ... process order
    this.emit('orderPlaced', order);
    this.emit('inventoryUpdate', order.items);
  }
}

const orderService = new OrderService();

// Subscribe to events
orderService.on('orderPlaced', (order) => {
  sendConfirmationEmail(order);
});

orderService.on('orderPlaced', (order) => {
  updateDashboard(order);
});

// One-time listener
orderService.once('firstOrder', (order) => {
  sendWelcomeGift(order);
});

// EventEmitter methods:
emitter.on(event, listener)        // add listener
emitter.once(event, listener)      // one-time listener
emitter.emit(event, ...args)       // trigger event
emitter.off(event, listener)       // remove listener
emitter.removeAllListeners(event)  // remove all
emitter.listeners(event)           // get all listeners
emitter.setMaxListeners(n)         // default 10 (memory leak warning)
```

---

### 40. Middlewares (Express)

Functions that have access to `req`, `res`, `next` — executed in a pipeline.

```js
// Middleware signature
function middleware(req, res, next) {
  // do something
  next(); // pass to next middleware
  // or next(error) for error handling
}

// Types:
// 1. Application-level
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// 2. Route-level
app.get('/profile', authenticate, authorize('admin'), (req, res) => {
  res.json(req.user);
});

// 3. Error-handling (4 params)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: err.message });
});

// 4. Third-party
app.use(cors());
app.use(helmet()); // security headers
app.use(express.json());
app.use(morgan('dev')); // logging

// Execution order matters!
app.use(cors());        // 1st
app.use(authenticate);  // 2nd
app.use(router);        // 3rd
app.use(errorHandler);  // Last
```

---

### 41. REST API HTTP Verbs

| Verb | Purpose | Idempotent | Body |
|---|---|---|---|
| GET | Retrieve resource | ✅ | ❌ |
| POST | Create resource | ❌ | ✅ |
| PUT | Replace entire resource | ✅ | ✅ |
| PATCH | Partial update | ❌ | ✅ |
| DELETE | Delete resource | ✅ | Optional |
| HEAD | GET without body | ✅ | ❌ |
| OPTIONS | Get allowed methods | ✅ | ❌ |

```js
// REST API Example
router.get('/users',         getUsers);        // list all
router.get('/users/:id',     getUserById);     // get one
router.post('/users',        createUser);      // create
router.put('/users/:id',     replaceUser);     // full replace
router.patch('/users/:id',   updateUser);      // partial update
router.delete('/users/:id',  deleteUser);      // delete

// Status codes
// 200 OK, 201 Created, 204 No Content
// 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found
// 409 Conflict, 422 Unprocessable Entity
// 500 Internal Server Error, 503 Service Unavailable

// PUT vs PATCH
// PUT: send full object { name, email, age }
// PATCH: send only changed fields { email }
```

---

---

# 🧠 QUIZ TIME

## Section A — JavaScript (Output Questions)

**Q1:** What is the output?
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
<details><summary>Answer</summary>3 3 3 — var is function-scoped, all closures share same `i`. Fix: use `let`.</details>

---

**Q2:** What is the output?
```js
console.log(typeof null);
console.log(null instanceof Object);
```
<details><summary>Answer</summary>"object", false — typeof null is a famous JS bug. null is NOT an Object instance.</details>

---

**Q3:** What is the output?
```js
const a = [1, 2, 3];
const b = [1, 2, 3];
console.log(a == b);
console.log(a === b);
```
<details><summary>Answer</summary>false, false — arrays are objects, compared by reference not value.</details>

---

**Q4:** What is the output?
```js
async function foo() {
  return 1;
}
const result = foo();
console.log(result);
```
<details><summary>Answer</summary>Promise {resolved: 1} — async functions always return a Promise.</details>

---

**Q5:** What is the output?
```js
console.log(0.1 + 0.2 === 0.3);
```
<details><summary>Answer</summary>false — floating point precision issue. Use `Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON`</details>

---

**Q6:** What is the output?
```js
const obj = { a: 1 };
Object.freeze(obj);
obj.a = 99;
console.log(obj.a);
```
<details><summary>Answer</summary>1 — freeze prevents mutation (silently fails in non-strict, throws in strict mode).</details>

---

**Q7:** What does this do?
```js
function* gen() {
  yield 1;
  yield 2;
  yield 3;
}
const g = gen();
console.log(g.next().value);
console.log(g.next().value);
```
<details><summary>Answer</summary>1, 2 — Generator functions pause at each yield, resuming on .next().</details>

---

## Section B — Conceptual Questions

**Q8:** What is the difference between `==` and `===`?
<details><summary>Answer</summary>`==` coerces types before comparing. `===` strict equality, no coercion. Always use `===`.</details>

---

**Q9:** What is the difference between `null` and `undefined`?
<details><summary>Answer</summary>`undefined` = variable declared but not assigned. `null` = intentional absence of value (manually set).</details>

---

**Q10:** What is `debounce` vs `throttle`?
<details><summary>Answer</summary>

```js
// Debounce: execute AFTER delay from last call (search input)
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// Throttle: execute AT MOST once per interval (scroll handler)
function throttle(fn, limit) {
  let inThrottle;
  return (...args) => {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}
```
</details>

---

**Q11:** What are WeakMap and WeakSet?
<details><summary>Answer</summary>Like Map/Set but keys (WeakMap) or values (WeakSet) must be objects and are weakly held — garbage collected when no other references exist. Cannot be iterated. Good for caching/private data without memory leaks.</details>

---

**Q12:** What is the Prototype Chain?
<details><summary>Answer</summary>When accessing a property, JS looks on the object, then its `__proto__`, then that object's `__proto__`, up to `Object.prototype` (null). This chain is prototype inheritance.</details>

---

## Section C — React

**Q13:** What happens when you call `setState` inside `useEffect` without a dependency array?
<details><summary>Answer</summary>Infinite loop — useEffect runs after render, setState triggers re-render, which triggers useEffect again.</details>

---

**Q14:** Why shouldn't you use index as a key in lists?
<details><summary>Answer</summary>When items reorder/add/remove, React uses keys to track elements. Index keys cause React to reuse wrong DOM nodes, leading to state bugs, incorrect animations, and performance issues.</details>

---

**Q15:** What is the difference between `useRef` and `useState`?
<details><summary>Answer</summary>

| | useState | useRef |
|---|---|---|
| Triggers re-render | ✅ | ❌ |
| Persists across renders | ✅ | ✅ |
| Use case | UI state | DOM refs, mutable values |
</details>

---

## Section D — CSS Challenges

**Q16:** What is specificity? What wins?
```css
#header .nav a { color: red; }     /* ? */
.nav a:hover { color: blue; }      /* ? */
a { color: green !important; }     /* ? */
```
<details><summary>Answer</summary>

Specificity = (inline, IDs, classes, elements)
- `#header .nav a` = (0,1,1,1) = 111
- `.nav a:hover` = (0,0,2,1) = 021
- `!important` overrides all (avoid using it)

**Winner: green** due to `!important`.
</details>

---

**Q17:** What is the difference between `display:none`, `visibility:hidden`, and `opacity:0`?
<details><summary>Answer</summary>

| | Renders | Visible | Takes Space | Events |
|---|---|---|---|---|
| `display:none` | ❌ | ❌ | ❌ | ❌ |
| `visibility:hidden` | ✅ | ❌ | ✅ | ❌ |
| `opacity:0` | ✅ | ❌ | ✅ | ✅ |
</details>

---

## Section E — Node.js / Architecture

**Q18:** What is the difference between `process.nextTick()` and `setImmediate()`?
<details><summary>Answer</summary>`process.nextTick()` runs before the next event loop iteration (microtask-like, highest priority). `setImmediate()` runs in the Check phase of the event loop. `nextTick` > Promises > `setImmediate` > `setTimeout(0)`</details>

---

**Q19:** How would you prevent a Node.js app from crashing on unhandled rejections?
```js
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection:', reason);
  // graceful shutdown
  process.exit(1);
});

process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  process.exit(1);
});
```

---

**Q20:** Design a REST API for a blog system. Define the endpoints.
<details><summary>Answer</summary>

```
GET    /posts              — list posts (pagination: ?page=1&limit=10)
GET    /posts/:id          — get post
POST   /posts              — create post (auth required)
PUT    /posts/:id          — full update (auth + owner)
PATCH  /posts/:id          — partial update (auth + owner)
DELETE /posts/:id          — delete (auth + owner)

GET    /posts/:id/comments — list comments
POST   /posts/:id/comments — add comment (auth)
DELETE /posts/:id/comments/:commentId — delete comment

GET    /users/:id/posts    — posts by user
POST   /auth/login         — authenticate
POST   /auth/logout
POST   /auth/refresh       — refresh token
```
</details>

---

## Bonus — Security Questions

**Q21:** What is XSS and how to prevent it in React?
<details><summary>Answer</summary>

**Cross-Site Scripting:** Injecting malicious scripts into web pages.

React auto-escapes JSX — `{userInput}` is safe. 

**Danger:** `dangerouslySetInnerHTML` — only use with sanitized content:
```js
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />
```
Other prevention: CSP headers, HTTPOnly cookies, input validation.
</details>

---

**Q22:** What is CSRF and how to prevent it?
<details><summary>Answer</summary>

**Cross-Site Request Forgery:** Tricking authenticated users into making unintended requests.

Prevention:
- **CSRF tokens** (synchronizer token pattern)
- **SameSite cookie attribute** (`SameSite=Strict/Lax`)
- Check `Origin`/`Referer` headers
- `double submit cookie` pattern
</details>

---

> 💡 **Pro Tips for Tech Lead Interviews:**
> - Always mention **trade-offs** when comparing technologies
> - Give **real-world examples** from your projects
> - Discuss **performance implications** of your choices
> - Mention **team scalability** and **maintainability** considerations
> - Show awareness of **security implications**
