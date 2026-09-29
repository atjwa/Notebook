# TypeScript Interview Study Guide: Mid–Senior Level

## 1. Core type-system concepts

Be able to explain the difference between:

- **Type annotations**: `let id: string`
- **Type inference**: `let id = "abc"`
- **Structural typing**: compatibility is based on shape, not explicit inheritance.
- **Union types**: `string | number`
- **Intersection types**: `A & B`
- **Literal types**: `"pending" | "success" | "error"`
- **`unknown`**: safe top type; requires narrowing before use.
- **`any`**: disables type checking and should be isolated.
- **`never`**: represents impossible values or functions that never return.
- **`void`**: commonly indicates a function’s return value is ignored.

```ts
function parse(value: unknown): string {
  if (typeof value === "string") return value;
  throw new Error("Expected a string");
}
```

### Key interview point

`unknown` is safer than `any`:

```ts
const unsafe: any = "hello";
unsafe.nonexistent(); // Compiles

const safe: unknown = "hello";
// safe.nonexistent(); // Error
```

## 2. Narrowing and type guards

Know how TypeScript narrows types using:

- `typeof`
- `instanceof`
- `in`
- Equality checks
- Discriminated unions
- User-defined type predicates
- Assertion functions

```ts
type Result =
  | { status: "success"; data: string }
  | { status: "error"; message: string };

function handle(result: Result) {
  if (result.status === "success") {
    return result.data;
  }

  return result.message;
}
```

Custom type guard:

```ts
function isString(value: unknown): value is string {
  return typeof value === "string";
}
```

Exhaustiveness checking:

```ts
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

function format(result: Result) {
  switch (result.status) {
    case "success":
      return result.data;
    case "error":
      return result.message;
    default:
      return assertNever(result);
  }
}
```

## 3. Generics

Understand generic functions, constraints, defaults, inference, and generic classes.

```ts
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

Important concepts:

- Generic parameters preserve relationships between inputs and outputs.
- `extends` constrains a generic; it does not mean classical inheritance only.
- Prefer generics over `any` when the relationship between values matters.
- Avoid unnecessary generic parameters that do not appear in inputs or outputs.

```ts
function identity<T>(value: T): T {
  return value;
}

function logLength<T extends { length: number }>(value: T): T {
  console.log(value.length);
  return value;
}
```

Be ready to discuss variance and assignability, especially with function parameters, callbacks, and mutable collections.

## 4. Utility types

Know how to use and implement the common built-ins:

```ts
Partial<T>
Required<T>
Readonly<T>
Pick<T, K>
Omit<T, K>
Record<K, T>
Exclude<T, U>
Extract<T, U>
NonNullable<T>
ReturnType<T>
Parameters<T>
Awaited<T>
InstanceType<T>
```

Examples:

```ts
type User = {
  id: string;
  name: string;
  email: string;
};

type UserUpdate = Partial<Omit<User, "id">>;
type UserMap = Record<string, User>;
```

Be able to implement simplified versions:

```ts
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

type MyReadonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

## 5. Mapped, conditional, and template literal types

Mapped type:

```ts
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};
```

Conditional type:

```ts
type ElementType<T> = T extends readonly (infer U)[] ? U : T;
```

Distributive conditional types:

```ts
type ToArray<T> = T extends unknown ? T[] : never;

type Result = ToArray<string | number>;
// string[] | number[]
```

Template literal types:

```ts
type EventName = `on${Capitalize<"click" | "focus">}`;
// "onClick" | "onFocus"
```

Interview topics include:

- `infer`
- Distributive conditionals
- Recursive types
- Key remapping with `as`
- Compiler performance implications of very complex types

## 6. `keyof`, indexed access, and `typeof`

```ts
type Person = {
  name: string;
  age: number;
};

type PersonKey = keyof Person;       // "name" | "age"
type PersonValue = Person[PersonKey]; // string | number
```

Value-to-type derivation:

```ts
const statuses = ["pending", "success", "error"] as const;

type Status = typeof statuses[number];
// "pending" | "success" | "error"
```

This pattern is especially useful for deriving types from configuration objects and constants.

## 7. Interfaces versus type aliases

Both can describe object shapes, but understand the practical differences.

### Interfaces

- Support declaration merging.
- Work naturally with class implementation.
- Are often preferred for public object-oriented APIs.

```ts
interface User {
  id: string;
}
```

### Type aliases

- Support unions, tuples, mapped types, and conditional types.
- Are generally more expressive.

```ts
type Status = "idle" | "loading" | "complete";
```

Avoid presenting this as an absolute rule. Team conventions, library APIs, declaration merging, and required type features should guide the choice.

## 8. Classes, inheritance, and modifiers

Know:

- `public`, `private`, and `protected`
- `readonly`
- Parameter properties
- Abstract classes
- Method overriding
- `implements` versus `extends`
- Static members
- Accessors
- Initialization behavior

```ts
abstract class Repository<T> {
  abstract findById(id: string): Promise<T | undefined>;
}

class UserRepository extends Repository<User> {
  async findById(id: string) {
    return undefined;
  }
}
```

Important distinction:

- `extends` establishes implementation inheritance.
- `implements` checks that a class satisfies a shape; it does not provide implementation.

## 9. Functions and callback typing

Understand:

- Optional and default parameters
- Rest parameters
- Overloads
- Function types
- Generic callbacks
- `this` typing
- `strictFunctionTypes`

Overloads:

```ts
function parseValue(value: string): string[];
function parseValue(value: string[]): string[];
function parseValue(value: string | string[]) {
  return Array.isArray(value) ? value : value.split(",");
}
```

Use overloads when callers need different precisely typed call signatures. Use unions when a single signature is clearer.

## 10. Async TypeScript

Know the types of:

```ts
Promise<string>
Promise<void>
Promise<Result>
Promise<Result | undefined>
```

Understand:

- `async` functions always return a `Promise`.
- `await` unwraps the fulfilled value.
- Errors are not represented in the standard return type.
- `Promise.all` preserves tuple types for tuple inputs.
- `Promise.allSettled` returns fulfillment/rejection results.
- Cancellation generally requires `AbortController`.

```ts
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/users/${id}`);

  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`);
  }

  return response.json() as Promise<User>;
}
```

Strong candidates mention that a type assertion does not validate runtime data. Use a schema validator when external input must be trusted.

## 11. Runtime validation

TypeScript types are erased at runtime. Data from these sources is untrusted:

- HTTP responses
- JSON
- User input
- Environment variables
- Files
- Local storage
- Database results

Be prepared to explain the difference between:

```ts
const user = data as User;
```

and actual validation:

```ts
function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) return false;

  const obj = value as Record<string, unknown>;
  return typeof obj.id === "string" && typeof obj.name === "string";
}
```

A senior-level answer should discuss schema validation libraries, error reporting, boundary validation, and keeping validated data typed internally.

## 12. Modules and project structure

Know:

- Named versus default exports
- ESM versus CommonJS
- `import type`
- Dynamic imports
- Module resolution
- Barrel files
- Circular dependencies
- Tree shaking
- Declaration files

```ts
import type { User } from "./models.js";
```

`import type` communicates that an import is used only by the type system and can help avoid runtime import issues.

Be familiar with:

- `package.json` module settings
- `tsconfig.json`
- `paths` and path aliases
- Project references
- Monorepos
- Generated declarations
- `module` and `moduleResolution`

## 13. `tsconfig` settings

Know the purpose and tradeoffs of:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noEmit": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

Especially understand:

- `strict`
- `strictNullChecks`
- `noImplicitAny`
- `noUncheckedIndexedAccess`
- `exactOptionalPropertyTypes`
- `useUnknownInCatchVariables`
- `noImplicitOverride`
- `isolatedModules`
- `verbatimModuleSyntax`
- `skipLibCheck`

Senior candidates should be able to explain why a team might enable or disable a setting rather than merely listing its name.

## 14. Nullability and optional properties

With strict null checking:

```ts
function greet(name?: string) {
  // name is string | undefined
  return name?.toUpperCase() ?? "Anonymous";
}
```

Understand the difference between:

```ts
type A = { value?: string };
type B = { value: string | undefined };
```

An optional property may be absent. A required property whose value is `undefined` must still exist. This distinction matters with `exactOptionalPropertyTypes`.

Avoid non-null assertions unless the invariant is genuinely guaranteed:

```ts
element!.textContent = "Done";
```

Prefer explicit checks where practical.

## 15. Common pitfalls

Be ready to identify and fix:

### Excess property checks

```ts
type Config = { timeout: number };

const config: Config = {
  timeout: 1000,
  retries: 3 // Error in this context
};
```

The behavior differs when assigning through an intermediate variable, so do not treat it as a general “extra properties are forbidden” rule.

### Type assertions

Assertions change compile-time checking but do not transform or validate values.

### Enums

Know the tradeoffs of numeric enums, string enums, `const enum`, and union literals. Many teams prefer:

```ts
const roles = ["admin", "editor", "viewer"] as const;
type Role = typeof roles[number];
```

### Arrays and mutability

```ts
readonly string[]
ReadonlyArray<string>
readonly [string, number]
```

Readonly types prevent mutation through that reference; they do not necessarily make nested data deeply immutable.

### Object spread

Object spread creates a shallow copy and may affect inferred types in ways that require careful checking.

### `as const`

It narrows values and adds readonly behavior:

```ts
const config = {
  mode: "production",
  retries: 3
} as const;
```

## 16. Testing and tooling

Know how TypeScript integrates with:

- Unit tests
- Type-level tests
- ESLint
- Formatting
- Build tools
- Bundlers
- Test runners
- Runtime execution
- Source maps

Distinguish:

- Type-checking
- Transpilation
- Bundling
- Runtime validation
- Test execution

A project can transpile successfully while still having type errors if type-checking is not run separately.

## 17. Design and architecture questions

Prepare to discuss:

- How to type an API client
- How to model loading, success, and error states
- How to type configuration objects
- How to design reusable generic utilities
- How to type event emitters
- How to migrate a JavaScript codebase incrementally
- How to handle third-party libraries with poor types
- How to avoid leaking implementation types from public APIs
- How to model domain types separately from transport DTOs
- Where to validate external data
- How to balance type safety with developer productivity

A useful state model:

```ts
type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };
```

This is usually safer than several independent booleans such as `isLoading`, `hasError`, and `hasData`.

## 18. Coding exercises to practice

Practice implementing:

1. `debounce` and `throttle` with preserved parameter and return types.
2. A typed `EventEmitter`.
3. `groupBy<T, K extends PropertyKey>`.
4. A deep `Readonly<T>`.
5. A type-safe `get(obj, path)`.
6. A function that unwraps nested promises.
7. A discriminated-union reducer.
8. A generic API client.
9. A runtime validator for a nested object.
10. A typed cache with expiration.
11. A `Result<T, E>` abstraction.
12. A type-safe object mapper.

Example:

```ts
function groupBy<T, K extends PropertyKey>(
  items: T[],
  getKey: (item: T) => K
): Record<K, T[]> {
  return items.reduce((groups, item) => {
    const key = getKey(item);
    (groups[key] ??= []).push(item);
    return groups;
  }, {} as Record<K, T[]>);
}
```

Be prepared to discuss whether the assertion is fully sound and how you would improve the implementation if keys can be symbols or if a null-prototype object is desirable.

## 19. Questions commonly asked in interviews

- What is structural typing?
- When would you use `unknown` instead of `any`?
- What is the difference between `never`, `void`, and `undefined`?
- How does type narrowing work?
- What are discriminated unions?
- What does `keyof` produce?
- How does `T[K]` work?
- What is the difference between a union and an intersection?
- How do conditional types distribute over unions?
- What does `infer` do?
- When should you use overloads?
- How do interfaces differ from type aliases?
- What does `as const` do?
- What is the difference between `readonly` and deep immutability?
- Why does TypeScript not validate JSON automatically?
- What does `strict` enable?
- What problems can path aliases cause?
- What is declaration merging?
- What does `implements` actually check?
- How would you type a function that preserves its input type?
- How would you safely model an API response?
- How would you migrate an untyped JavaScript application?

## 20. A focused preparation plan

| Time | Focus |
|---|---|
| Days 1–2 | Inference, unions, intersections, narrowing, nullability |
| Days 3–4 | Generics, `keyof`, indexed access, utility types |
| Days 5–6 | Conditional, mapped, recursive, and template literal types |
| Day 7 | Functions, overloads, variance, classes |
| Day 8 | Async code, modules, runtime validation |
| Day 9 | `tsconfig`, build systems, testing, project architecture |
| Day 10 | Timed coding exercises and mock interview questions |

For mid-level interviews, prioritize practical typing, narrowing, generics, async code, and debugging compiler errors. For senior interviews, emphasize API design, runtime boundaries, compiler configuration, type-system tradeoffs, maintainability, migration strategy, and explaining when not to use advanced types.
