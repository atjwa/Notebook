# TypeScript Interview Study Guide: Mid–Senior Level

This guide covers the TypeScript concepts most commonly tested in mid- and senior-level interviews, including what they mean, why they matter, and how to use them.

## 1. TypeScript’s type system

TypeScript is a statically typed language that adds compile-time checking to JavaScript. Its types are removed when code is compiled, so TypeScript does not provide runtime validation by itself.

```ts
const username: string = "Ada";
```

The annotation helps the compiler catch mistakes, but at runtime this is still ordinary JavaScript.

### Type inference

TypeScript often determines types automatically:

```ts
const name = "Ada"; // string
const age = 37;     // number
```

You should generally rely on inference when the type is obvious. Add annotations when they improve clarity, define a public API, or prevent an unintended type from being inferred.

```ts
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

### Structural typing

TypeScript uses structural typing. An object is compatible with a type if it has the required structure, regardless of whether it explicitly declares that it implements the type.

```ts
type HasId = {
  id: string;
};

const user = {
  id: "u1",
  name: "Ada"
};

const item: HasId = user; // Valid
```

The object has an `id`, so it satisfies `HasId`.

This differs from nominal typing, where compatibility depends on explicit declarations or class identity.

### Union types

A union means a value can be one of several types:

```ts
let value: string | number;

value = "hello";
value = 42;
```

You may use only operations that are valid for every member of the union until you narrow it.

```ts
function print(value: string | number) {
  if (typeof value === "string") {
    console.log(value.toUpperCase());
  } else {
    console.log(value.toFixed(2));
  }
}
```

### Intersection types

An intersection combines multiple types:

```ts
type Identifiable = {
  id: string;
};

type Timestamped = {
  createdAt: Date;
};

type Entity = Identifiable & Timestamped;
```

An `Entity` must contain both `id` and `createdAt`.

```ts
const entity: Entity = {
  id: "1",
  createdAt: new Date()
};
```

Unions represent alternatives; intersections represent combined requirements.

### Literal types

Literal types represent exact values rather than broad types:

```ts
type Status = "idle" | "loading" | "success" | "error";
```

This is useful for state machines, configuration, actions, and API contracts.

```ts
let status: Status = "idle";
// status = "finished"; // Error
```

### `any`

`any` disables most type checking:

```ts
let value: any = "hello";

value.nonexistentMethod(); // Compiles
value.foo.bar.baz();       // Compiles
```

It is sometimes useful when migrating legacy JavaScript or dealing with an untyped library, but it spreads uncertainty through the codebase.

Prefer narrowing `unknown`, defining a type, or isolating the `any` at a boundary.

### `unknown`

`unknown` means “a value exists, but its type has not been established yet.”

Unlike `any`, it cannot be used without narrowing:

```ts
function print(value: unknown) {
  // value.toUpperCase(); // Error

  if (typeof value === "string") {
    console.log(value.toUpperCase());
  }
}
```

Use `unknown` for data from untrusted sources such as JSON, HTTP responses, or user input.

### `never`

`never` represents a value that cannot exist.

A function that always throws or never finishes returns `never`:

```ts
function fail(message: string): never {
  throw new Error(message);
}
```

`never` is also useful for exhaustive checking:

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; size: number };

function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.size ** 2;
    default:
      return assertNever(shape);
  }
}
```

If another shape is added later but not handled, TypeScript reports an error.

### `void`

`void` generally means that a function’s return value is not intended to be used:

```ts
function logMessage(message: string): void {
  console.log(message);
}
```

A function returning `void` can technically return a value in some callback contexts, but callers are expected to ignore it.

### `null` and `undefined`

With `strictNullChecks`, `null` and `undefined` are separate types.

```ts
let name: string | undefined;
let result: string | null;
```

You must check them before using the value:

```ts
function uppercase(value: string | undefined) {
  if (value === undefined) return "";
  return value.toUpperCase();
}
```

Optional properties are effectively possibly absent:

```ts
type User = {
  name?: string;
};
```

This means `user.name` has type `string | undefined`.

---

## 2. Type narrowing and type guards

Narrowing is the process by which TypeScript removes possible types from a union based on runtime checks.

For example:

```ts
function format(value: string | number) {
  if (typeof value === "string") {
    // value is string here
    return value.toUpperCase();
  }

  // value is number here
  return value.toFixed(2);
}
```

TypeScript analyzes control flow and determines the type at each point.

### `typeof` narrowing

`typeof` works well for JavaScript primitive types:

```ts
function describe(value: unknown): string {
  if (typeof value === "string") {
    return `String: ${value}`;
  }

  if (typeof value === "number") {
    return `Number: ${value}`;
  }

  if (typeof value === "boolean") {
    return `Boolean: ${value}`;
  }

  if (typeof value === "bigint") {
    return `BigInt: ${value}`;
  }

  if (typeof value === "symbol") {
    return "Symbol";
  }

  if (typeof value === "function") {
    return "Function";
  }

  if (typeof value === "object" && value !== null) {
    return "Object";
  }

  return "Null or undefined";
}
```

Important detail: `typeof null` is `"object"`, so you must check for `null` separately.

```ts
if (typeof value === "object" && value !== null) {
  // value is a non-null object
}
```

### `instanceof` narrowing

`instanceof` checks whether an object was created by a particular class or constructor.

```ts
function formatError(error: unknown): string {
  if (error instanceof Error) {
    return error.message;
  }

  return String(error);
}
```

This is useful with built-in classes and custom classes:

```ts
class NetworkError extends Error {
  constructor(public statusCode: number) {
    super("Network request failed");
  }
}

function handle(error: unknown) {
  if (error instanceof NetworkError) {
    console.log(error.statusCode);
  }
}
```

`instanceof` is not always reliable across different JavaScript realms, such as iframes or certain package-boundary scenarios. For plain objects, structural checks are usually more appropriate.

### The `in` operator

The `in` operator checks whether a property exists on an object. TypeScript can use that check to narrow a union.

```ts
type Bird = {
  fly(): void;
};

type Fish = {
  swim(): void;
};

function move(animal: Bird | Fish) {
  if ("fly" in animal) {
    animal.fly();
  } else {
    animal.swim();
  }
}
```

The property does not necessarily need to be callable:

```ts
type Success = {
  data: string;
};

type Failure = {
  error: string;
};

function getMessage(result: Success | Failure) {
  if ("data" in result) {
    return result.data;
  }

  return result.error;
}
```

### Equality narrowing

Equality checks can narrow values:

```ts
function compare(value: string | null) {
  if (value === null) {
    return "No value";
  }

  return value.toUpperCase();
}
```

Comparing two values can also narrow them to a shared type:

```ts
function equal(a: string | number, b: string | boolean) {
  if (a === b) {
    // Both values must be string here
    return a.toUpperCase();
  }

  return false;
}
```

Use strict equality checks such as `===` and `!==` to make behavior predictable.

### Truthiness narrowing

TypeScript can narrow based on truthiness:

```ts
function greet(name?: string) {
  if (name) {
    return `Hello, ${name}`;
  }

  return "Hello, anonymous user";
}
```

However, truthiness also excludes valid falsy values such as:

- `0`
- `""`
- `false`
- `NaN`

This can be incorrect:

```ts
function printCount(count: number | undefined) {
  if (count) {
    console.log(count);
  }
}
```

The code does not print `0`. Prefer an explicit check when zero is valid:

```ts
if (count !== undefined) {
  console.log(count);
}
```

### Discriminated unions

A discriminated union is a union whose members share a property with different literal values.

```ts
type RequestState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: string }
  | { status: "error"; message: string };
```

The `status` property is the discriminator.

```ts
function render(state: RequestState): string {
  switch (state.status) {
    case "idle":
      return "Not started";

    case "loading":
      return "Loading...";

    case "success":
      return state.data;

    case "error":
      return `Error: ${state.message}`;
  }
}
```

Once TypeScript sees `state.status === "success"`, it knows that `data` exists. This is safer than modeling state with unrelated booleans:

```ts
type BadState = {
  isLoading: boolean;
  hasError: boolean;
  data?: string;
  error?: string;
};
```

The `BadState` type permits contradictory combinations, such as both `isLoading` and `hasError` being true.

### User-defined type predicates

A type predicate tells TypeScript that a function returns a boolean that establishes a type.

```ts
function isString(value: unknown): value is string {
  return typeof value === "string";
}
```

Usage:

```ts
function print(value: unknown) {
  if (isString(value)) {
    console.log(value.toUpperCase());
  }
}
```

The syntax is:

```ts
parameterName is Type
```

A more complex example:

```ts
type User = {
  id: string;
  name: string;
};

function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) {
    return false;
  }

  const object = value as Record<string, unknown>;

  return (
    typeof object.id === "string" &&
    typeof object.name === "string"
  );
}
```

A type predicate is trusted by the compiler. If the implementation is wrong, the type system can be fooled:

```ts
function isString(value: unknown): value is string {
  return true; // Dangerous
}
```

Therefore, predicates must perform real checks.

### Assertion functions

An assertion function throws if a condition is not met. Its return type uses `asserts`.

```ts
function assertString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error("Expected a string");
  }
}
```

After calling it, TypeScript treats the value as a string:

```ts
function process(value: unknown) {
  assertString(value);
  value.toUpperCase();
}
```

You can also assert general conditions:

```ts
function assert(condition: unknown, message: string): asserts condition {
  if (!condition) {
    throw new Error(message);
  }
}
```

### Exhaustive narrowing with `never`

For discriminated unions, use `never` to detect missing cases:

```ts
type Event =
  | { type: "created"; id: string }
  | { type: "deleted"; id: string };

function assertNever(value: never): never {
  throw new Error(`Unhandled value: ${JSON.stringify(value)}`);
}

function handleEvent(event: Event) {
  switch (event.type) {
    case "created":
      return `Created ${event.id}`;

    case "deleted":
      return `Deleted ${event.id}`;

    default:
      return assertNever(event);
  }
}
```

If someone adds `{ type: "updated" }` to `Event`, the `default` branch will no longer receive `never`, and compilation will fail.

---

## 3. Generics

Generics allow you to preserve relationships between values while still supporting multiple types.

Without generics:

```ts
function identity(value: any): any {
  return value;
}
```

This loses type information.

With a generic:

```ts
function identity<T>(value: T): T {
  return value;
}

const result = identity("hello");
// result is string
```

The same type `T` is used for the input and output.

### Generic functions

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}

const number = first([1, 2, 3]);     // number | undefined
const word = first(["a", "b", "c"]); // string | undefined
```

### Generic constraints

A constraint limits which types can be passed.

```ts
function logLength<T extends { length: number }>(value: T): T {
  console.log(value.length);
  return value;
}

logLength("hello");
logLength([1, 2, 3]);
// logLength(42); // Error
```

`extends` here means “must be assignable to,” not necessarily class inheritance.

### `keyof` constraints

You can ensure that a property key exists on an object:

```ts
function getProperty<T, K extends keyof T>(object: T, key: K): T[K] {
  return object[key];
}

const user = {
  id: "u1",
  age: 30
};

const id = getProperty(user, "id");   // string
const age = getProperty(user, "age"); // number
// getProperty(user, "name");         // Error
```

This preserves the relationship between the selected key and its value.

### Generic defaults

Generic parameters can have defaults:

```ts
type ApiResponse<T = unknown> = {
  data: T;
  status: number;
};

const response: ApiResponse = {
  data: "anything",
  status: 200
};
```

### Generic classes

```ts
class Box<T> {
  constructor(public value: T) {}

  get(): T {
    return this.value;
  }
}

const stringBox = new Box("hello");
const numberBox = new Box(42);
```

### Generic interfaces

```ts
interface Repository<T> {
  findById(id: string): Promise<T | undefined>;
  save(value: T): Promise<T>;
}
```

### When to use generics

Use generics when:

- The same type relationship should be preserved.
- A function should work with multiple types.
- A collection should retain the type of its elements.
- An API should be reusable without falling back to `any`.

Avoid adding generic parameters that do not affect the input, output, or constraints. Unnecessary generics make APIs harder to understand.

---

## 4. Variance and function assignability

Variance describes how type relationships behave when types are used inside other types.

This becomes important with:

- Callback functions
- Generic containers
- Mutable arrays
- Event handlers
- `strictFunctionTypes`

Consider:

```ts
type Animal = {
  name: string;
};

type Dog = Animal & {
  bark(): void;
};
```

A `Dog` can be used where an `Animal` is expected because every dog is an animal.

However, function parameter types require care:

```ts
function handleAnimal(animal: Animal) {}
function handleDog(dog: Dog) {}
```

A function that can handle every `Animal` is generally safe where a function handling a `Dog` is expected. The reverse may be unsafe because not every animal can bark.

When discussing variance in interviews, emphasize that TypeScript’s assignability rules are designed to prevent unsafe callback usage, but some method and mutable-container behavior can still be surprising.

---

## 5. Utility types

Utility types transform existing types.

### `Partial<T>`

Makes every property optional:

```ts
type User = {
  id: string;
  name: string;
  email: string;
};

type UserUpdate = Partial<User>;
```

Equivalent conceptually to:

```ts
type UserUpdate = {
  id?: string;
  name?: string;
  email?: string;
};
```

Useful for patch operations, but be careful: `Partial<User>` may allow updates that should not be allowed, such as changing an ID.

```ts
type BetterUserUpdate = Partial<Omit<User, "id">>;
```

### `Required<T>`

Makes all properties required:

```ts
type OptionalConfig = {
  timeout?: number;
  retries?: number;
};

type CompleteConfig = Required<OptionalConfig>;
```

### `Readonly<T>`

Prevents assignment through that reference:

```ts
type User = {
  name: string;
};

const user: Readonly<User> = {
  name: "Ada"
};

// user.name = "Grace"; // Error
```

`Readonly` is shallow. Nested objects can still be mutable unless recursively made readonly.

### `Pick<T, K>`

Selects a subset of properties:

```ts
type UserSummary = Pick<User, "id" | "name">;
```

### `Omit<T, K>`

Removes properties:

```ts
type PublicUser = Omit<User, "email">;
```

### `Record<K, T>`

Creates an object type with a set of keys and a common value type:

```ts
type UserMap = Record<string, User>;

const users: UserMap = {
  u1: {
    id: "u1",
    name: "Ada",
    email: "ada@example.com"
  }
};
```

It is also useful with literal unions:

```ts
type Status = "idle" | "loading" | "success" | "error";

const statusMessages: Record<Status, string> = {
  idle: "Not started",
  loading: "Loading",
  success: "Complete",
  error: "Failed"
};
```

This ensures every status has a message.

### `Exclude<T, U>`

Removes members assignable to `U`:

```ts
type All = "id" | "name" | "email";
type PublicFields = Exclude<All, "email">;
// "id" | "name"
```

### `Extract<T, U>`

Keeps members assignable to `U`:

```ts
type Values = string | number | boolean;
type StringOrNumber = Extract<Values, string | number>;
// string | number
```

### `NonNullable<T>`

Removes `null` and `undefined`:

```ts
type MaybeString = string | null | undefined;
type StringOnly = NonNullable<MaybeString>;
// string
```

### `ReturnType<T>`

Gets a function’s return type:

```ts
function createUser() {
  return {
    id: "u1",
    name: "Ada"
  };
}

type User = ReturnType<typeof createUser>;
```

### `Parameters<T>`

Gets a function’s parameter tuple:

```ts
function request(url: string, timeout: number) {}

type RequestParameters = Parameters<typeof request>;
// [url: string, timeout: number]
```

### `Awaited<T>`

Gets the resolved value of a promise:

```ts
type ResponseData = Awaited<Promise<{ id: string }>>;
// { id: string }
```

It also handles nested promises.

### Implementing utility types

Many utility types use mapped and conditional types:

```ts
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

type MyPartial<T> = {
  [P in keyof T]?: T[P];
};

type MyReadonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

---

## 6. Mapped types

A mapped type iterates over the keys of another type.

```ts
type User = {
  id: string;
  name: string;
};

type NullableUser = {
  [K in keyof User]: User[K] | null;
};
```

This produces:

```ts
type NullableUser = {
  id: string | null;
  name: string | null;
};
```

### Modifying property modifiers

You can add or remove `readonly` and optional modifiers:

```ts
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type RequiredProperties<T> = {
  [K in keyof T]-?: T[K];
};
```

### Key remapping

You can generate new property names:

```ts
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type User = {
  name: string;
  age: number;
};

type UserGetters = Getters<User>;
// {
//   getName: () => string;
//   getAge: () => number;
// }
```

The `string & K` pattern ensures that `K` can be used with string manipulation utilities.

---

## 7. Conditional types

A conditional type chooses one type or another based on assignability:

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false
```

### `infer`

`infer` extracts a type from another type:

```ts
type ElementType<T> =
  T extends readonly (infer U)[] ? U : T;

type A = ElementType<string[]>; // string
type B = ElementType<number[]>; // number
type C = ElementType<boolean>;   // boolean
```

Another example:

```ts
type PromiseValue<T> =
  T extends Promise<infer U> ? U : T;

type Result = PromiseValue<Promise<string>>;
// string
```

### Distributive conditional types

A conditional type distributes over a union when the checked type is a naked type parameter:

```ts
type ToArray<T> = T extends unknown ? T[] : never;

type Result = ToArray<string | number>;
// string[] | number[]
```

Conceptually, this becomes:

```ts
string[] | number[]
```

To prevent distribution, wrap the type parameter:

```ts
type ToArrayNonDistributive<T> =
  [T] extends [unknown] ? T[] : never;

type Result = ToArrayNonDistributive<string | number>;
// (string | number)[]
```

This distinction is a common senior-level interview topic.

---

## 8. Template literal types

Template literal types combine string literals programmatically:

```ts
type Direction = "top" | "bottom";
type Side = "left" | "right";

type Position = `${Direction}-${Side}`;
// "top-left" | "top-right" | "bottom-left" | "bottom-right"
```

Built-in string helpers include:

```ts
Uppercase<"hello">      // "HELLO"
Lowercase<"HELLO">      // "hello"
Capitalize<"hello">     // "Hello"
Uncapitalize<"Hello">   // "hello"
```

Example:

```ts
type Event = "click" | "focus";
type HandlerName = `on${Capitalize<Event>}`;
// "onClick" | "onFocus"
```

These are useful for event APIs, route definitions, CSS-like keys, and generated method names. They can become difficult to maintain if overused.

---

## 9. `keyof`, indexed access, and `typeof`

### `keyof`

`keyof` produces a union of property keys:

```ts
type User = {
  id: string;
  name: string;
};

type UserKey = keyof User;
// "id" | "name"
```

For an index signature:

```ts
type Dictionary = {
  [key: string]: number;
};

type Key = keyof Dictionary;
// string | number
```

The `number` exists because JavaScript object keys can be accessed using numeric keys that are converted to strings.

### Indexed access types

Indexed access retrieves a property type:

```ts
type User = {
  id: string;
  age: number;
};

type IdType = User["id"];
// string

type ValueType = User[keyof User];
// string | number
```

For arrays:

```ts
type Names = string[];
type Name = Names[number];
// string
```

### Deriving a union from an array

```ts
const statuses = ["pending", "success", "error"] as const;

type Status = typeof statuses[number];
// "pending" | "success" | "error"
```

Without `as const`, the array would usually be inferred as `string[]`, and the union would become just `string`.

### Deriving a type from an object

```ts
const config = {
  retries: 3,
  mode: "production"
} as const;

type Config = typeof config;
```

The resulting type preserves literal values:

```ts
{
  readonly retries: 3;
  readonly mode: "production";
}
```

---

## 10. Interfaces versus type aliases

### Interfaces

Interfaces describe object shapes:

```ts
interface User {
  id: string;
  name: string;
}
```

They support declaration merging:

```ts
interface User {
  email: string;
}
```

The resulting `User` requires all three properties.

Interfaces are commonly used for public object-oriented APIs and class contracts.

### Type aliases

Type aliases can describe objects:

```ts
type User = {
  id: string;
  name: string;
};
```

They can also describe unions, tuples, primitives, conditional types, and mapped types:

```ts
type ID = string | number;
type Point = [number, number];
type Status = "idle" | "busy";
```

### `implements`

`implements` checks whether a class satisfies a type:

```ts
interface Serializable {
  serialize(): string;
}

class User implements Serializable {
  serialize() {
    return "user";
  }
}
```

`implements` does not copy methods or change the class’s inferred types. It only checks compatibility.

### `extends`

`extends` creates inheritance:

```ts
class Admin extends User {
  deleteUser() {}
}
```

A class can implement multiple interfaces but extend only one class.

---

## 11. Functions, overloads, and callbacks

### Function types

```ts
type Formatter = (value: string) => string;

const uppercase: Formatter = value => value.toUpperCase();
```

### Optional and default parameters

```ts
function greet(name?: string) {
  return `Hello, ${name ?? "anonymous"}`;
}
```

An optional parameter has a type that includes `undefined`.

```ts
function connect(timeout = 5000) {
  // timeout is number
}
```

### Rest parameters

```ts
function sum(...values: number[]): number {
  return values.reduce((total, value) => total + value, 0);
}
```

### Function overloads

Overloads provide multiple call signatures with one implementation:

```ts
function parse(value: string): string[];
function parse(value: string[]): string[];

function parse(value: string | string[]): string[] {
  return Array.isArray(value) ? value : value.split(",");
}
```

Callers receive precise types:

```ts
const a = parse("a,b");       // string[]
const b = parse(["a", "b"]);  // string[]
```

The implementation signature is not directly visible to callers. It must be compatible with every overload.

Use overloads when different input forms produce meaningfully different typed results. Use a union when one signature is simpler and sufficiently precise.

### Generic callbacks

```ts
function mapValues<T, U>(
  values: T[],
  callback: (value: T) => U
): U[] {
  return values.map(callback);
}
```

### `this` parameters

TypeScript supports an explicit fake first parameter for typing `this`:

```ts
function printName(this: { name: string }) {
  console.log(this.name);
}
```

The `this` parameter is removed from the actual JavaScript function parameters.

---

## 12. Async TypeScript

An `async` function always returns a promise.

```ts
async function getName(): Promise<string> {
  return "Ada";
}
```

Even though the function returns a string internally, callers receive `Promise<string>`.

### `await`

```ts
const name = await getName();
// name is string
```

`await` unwraps the fulfilled value of a promise.

### Error handling

Promise return types do not express rejected errors:

```ts
async function loadUser(): Promise<User> {
  throw new Error("Failed");
}
```

The type says the successful result is `User`; it does not describe every possible thrown error.

Use `try/catch`:

```ts
async function load() {
  try {
    return await loadUser();
  } catch (error: unknown) {
    if (error instanceof Error) {
      console.error(error.message);
    }
  }
}
```

### `Promise.all`

With a tuple of promises, TypeScript preserves each result type:

```ts
const [user, settings] = await Promise.all([
  getUser(),
  getSettings()
]);
```

If `getUser()` returns `Promise<User>` and `getSettings()` returns `Promise<Settings>`, the results are typed as `[User, Settings]`.

### `Promise.allSettled`

`Promise.allSettled` waits for every promise, including rejected ones:

```ts
const results = await Promise.allSettled([
  getUser(),
  getSettings()
]);

for (const result of results) {
  if (result.status === "fulfilled") {
    console.log(result.value);
  } else {
    console.error(result.reason);
  }
}
```

The `status` property is a discriminant.

### `Promise.race` and `Promise.any`

- `Promise.race` settles when the first promise settles.
- `Promise.any` fulfills when the first promise fulfills, or rejects with an aggregate error if all reject.

### Runtime validation of responses

This is unsafe by itself:

```ts
const user = await response.json() as User;
```

The assertion does not inspect the data. A malformed response still passes through.

A safer approach is to validate at the boundary:

```ts
function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) {
    return false;
  }

  const object = value as Record<string, unknown>;

  return (
    typeof object.id === "string" &&
    typeof object.name === "string"
  );
}
```

After validation, the rest of the application can use the typed value confidently.

---

## 13. Runtime validation

TypeScript cannot validate values that arrive at runtime.

Potentially unsafe inputs include:

- JSON
- HTTP responses
- Environment variables
- Form values
- Local storage
- Database results
- Message queues
- Third-party integrations

### Type assertion versus validation

A type assertion:

```ts
const user = value as User;
```

means “tell the compiler to treat this value as `User`.” It does not perform a runtime check.

A type guard validates:

```ts
function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) {
    return false;
  }

  const object = value as Record<string, unknown>;

  return (
    typeof object.id === "string" &&
    typeof object.name === "string"
  );
}
```

Use validation at system boundaries, then use strongly typed values internally.

### Schema validation

For complex data, manually written guards can become verbose. Schema validation libraries can define a runtime schema and derive or produce a corresponding TypeScript type.

A senior-level answer should mention:

- Validate external data once near the boundary.
- Return useful validation errors.
- Do not trust type assertions as validation.
- Keep transport DTOs separate from internal domain models when their shapes differ.

---

## 14. Classes and object-oriented TypeScript

### Access modifiers

```ts
class Account {
  public owner: string;
  private balance = 0;
  protected accountType = "standard";

  constructor(owner: string) {
    this.owner = owner;
  }

  deposit(amount: number) {
    this.balance += amount;
  }
}
```

- `public`: accessible everywhere.
- `private`: accessible only inside the declaring class.
- `protected`: accessible inside the class and subclasses.
- `readonly`: cannot be reassigned after initialization.

TypeScript’s `private` is primarily a compile-time restriction. JavaScript’s `#privateField` provides runtime private fields.

### Parameter properties

```ts
class User {
  constructor(
    public readonly id: string,
    private email: string
  ) {}
}
```

This creates and initializes the properties automatically.

### Abstract classes

```ts
abstract class Repository<T> {
  abstract findById(id: string): Promise<T | undefined>;

  log(message: string) {
    console.log(message);
  }
}
```

Subclasses must implement abstract members:

```ts
class UserRepository extends Repository<User> {
  async findById(id: string): Promise<User | undefined> {
    return undefined;
  }
}
```

### Method overriding

```ts
class Base {
  save() {}
}

class Derived extends Base {
  override save() {}
}
```

The `override` keyword makes the intent explicit and helps catch renamed or removed base methods.

---

## 15. Modules and project structure

### Named exports

```ts
export function createUser() {}
export type User = {};
```

```ts
import { createUser, type User } from "./users";
```

### Default exports

```ts
export default function createUser() {}
```

Default imports can be renamed arbitrarily:

```ts
import makeUser from "./users";
```

Named exports are often easier to search for and refactor, while default exports may be convenient for a module with one primary export.

### Type-only imports

```ts
import type { User } from "./models";
```

This communicates that `User` is needed only for type checking, not at runtime. It helps avoid certain runtime import and circular dependency problems.

### Dynamic imports

```ts
const module = await import("./heavy-module");
module.start();
```

Dynamic imports can support lazy loading and code splitting.

### ESM and CommonJS

TypeScript can target different module systems. Important settings include:

- `module`
- `moduleResolution`
- `target`
- Package metadata such as `"type": "module"`

Many module problems come from mixing ESM and CommonJS expectations. Be prepared to explain how the compiler, package configuration, runtime, and bundler interact.

### Barrel files

A barrel file re-exports from multiple modules:

```ts
export * from "./user";
export * from "./account";
```

Barrels can simplify imports but may:

- Create circular dependencies.
- Increase module-loading complexity.
- Make tree shaking less predictable in some setups.

### Declaration files

A `.d.ts` file describes types without implementation:

```ts
declare module "legacy-library" {
  export function parse(value: string): unknown;
}
```

Declaration files are useful for JavaScript libraries that do not ship their own types.

---

## 16. `tsconfig.json`

A strict project commonly starts with:

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
    "noImplicitOverride": true,
    "useUnknownInCatchVariables": true,
    "noEmit": true
  }
}
```

### `strict`

Enables a group of strict checking options. It is strongly recommended for new projects.

### `noImplicitAny`

Reports locations where TypeScript would otherwise infer `any`:

```ts
function greet(name) {
  return `Hello ${name}`;
}
```

With `noImplicitAny`, `name` must be typed.

### `strictNullChecks`

Treats `null` and `undefined` as distinct types instead of allowing them everywhere.

### `noUncheckedIndexedAccess`

Adds `undefined` to indexed access:

```ts
const users: User[] = [];
const user = users[0];
// User | undefined
```

This reflects the fact that the index may not exist.

### `exactOptionalPropertyTypes`

Distinguishes an absent optional property from a property explicitly set to `undefined`.

```ts
type Config = {
  timeout?: number;
};
```

With this option, `timeout` may be absent, but assigning `timeout: undefined` may not be equivalent unless `undefined` is explicitly included.

### `noUnusedLocals` and `noUnusedParameters`

Detect unused declarations and parameters. These improve maintainability but may need exceptions for framework-specific code.

### `noImplicitOverride`

Requires the `override` keyword when a subclass overrides a base class member.

### `useUnknownInCatchVariables`

Treats caught errors as `unknown`:

```ts
try {
  riskyOperation();
} catch (error) {
  if (error instanceof Error) {
    console.log(error.message);
  }
}
```

This is safer because JavaScript allows anything to be thrown.

### `isolatedModules`

Ensures files can be transpiled independently. This is important when using transpilers that do not perform full type analysis.

### `skipLibCheck`

Skips checking declaration files in dependencies. It can improve build performance but may hide type problems in external packages.

### `noEmit`

Runs type checking without generating JavaScript. Useful when another tool handles transpilation.

---

## 17. Nullability and optional properties

### Optional properties

```ts
type User = {
  id: string;
  nickname?: string;
};
```

A `User` may not have a `nickname`.

```ts
function display(user: User) {
  return user.nickname?.toUpperCase() ?? user.id;
}
```

### Optional chaining

```ts
const city = user.address?.city;
```

If `address` is `null` or `undefined`, the expression returns `undefined` rather than throwing.

### Nullish coalescing

```ts
const timeout = config.timeout ?? 5000;
```

`??` uses the fallback only for `null` or `undefined`.

This differs from `||`:

```ts
const count = value || 10;
```

`||` also replaces valid falsy values such as `0`, `false`, and `""`.

### Non-null assertion

```ts
const element = document.getElementById("app")!;
```

The `!` tells TypeScript that the value is not null. It does not perform a runtime check. If the assumption is wrong, the program can still fail.

Use it only when the invariant is genuinely guaranteed.

---

## 18. Arrays, tuples, and readonly types

### Arrays

```ts
const names: string[] = ["Ada", "Grace"];
```

Equivalent syntax:

```ts
const names: Array<string> = ["Ada", "Grace"];
```

### Tuples

Tuples represent fixed positions and types:

```ts
const point: [number, number] = [10, 20];
```

Named tuple elements improve readability:

```ts
type Point = [x: number, y: number];
```

Optional and rest tuple elements are supported:

```ts
type Command = [name: string, ...args: string[]];
```

### Readonly arrays

```ts
function total(values: readonly number[]): number {
  return values.reduce((sum, value) => sum + value, 0);
}
```

The function promises not to mutate the input.

### `as const`

```ts
const colors = ["red", "green", "blue"] as const;
```

This produces a readonly tuple of literal values:

```ts
readonly ["red", "green", "blue"]
```

Without `as const`, the type would generally be `string[]`.

### Shallow immutability

```ts
type Config = {
  nested: {
    enabled: boolean;
  };
};

const config: Readonly<Config> = {
  nested: { enabled: true }
};

config.nested.enabled = false; // Allowed by shallow Readonly
```

A deeply immutable type must recursively apply `readonly`:

```ts
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>
    : T[K];
};
```

This simplified version has edge cases for functions, arrays, maps, sets, and special objects.

---

## 19. Common TypeScript pitfalls

### Excess property checks

```ts
type Config = {
  timeout: number;
};

const config: Config = {
  timeout: 1000,
  retries: 3 // Error
};
```

Object literals receive special checking for unexpected properties.

However:

```ts
const raw = {
  timeout: 1000,
  retries: 3
};

const config: Config = raw; // Usually allowed
```

This is because structural typing checks whether `raw` has at least the required properties. It does not generally prohibit additional properties.

### Type assertions do not transform values

```ts
const value = "123" as unknown as number;
```

This does not convert the string to a number. Runtime value remains `"123"`.

Use actual conversion:

```ts
const value = Number("123");
```

### Enums

Enums provide named constants:

```ts
enum Direction {
  Up,
  Down
}
```

String enums are more explicit:

```ts
enum Status {
  Pending = "pending",
  Complete = "complete"
}
```

Many teams prefer literal unions:

```ts
type Status = "pending" | "complete";
```

Literal unions are often simpler to serialize, narrow, and integrate with JavaScript.

### Object spread is shallow

```ts
const copy = {
  ...original
};
```

Nested objects remain shared references:

```ts
copy.nested.value = 10;
// May also modify original.nested.value
```

### `any` leakage

A single `any` can spread through return values, arrays, and function calls. Isolate it at the smallest possible boundary and convert it to a known type quickly.

### Overusing advanced types

A type can be technically impressive but practically harmful if it is:

- Difficult to read.
- Slow for the compiler.
- Hard to debug.
- More complex than the runtime behavior.
- Exposed unnecessarily in public APIs.

Senior engineers should know when to choose a simpler type.

---

## 20. Runtime, compilation, transpilation, and bundling

These are separate concepts:

- **Type checking:** verifies TypeScript types.
- **Transpilation:** converts TypeScript or modern JavaScript into JavaScript.
- **Bundling:** combines modules and optimizes assets.
- **Runtime validation:** checks actual values while the program is running.
- **Testing:** verifies behavior.

A tool may transpile code without type-checking it. For example, a fast build tool may remove TypeScript syntax but leave type errors undetected. Many projects therefore run a separate type-check command in CI.

---

## 21. API and application design

### Model mutually exclusive states

Prefer discriminated unions:

```ts
type LoadState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };
```

This prevents invalid states and makes rendering logic easier to narrow.

### Separate DTOs from domain models

A network response may not match the internal model:

```ts
type UserResponse = {
  user_id: string;
  display_name: string;
};

type User = {
  id: string;
  name: string;
};
```

Convert at the boundary:

```ts
function toUser(response: UserResponse): User {
  return {
    id: response.user_id,
    name: response.display_name
  };
}
```

This keeps external API quirks from spreading through the application.

### Type API clients

```ts
type ApiResponse<T> = {
  data: T;
  status: number;
};

async function get<T>(url: string): Promise<ApiResponse<T>> {
  const response = await fetch(url);
  const data = await response.json();

  return {
    data: data as T,
    status: response.status
  };
}
```

A senior answer should note that the generic assertion still requires runtime validation for untrusted data.

### Use branded types when values need stronger identity

Structural typing means two strings are normally interchangeable:

```ts
type UserId = string;
type OrderId = string;
```

This does not prevent accidental mixing.

A branded type can distinguish them:

```ts
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };

function createUserId(value: string): UserId {
  return value as UserId;
}
```

This improves compile-time separation, but the brand itself does not validate the string’s format.

---

## 22. Type-safe coding patterns

### Type-safe `groupBy`

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

Interview discussion points:

- `K extends PropertyKey` allows strings, numbers, and symbols.
- `Record<K, T[]>` maps each key to an array of values.
- The assertion is needed because the reducer starts with an empty object.
- A null-prototype object may be safer if arbitrary string keys are accepted.

### Type-safe event emitter

```ts
type Events = {
  userCreated: { id: string };
  userDeleted: { id: string };
};

class EventEmitter<E extends Record<string, unknown>> {
  private handlers: {
    [K in keyof E]?: Array<(payload: E[K]) => void>;
  } = {};

  on<K extends keyof E>(
    event: K,
    handler: (payload: E[K]) => void
  ): void {
    const handlers = this.handlers[event] ??= [];
    handlers.push(handler);
  }

  emit<K extends keyof E>(event: K, payload: E[K]): void {
    this.handlers[event]?.forEach(handler => handler(payload));
  }
}

const emitter = new EventEmitter<Events>();

emitter.on("userCreated", payload => {
  payload.id; // string
});

emitter.emit("userCreated", { id: "u1" });
// emitter.emit("userCreated", { wrong: true }); // Error
```

### Result type

Instead of throwing for expected failures:

```ts
type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { ok: false, error: "Cannot divide by zero" };
  }

  return { ok: true, value: a / b };
}
```

Consumers can narrow using `ok`:

```ts
const result = divide(10, 2);

if (result.ok) {
  console.log(result.value);
} else {
  console.error(result.error);
}
```

---

## 23. Testing and tooling

Be able to distinguish:

- Runtime unit tests.
- Integration tests.
- End-to-end tests.
- Compile-time type tests.
- Linting.
- Formatting.
- Type checking.

A function can have correct types and still behave incorrectly. Conversely, a function can behave correctly for current inputs while exposing an unsafe type.

Type-level tests may check that:

- An invalid call produces a compiler error.
- A generic function preserves the expected type.
- A public API does not expose unwanted implementation details.

Also understand how TypeScript integrates with:

- Test runners.
- Linters.
- Formatters.
- Bundlers.
- Source maps.
- Monorepos.
- Project references.
- Generated declaration files.

---

## 24. Senior-level architecture topics

Prepare to discuss:

- How to introduce TypeScript into an existing JavaScript project.
- How to choose strict compiler settings.
- How to handle untyped third-party libraries.
- Where runtime validation belongs.
- How to avoid `any` spreading through a codebase.
- How to model loading, success, and failure states.
- How to separate API DTOs from domain objects.
- How to design public library types.
- How to prevent circular dependencies.
- How to balance type precision against maintainability.
- How to diagnose slow compiler performance.
- How to migrate incrementally without blocking feature work.

A strong migration strategy usually includes:

1. Enable TypeScript incrementally.
2. Start with clear module boundaries.
3. Add strict settings gradually if necessary.
4. Type external boundaries first.
5. Avoid replacing every value with `any`.
6. Add tests while changing types.
7. Use declaration files for untyped dependencies.
8. Increase strictness as the codebase improves.

---

## 25. Common interview questions and strong answers

### What is the difference between `any` and `unknown`?

`any` disables type checking. `unknown` accepts any value but requires narrowing before use. `unknown` is preferable for untrusted data.

### What is the difference between `never` and `void`?

`void` means a function’s return value is not intended to be used. `never` means the function cannot successfully return, usually because it always throws or never terminates.

### What is structural typing?

Types are compatible based on their members rather than explicit declarations. If an object has the required properties, it can satisfy the type.

### What is a discriminated union?

A union whose members share a property with distinct literal values. Checking that property narrows the union and makes invalid states harder to represent.

### What does `keyof` do?

It produces a union of the keys of a type:

```ts
type Keys = keyof { id: string; name: string };
// "id" | "name"
```

### What does `T[K]` mean?

It is indexed access syntax. It retrieves the type of property `K` from `T`:

```ts
type Value<T, K extends keyof T> = T[K];
```

### What is the difference between a union and an intersection?

A union means one of several possibilities. An intersection combines multiple requirements.

```ts
type Input = string | number;
type User = Identifiable & Timestamped;
```

### What does `infer` do?

It extracts a type inside a conditional type:

```ts
type Value<T> = T extends Promise<infer U> ? U : T;
```

### Why is `as User` not validation?

A type assertion changes only the compiler’s view. It does not inspect, convert, or validate the runtime value.

### When would you use an interface instead of a type alias?

Use either based on team conventions. Interfaces are useful for object contracts, class implementation, and declaration merging. Type aliases are required or more convenient for unions, tuples, mapped types, and conditional types.

### What does `as const` do?

It narrows literals and makes properties or array elements readonly:

```ts
const status = "success" as const;
// type: "success"
```

### What does `strictNullChecks` do?

It treats `null` and `undefined` as distinct types, requiring code to handle their possible absence.

### What is the difference between `implements` and `extends`?

`extends` inherits from a class or combines types. `implements` checks that a class satisfies a type but does not provide implementation.

### How would you safely handle an API response?

Validate the unknown response at the boundary, convert it into a known internal type, and keep the rest of the application working with validated values.

### How would you preserve types in a generic function?

Use the same generic parameter in the relevant inputs and outputs:

```ts
function identity<T>(value: T): T {
  return value;
}
```

---

## 26. Coding exercises to practice

Practice implementing the following without relying on `any`:

1. A generic `identity` function.
2. A typed `getProperty` function.
3. `groupBy<T, K extends PropertyKey>`.
4. A type-safe event emitter.
5. `debounce` while preserving parameter types.
6. `throttle` while preserving parameter types.
7. A discriminated-union reducer.
8. A generic API client.
9. A runtime validator for a nested object.
10. A typed cache with expiration.
11. A `Result<T, E>` type.
12. A `DeepReadonly<T>` utility.
13. A function that unwraps nested promises.
14. A type-safe `get` function for object paths.
15. A typed pub/sub system.
16. A repository interface and in-memory implementation.
17. A function that converts DTOs to domain models.
18. A retry wrapper for asynchronous functions.

For each exercise, be ready to explain:

- Why your types are sound.
- Where runtime validation is required.
- Whether the API is easy to use.
- Whether the implementation needs assertions.
- What tradeoffs your design makes.
- How the design would change for production use.

---

## 27. Ten-day preparation plan

| Day | Focus |
|---|---|
| 1 | Inference, primitive types, unions, intersections, literals |
| 2 | Narrowing, type guards, discriminated unions, exhaustiveness |
| 3 | Generics, constraints, `keyof`, indexed access |
| 4 | Utility types, mapped types, conditional types, `infer` |
| 5 | Template literal types, tuples, readonly types, `as const` |
| 6 | Functions, overloads, callbacks, variance, classes |
| 7 | Async code, runtime validation, API typing |
| 8 | Modules, ESM/CommonJS, declaration files, `tsconfig` |
| 9 | Architecture, migration strategy, testing, tooling |
| 10 | Timed coding exercises and mock interview questions |

For mid-level interviews, prioritize narrowing, generics, utility types, async code, nullability, and practical API design.

For senior-level interviews, additionally emphasize runtime boundaries, compiler configuration, structural typing tradeoffs, variance, public API design, migration strategy, domain modeling, maintainability, and knowing when a simpler type is better than a highly sophisticated one.

## 28. Questions commonly asked in interviews

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

## 29. A focused preparation plan

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
