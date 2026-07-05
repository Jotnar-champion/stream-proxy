Here's a full deep-dive covering each pattern with real-world use, code, and gotchas — organized for a React + Node stack.

# 📚 Quick Resource List (for reference/practice)
- **Patterns.dev** – interactive, React/Node focused
- **Learning JavaScript Design Patterns** (Addy Osmani) – free at patterns.addy.ie
- **Refactoring.Guru** – best diagrams for GoF patterns
- **LogRocket JS Design Patterns Guide** – Node-focused
- **GeeksforGeeks JS Design Patterns** – interview Q&A style

---

# 🏗️ CREATIONAL PATTERNS
*(Concerned with how objects are created)*

## 1. Constructor Pattern
**What:** Use a function/class as a blueprint to create multiple similar objects.

**Real-life:** A car factory blueprint — every car built from it has wheels, engine, color, but each instance is independent.

```js
class User {
  constructor(name, role) {
    this.name = name;
    this.role = role;
  }
  greet() { return `Hi, I'm ${this.name} (${this.role})`; }
}
const admin = new User('Alice', 'admin');
```

**When/Why:** Whenever you need multiple instances of an entity with shared behavior but distinct state (e.g., `User`, `Product`, `OrderItem` models in a Node API).

**Edge cases:**
- Forgetting `new` → `this` becomes `undefined`/global in non-strict mode. Use classes (they throw `TypeError` if called without `new`) instead of plain functions to avoid this.
- Avoid putting methods inside the constructor (`this.greet = function(){}`) — it duplicates the function per instance, wasting memory. Put methods on the prototype (which `class` does automatically).

---

## 2. Factory Pattern
**What:** A function that creates and returns objects without exposing the instantiation logic or requiring the caller to use `new` directly. Useful when object creation logic is complex or varies by input.

**Real-life:** A restaurant kitchen — you order "pizza", the kitchen (factory) decides how to make it; you don't need to know the recipe.

```js
// Node.js example: creating different notification senders
function notificationFactory(type) {
  switch (type) {
    case 'email': return { send: (msg) => sendEmail(msg) };
    case 'sms':   return { send: (msg) => sendSMS(msg) };
    case 'push':  return { send: (msg) => sendPush(msg) };
    default: throw new Error('Unknown notification type');
  }
}
const notifier = notificationFactory('email');
notifier.send('Order shipped!');
```

**When/Why:** Use when object creation involves conditional logic, or when the exact class isn't known until runtime (e.g., creating different DB connectors, different payment gateway clients based on config).

**Edge cases:**
- Don't overuse it for simple objects — adds unnecessary indirection.
- If the factory grows too many branches, consider **Abstract Factory** (a factory of factories) or a registry/map pattern instead of a switch statement.

---

## 3. Singleton Pattern
**What:** Ensures a class has only **one instance** and provides a global access point to it.

**Real-life:** A country's government — there's only one active instance; everyone accesses the same one.

```js
// Node.js: DB connection singleton
class Database {
  constructor() {
    if (Database.instance) return Database.instance;
    this.connection = connectToDB();
    Database.instance = this;
  }
}
const db1 = new Database();
const db2 = new Database();
console.log(db1 === db2); // true
```
In modern JS, ES Modules are naturally singletons — a module's exported object is cached after first import:
```js
// db.js
const connection = connectToDB();
export default connection; // same instance everywhere it's imported
```

**When/Why:** Config managers, DB connections, logging services, caching layers — anything where you want exactly one shared instance and shared state across your app (e.g., a Redis client in Node).

**Edge cases:**
- **Testing difficulty:** Singletons carry global state, making unit tests flaky (state leaks between tests). Mitigate with a `resetInstance()` method for tests.
- **Hidden dependencies:** Code using a singleton doesn't declare it as a dependency (no dependency injection), hurting testability — prefer DI where possible.
- Not thread-related in JS (single-threaded), but be careful with **module caching busting** (e.g., `require.cache` deletion in Node) which can accidentally create a second instance.

---

# 🧱 STRUCTURAL PATTERNS
*(Concerned with composing objects/classes)*

## 4. Module Pattern
**What:** Encapsulate private state and expose only a public API, using closures (or ES Modules natively).

**Real-life:** A vending machine — you interact with buttons (public API); the internal wiring/inventory (private state) is hidden.

```js
const CounterModule = (function () {
  let count = 0; // private
  return {
    increment: () => ++count,
    reset: () => (count = 0),
    getCount: () => count,
  };
})();
```
Modern equivalent — ES Modules give you this for free:
```js
// counter.js
let count = 0;
export const increment = () => ++count;
export const getCount = () => count;
```

**When/Why:** Encapsulating implementation details, avoiding global namespace pollution — every Node file/React hook file is effectively using this pattern already.

**Edge cases:**
- IIFE-based modules make private state hard to unit test directly (only via public API — which is often the *point*).
- Watch for accidental shared mutable state if the module is imported in multiple places expecting fresh state (ES Modules are singletons — see above).

---

## 5. Decorator Pattern
**What:** Dynamically add new behavior/responsibilities to an object without modifying its original class.

**Real-life:** Adding toppings to a coffee — base coffee stays the same class, but you "wrap" it with milk, sugar, etc.

```js
function withLogging(fn) {
  return function (...args) {
    console.log(`Calling ${fn.name} with`, args);
    return fn(...args);
  };
}
const add = (a, b) => a + b;
const loggedAdd = withLogging(add);
loggedAdd(2, 3); // logs call, then returns 5
```
**React equivalent — Higher-Order Components (HOC):**
```jsx
function withAuth(Component) {
  return function Wrapped(props) {
    const { user } = useAuth();
    if (!user) return <Redirect to="/login" />;
    return <Component {...props} user={user} />;
  };
}
const ProtectedDashboard = withAuth(Dashboard);
```

**When/Why:** Cross-cutting concerns — logging, auth checks, caching, memoization — without touching the original function/component. Express middleware is essentially the decorator pattern applied to `req/res`.

**Edge cases:**
- Overusing HOCs causes "wrapper hell" (deeply nested trees, hard to debug in React DevTools) — modern React prefers **custom hooks** for this reason.
- Order of decorators matters (`withAuth(withLogging(Component))` vs reverse) — behavior can differ.

---

## 6. Adapter Pattern
**What:** Converts one interface into another that a client expects, letting incompatible interfaces work together.

**Real-life:** A power plug adapter — lets a US plug work in a European socket without changing the appliance.

```js
// Legacy API returns { first_name, last_name }; new UI expects { name }
function userAdapter(legacyUser) {
  return {
    name: `${legacyUser.first_name} ${legacyUser.last_name}`,
    id: legacyUser.user_id,
  };
}
// Node.js: adapting a 3rd-party payment SDK to your internal PaymentGateway interface
class StripeAdapter {
  constructor(stripeClient) { this.stripe = stripeClient; }
  charge(amount) { return this.stripe.createCharge({ amount_cents: amount * 100 }); }
}
```

**When/Why:** Integrating 3rd-party libraries/APIs, migrating legacy backends incrementally, normalizing inconsistent API response shapes in a React frontend.

**Edge cases:**
- Chaining too many adapters adds performance overhead and debugging complexity.
- Keep adapters thin — if they start containing business logic, that's scope creep (should be a service/use case, not an adapter).

---

## 7. Facade Pattern
**What:** Provides a simplified interface to a complex subsystem.

**Real-life:** A car's ignition key — one turn hides the complex engine-starting subsystem.

```js
// Simplifying multiple Node subsystems (auth, db, cache) into one call
class UserService {
  async getUser(id) {
    let user = await cache.get(id);
    if (!user) {
      user = await db.users.findById(id);
      await cache.set(id, user);
    }
    return user;
  }
}
```
**When/Why:** Wrapping complex library calls (e.g., `axios` + retry + auth headers into one `apiClient.get()`), simplifying a React component's interaction with multiple contexts/hooks.

**Edge cases:** Can become a "god object" if it absorbs too much logic — keep it a thin coordination layer, not a dumping ground.

---

# 🔄 BEHAVIORAL PATTERNS
*(Concerned with communication between objects)*

## 8. Observer Pattern
**What:** An object (subject) maintains a list of dependents (observers) and notifies them automatically of state changes.

**Real-life:** YouTube subscriptions — subscribers get notified when a channel uploads, without polling.

```js
class EventEmitter {
  constructor() { this.listeners = {}; }
  on(event, cb) { (this.listeners[event] ??= []).push(cb); }
  emit(event, data) { (this.listeners[event] || []).forEach(cb => cb(data)); }
}
const bus = new EventEmitter();
bus.on('orderPlaced', (order) => console.log('Send email for', order));
bus.emit('orderPlaced', { id: 42 });
```
**Node.js:** built-in `EventEmitter` class uses this exact pattern (used everywhere — streams, HTTP server events).
**React:** State updates + `useEffect` subscriptions, and libraries like Redux (`store.subscribe`) are Observer under the hood.

**When/Why:** Decoupling producers/consumers of events — real-time notifications, pub/sub microservices communication (Node), UI reactivity (React state).

**Edge cases:**
- **Memory leaks:** forgetting to unsubscribe (`removeListener` / cleanup in `useEffect`) keeps references alive.
- Order of notification isn't guaranteed to matter but can cause subtle bugs if observers depend on execution order.
- Too many observers on one event → hard to trace data flow ("callback hell" for debugging) — mitigate with clear event naming/logging.

---

## 9. Strategy Pattern
**What:** Define a family of interchangeable algorithms and select one at runtime.

**Real-life:** Google Maps route options — walking, driving, transit — same goal, different strategy.

```js
const strategies = {
  creditCard: (amount) => chargeCreditCard(amount),
  paypal: (amount) => chargePaypal(amount),
  crypto: (amount) => chargeCrypto(amount),
};
function checkout(method, amount) {
  return strategies[method](amount);
}
```
**When/Why:** Avoiding long if/else or switch chains for interchangeable business rules — sorting algorithms, validation rules, pricing strategies in Node services.

**Edge cases:** If strategies need very different input/output shapes, the abstraction leaks — keep interfaces consistent across strategies.

---

## 10. Command Pattern
**What:** Encapsulate a request/action as an object, allowing queuing, undo, and logging.

**Real-life:** A restaurant order slip — decouples the waiter (invoker) from the chef (receiver); slip can be queued, cancelled, or redone.

```js
class AddTodoCommand {
  constructor(store, todo) { this.store = store; this.todo = todo; }
  execute() { this.store.add(this.todo); }
  undo() { this.store.remove(this.todo); }
}
```
**When/Why:** Undo/redo systems, task queues (Node job queues like Bull), Redux actions (each action is essentially a Command object).

**Edge cases:** Storing large command histories for undo can bloat memory — consider capping history size or using diffs.

---

## 11. Mediator Pattern
**What:** Centralizes communication between components so they don't reference each other directly.

**Real-life:** An air traffic control tower — planes don't talk to each other directly; the tower mediates.

```js
// React: Context API as a Mediator between deeply nested components
const ChatContext = createContext();
function ChatProvider({ children }) {
  const [messages, setMessages] = useState([]);
  const sendMessage = (msg) => setMessages(prev => [...prev, msg]);
  return <ChatContext.Provider value={{ messages, sendMessage }}>{children}</ChatContext.Provider>;
}
```
**When/Why:** Reducing tight coupling between many components/services that need to talk to each other (chat apps, form wizards with cross-field validation, microservice orchestration in Node).

**Edge cases:** The mediator itself can become a bottleneck/god-object if it absorbs too much logic — keep it focused on coordination only.

---

# 🏛️ ARCHITECTURAL PATTERNS

## 12. MVC (Model-View-Controller)
**What:** Separates data (Model), UI (View), and input-handling/business logic (Controller).

**Real-life:** A restaurant — Model = kitchen/inventory, View = menu shown to customer, Controller = waiter relaying orders and responses.

```js
// Express.js (classic Node MVC)
// Model
const UserModel = { findById: (id) => db.query('SELECT * FROM users WHERE id=?', id) };
// Controller
app.get('/users/:id', async (req, res) => {
  const user = await UserModel.findById(req.params.id);
  res.render('userView', { user }); // View
});
```

**When/Why:** Standard for Express/Node REST APIs and server-rendered apps — clean separation of concerns, easy to test Controllers/Models independently.

**Edge cases:**
- "Fat controllers" — business logic creeps into controllers instead of a Service layer. Mitigate with MVC + Service layer (Controller → Service → Model).
- In React, MVC doesn't map cleanly (no single "Controller") — React is more Component-driven; Redux/Context takes over Controller-like responsibilities.

---

## 13. MVP (Model-View-Presenter)
**What:** Like MVC, but the **Presenter** handles all UI logic and updates the View via an interface — View is passive (dumb), knows nothing about the Model.

**Real-life:** A TV weather presenter — reads data (Model) and *tells* the screen (View) exactly what to display; the screen has no logic of its own.

```jsx
// React approximation: Presentational vs Container components
// Presenter (Container) - fetches data, has logic
function UserPresenter() {
  const [user, setUser] = useState(null);
  useEffect(() => { fetchUser().then(setUser); }, []);
  return <UserView user={user} />;
}
// View - purely renders props, no logic
function UserView({ user }) {
  if (!user) return <Spinner />;
  return <div>{user.name}</div>;
}
```

**When/Why:** Maximizing testability — Presenter can be unit tested without rendering UI. Common in Android/Angular; in React it maps to the **Container/Presentational component** split (though hooks have largely replaced this need).

**Edge cases:** In React, hooks (`useUser()`) often replace the need for explicit Presenters — don't force MVP where a custom hook is simpler/idiomatic.

---

## 14. Bonus: Provider / HOC / Hooks (React-specific "patterns" often asked in interviews)
- **Provider Pattern:** Context API (`<ThemeProvider>`) — avoids prop drilling. Ask "why not just props?" → answer: deeply nested trees.
- **Compound Component Pattern:** `<Select><Select.Option/></Select>` — components share implicit state via Context, giving flexible composition (used in libraries like Radix/Headless UI).
- **Render Props:** passing a function as a child/prop to share logic (`<DataFetcher render={data => ...}/>`) — largely superseded by hooks but still asked in interviews.

---

# 🎯 Interview Cheat-Sheet Summary

| Pattern | Category | One-line "why" |
|---|---|---|
| Constructor | Creational | Blueprint for multiple similar objects |
| Factory | Creational | Hide/centralize complex creation logic |
| Singleton | Creational | Exactly one shared instance (config, DB) |
| Module | Structural | Encapsulate private state, expose public API |
| Decorator | Structural | Add behavior without modifying original (HOCs, middleware) |
| Adapter | Structural | Bridge incompatible interfaces (3rd-party APIs) |
| Facade | Structural | Simplify complex subsystem access |
| Observer | Behavioral | Decouple event producers/consumers (EventEmitter, React state) |
| Strategy | Behavioral | Swap algorithms at runtime |
| Command | Behavioral | Encapsulate actions for undo/queue (Redux actions) |
| Mediator | Behavioral | Centralize inter-component communication (Context API) |
| MVC | Architectural | Separate data/UI/logic (Express apps) |
| MVP | Architectural | Passive View, testable Presenter (Container/Presentational) |

Want me to scaffold an actual **runnable React + Node demo repo** implementing all of these side-by-side (e.g., `/patterns/singleton`, `/patterns/observer`, etc.) so you can run and debug each one?
