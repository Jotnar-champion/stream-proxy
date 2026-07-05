# TypeScript — Senior Engineer Cheat Sheet

## 1. Type System Fundamentals
- **Structural typing** ("duck typing"): compatibility is shape-based, not name-based. Two unrelated types with identical members are interchangeable.
- **Soundness tradeoffs**: TS is intentionally *unsound* in places (e.g., array covariance, `any`, function param bivariance) for ergonomics. Know where: casting, `!` non-null assertion, index signatures.
- **Erased at runtime** — no reflection on interfaces/types. Only `class` and `enum` leave a runtime trace.
- **Declaration vs statement space**: `type`/`interface` live in a separate namespace from `const`/`let`, so a type and a value can share a name (e.g. classes).
- **Widening vs narrowing**: `let x = "a"` widens to `string`; `const x = "a"` keeps literal type `"a"`. Use `as const` to freeze literal/tuple types.

```ts
const point = { x: 1, y: 2 } as const;      // { readonly x: 1; readonly y: 2 }
const tuple = [1, 2] as const;              // readonly [1, 2]
```

---

## 2. Advanced Type Construction

### Mapped types
```ts
type Partial<T> = { [K in keyof T]?: T[K] };
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };

// key remapping (TS 4.1+)
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
```

### Conditional types & `infer`
```ts
type ElementType<T> = T extends (infer U)[] ? U : T;
type ReturnOf<T> = T extends (...args: any[]) => infer R ? R : never;
type Awaited2<T> = T extends Promise<infer U> ? Awaited2<U> : T; // recursive

// distributive conditional types (naked type param distributes over unions)
type ToArray<T> = T extends any ? T[] : never;
type A = ToArray<string | number>; // string[] | number[]
// prevent distribution by wrapping in tuple:
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
```

### Template literal types
```ts
type Route = `/users/${number}`;
type EventName<T extends string> = `on${Capitalize<T>}`;
type CSSProp = `${"margin"|"padding"}-${"top"|"bottom"|"left"|"right"}`;
```

### `keyof`, indexed access, `typeof`
```ts
type Keys = keyof User;              // "id" | "name" | ...
type IdType = User["id"];            // number
const config = { retries: 3 };
type Config = typeof config;         // { retries: number }
```

### Variadic tuple types
```ts
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];
function tail<T extends unknown[]>(arr: readonly [unknown, ...T]): T { /* ... */ return arr.slice(1) as T; }
```

---

## 3. Generics — Beyond Basics
- **Variance**: function params are checked *bivariantly* under `strictFunctionTypes` for method syntax but *contravariantly* for function-typed properties — know this distinction when designing callback-heavy APIs.
- **Constraints + defaults**:
```ts
interface Repo<T extends { id: string | number } = { id: string }> {
  find(id: T["id"]): T | undefined;
}
```
- **`this` parameter typing**:
```ts
function bindHandler(this: HTMLElement, e: Event) { this.classList.add("x"); }
```
- **Higher-order generic functions / currying**:
```ts
const map = <T, U>(fn: (x: T) => U) => (arr: T[]): U[] => arr.map(fn);
```
- **Overload resolution order** matters — most specific overload first; last matching signature wins for implementation shape.

---

## 4. Type Narrowing (Full Toolkit)
| Technique | Example |
|---|---|
| `typeof` | `typeof x === "string"` |
| `instanceof` | `x instanceof Error` |
| `in` | `"prop" in obj` |
| Discriminated union | `switch(x.kind)` |
| Custom predicate | `function isX(v): v is X` |
| Assertion function | `function assertIsX(v): asserts v is X` |
| Equality narrowing | `x === "literal"` |
| Truthiness | `if (x)` |
| Control-flow analysis via `never` | exhaustiveness checks in `default` |

```ts
function assertIsDefined<T>(val: T): asserts val is NonNullable<T> {
  if (val === null || val === undefined) throw new Error("expected value");
}

function exhaustiveCheck(x: never): never { throw new Error(`Unhandled: ${x}`); }
```

---

## 5. Strictness Flags (know what each buys you)
| Flag | Effect |
|---|---|
| `strict` | umbrella for all below |
| `noImplicitAny` | disallow inferred `any` |
| `strictNullChecks` | `null`/`undefined` are distinct, not assignable everywhere |
| `strictFunctionTypes` | contravariant checking of function param types |
| `strictBindCallApply` | typed `.bind/.call/.apply` |
| `strictPropertyInitialization` | class fields must be initialized or marked `!`/optional |
| `noUncheckedIndexedAccess` | `arr[i]` returns `T | undefined` — **turn this on**, catches real bugs |
| `exactOptionalPropertyTypes` | `{ a?: string }` disallows explicit `a: undefined` |
| `noImplicitOverride` | requires `override` keyword when overriding base methods |
| `noPropertyAccessFromIndexSignature` | forces `obj["key"]` over `obj.key` for index-signature types |

Senior-level opinion: enable `strict` + `noUncheckedIndexedAccess` + `noImplicitOverride` on every new project.

---

## 6. Classes — Advanced
```ts
class Base {
  #private = 1;                       // true JS private (runtime enforced)
  protected readonly id: string;
  static instances = 0;

  constructor(id: string) { this.id = id; Base.instances++; }

  protected abstract validate(): boolean; // only inside abstract class
}

abstract class Shape {
  abstract area(): number;
}

class Circle extends Shape {
  constructor(private radius: number) { super(); }
  override area() { return Math.PI * this.radius ** 2; } // `override` + noImplicitOverride
}

// Mixins (composable class factories)
type Constructor<T = {}> = new (...args: any[]) => T;
function Timestamped<TBase extends Constructor>(Base: TBase) {
  return class extends Base { timestamp = Date.now(); };
}
class Doc {}
class TimestampedDoc extends Timestamped(Doc) {}
```
Decorators (stage-3 standard decorators, TS 5.0+, no `experimentalDecorators` needed):
```ts
function logged(target: any, ctx: ClassMethodDecoratorContext) {
  return function (this: any, ...args: any[]) {
    console.log(`calling ${String(ctx.name)}`);
    return target.call(this, ...args);
  };
}
class Service {
  @logged
  fetch() { /* ... */ }
}
```

---

## 7. Module System & Declaration Files
- `.d.ts` files: type-only, no runtime code. `declare module "x" { ... }` for ambient/untyped libs.
- `export =` / `import ... = require(...)` for CommonJS interop; prefer ESM `export`/`import` for new code.
- **Module augmentation** (extend a third-party type):
```ts
declare module "express" {
  interface Request { user?: { id: string } }
}
```
- **Global augmentation**:
```ts
declare global {
  interface Window { myGlobal: string }
}
export {}; // needed to make file a module
```
- **Triple-slash directives** (`/// <reference types="node" />`) — legacy, avoid in modern bundler setups.
- **Project references** (`tsconfig` `"references"`) — for monorepos, enables incremental builds and enforces build order between packages.
```jsonc
// tsconfig.json (root)
{ "references": [{ "path": "./packages/core" }, { "path": "./packages/app" }] }
```
- **Path aliases** (`compilerOptions.paths`) — remember to mirror them in the bundler config (Vite/webpack) since `tsc` doesn't rewrite import paths itself.

---

## 8. Utility Types — Full Reference
```ts
Partial<T> / Required<T> / Readonly<T>
Pick<T, K> / Omit<T, K>
Record<K, V>
Exclude<T, U> / Extract<T, U>
NonNullable<T>
Parameters<T> / ConstructorParameters<T>
ReturnType<T> / InstanceType<T>
Awaited<T>                         // unwraps nested Promises
ThisParameterType<T> / OmitThisParameter<T>
Uppercase/Lowercase/Capitalize/Uncapitalize<S>
```
Writing your own (senior interviews love these):
```ts
type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;
type DeepReadonly<T> = T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } : T;
type UnionToIntersection<U> = (U extends any ? (k: U) => void : never) extends (k: infer I) => void ? I : never;
```

---

## 9. Error Handling & Type Safety Patterns
```ts
// Result/Either pattern instead of throwing (functional error handling)
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function tryParse(json: string): Result<unknown> {
  try { return { ok: true, value: JSON.parse(json) }; }
  catch (e) { return { ok: false, error: e instanceof Error ? e : new Error(String(e)) }; }
}

// catch clause variables are `unknown` under strict/useUnknownInCatchVariables
try { risky(); } catch (e) {
  if (e instanceof MyError) handle(e);
}
```
`unknown` > `any` everywhere at API boundaries (parsing JSON, catch blocks, third-party callbacks). Pair with runtime validators (`zod`, `io-ts`, `valibot`) for actual boundary safety — **TS types don't exist at runtime, so trusting `unknown as MyType` without validation is a false sense of security**.

---

## 10. Performance & Tooling
- **Type-checking is separate from transpilation** — Babel/`esbuild`/`swc` strip types without checking; run `tsc --noEmit` in CI as the real gate.
- **`skipLibCheck: true`** — speeds up builds significantly by not re-checking `.d.ts` in `node_modules`.
- **Incremental builds**: `"incremental": true` + `.tsbuildinfo` cache; combine with project references for large monorepos.
- **Avoid deeply recursive conditional/mapped types** on large unions — can blow past the compiler's recursion limit or slow the IDE (`tsserver`) noticeably.
- **Prefer `interface` for hot-path object shapes** — historically cheaper for the checker to cache/extend than large `type` intersections.
- **`import type` / `export type`** (with `verbatimModuleSyntax` or `isolatedModules`) — ensures type-only imports are elided from output, critical for transpile-only pipelines (esbuild/SWC) that can't do cross-file type analysis.

---

## 11. Design Patterns in TS
```ts
// Branded / nominal types (fake nominal typing on top of structural system)
type UserId = string & { readonly __brand: "UserId" };
function toUserId(s: string): UserId { return s as UserId; }

// Builder pattern with fluent generics
class QueryBuilder<T> {
  private filters: Partial<T> = {};
  where<K extends keyof T>(key: K, value: T[K]): this {
    this.filters[key] = value;
    return this;
  }
}

// Exhaustive factory via discriminated union + Record
type Shape = { kind: "circle"; r: number } | { kind: "square"; s: number };
const areaOf: Record<Shape["kind"], (s: any) => number> = {
  circle: (s: { r: number }) => Math.PI * s.r ** 2,
  square: (s: { s: number }) => s.s ** 2,
};
```

---

## 12. React + TS (senior-level specifics)
```tsx
// Generic component
function List<T>({ items, render }: { items: T[]; render: (item: T) => React.ReactNode }) {
  return <ul>{items.map((i, idx) => <li key={idx}>{render(i)}</li>)}</ul>;
}

// Polymorphic "as" prop component
type PolyProps<E extends React.ElementType> = { as?: E } & React.ComponentPropsWithoutRef<E>;
function Box<E extends React.ElementType = "div">({ as, ...rest }: PolyProps<E>) {
  const Component = as || "div";
  return <Component {...rest} />;
}

// Discriminated union props (mutually exclusive prop sets)
type ButtonProps =
  | { variant: "link"; href: string }
  | { variant: "button"; onClick: () => void };

// Strongly typed reducer
type Action = { type: "increment" } | { type: "set"; payload: number };
function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "increment": return state + 1;
    case "set": return action.payload;
  }
}
```
- `ComponentPropsWithoutRef<"button">` / `ComponentPropsWithRef` to extend native element props correctly.
- `forwardRef<HTMLButtonElement, Props>` for ref-forwarding components — type both generic args.

---

## 13. Testing & Type-Level Testing
```ts
// Compile-time type assertions (no runtime cost) — common in libraries
type Expect<T extends true> = T;
type Equal<A, B> = (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;
type _test1 = Expect<Equal<ReturnType<typeof add>, number>>;
```
Or use `tsd` / `expect-type` libraries for this in real test suites (common in library/SDK codebases).

---

## 14. Common Senior-Level Pitfalls to Avoid
- Overusing `as` casts instead of narrowing/validating — casts lie to the compiler, they don't check anything.
- Excess `any` creeping in via untyped 3rd-party code — wrap at the boundary with a typed adapter function instead of leaking `any` through the app.
- Enum overuse where string literal unions are safer, tree-shakeable, and serialize better.
- Ignoring `noUncheckedIndexedAccess` — array/object index access silently assumed non-undefined is a top real-world bug source.
- Writing overly clever recursive conditional types where a simpler explicit type would be more maintainable — optimize for team readability, not type-golf.
- Not distinguishing `interface` (extendable, mergeable) vs `type` (unions/mapped/conditional) intentionally.

---

**Quick self-check for "am I senior-ready":** you should be able to write `DeepPartial<T>`, explain why `noUncheckedIndexedAccess` matters, design a discriminated-union API, know when `unknown` beats `any`, and set up project references for a monorepo — without looking anything up.
