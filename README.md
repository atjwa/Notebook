# JavaScript Interview Study Prep

## 1. `filter()` and `reduce()`

### `Array.prototype.filter()`

`filter()` creates a **new array** containing only the elements for which the callback returns a truthy value.

```js
const numbers = [1, 2, 3, 4, 5, 6];

const evenNumbers = numbers.filter((number) => number % 2 === 0);

console.log(evenNumbers); // [2, 4, 6]
console.log(numbers);     // [1, 2, 3, 4, 5, 6]
```

The original array is not changed.

The callback receives:

```js
array.filter((element, index, array) => {
  // return true to keep the element
});
```

Example:

```js
const users = [
  { name: "Ana", active: true },
  { name: "Ben", active: false },
  { name: "Chris", active: true }
];

const activeUsers = users.filter((user) => user.active);

console.log(activeUsers);
// [{ name: "Ana", active: true }, { name: "Chris", active: true }]
```

Important points:

- Returns a new array.
- May return an empty array.
- Does not mutate the original array.
- The callback should normally be a predicate: something that returns `true` or `false`.
- It is usually used when the result should still be an array.

### `Array.prototype.reduce()`

`reduce()` processes an array and combines its elements into one accumulated result. That result can be a number, string, object, array, map, or anything else.

```js
const numbers = [1, 2, 3, 4];

const sum = numbers.reduce((accumulator, currentValue) => {
  return accumulator + currentValue;
}, 0);

console.log(sum); // 10
```

The second argument, `0`, is the initial accumulator value.

A useful mental model is:

```js
accumulator = initialValue;

for (const currentValue of array) {
  accumulator = callback(accumulator, currentValue);
}

return accumulator;
```

### Common `reduce()` examples

Counting values:

```js
const fruits = ["apple", "banana", "apple", "orange", "banana", "apple"];

const counts = fruits.reduce((result, fruit) => {
  result[fruit] = (result[fruit] || 0) + 1;
  return result;
}, {});

console.log(counts);
// { apple: 3, banana: 2, orange: 1 }
```

Grouping objects:

```js
const users = [
  { name: "Ana", role: "admin" },
  { name: "Ben", role: "user" },
  { name: "Cara", role: "admin" }
];

const usersByRole = users.reduce((groups, user) => {
  if (!groups[user.role]) {
    groups[user.role] = [];
  }

  groups[user.role].push(user);
  return groups;
}, {});

console.log(usersByRole);
```

Calculating an account total:

```js
const accounts = [
  { owner: "A", balance: 100 },
  { owner: "B", balance: 250 },
  { owner: "C", balance: 50 }
];

const totalBalance = accounts.reduce((total, account) => {
  return total + account.balance;
}, 0);

console.log(totalBalance); // 400
```

### `filter()` versus `reduce()`

| Method | Main purpose | Return value | Example |
|---|---|---|---|
| `filter()` | Select matching elements | New array | Active users |
| `reduce()` | Accumulate or transform data | Any value | Total balance |
| `forEach()` | Perform an action for each element | `undefined` | Log each item |

You can combine them:

```js
const totalActiveBalance = accounts
  .filter((account) => account.active)
  .reduce((total, account) => total + account.balance, 0);
```

Use `reduce()` only when it improves clarity. Sometimes `filter()`, `map()`, or a normal loop is easier to understand.

---

## 2. Follow-up questions about `filter()` and `reduce()`

### “How do you know the `reduce()` calculation is working?”

Use several forms of verification:

1. Test a known simple input.
2. Test edge cases.
3. Compare against an independently calculated expected value.
4. Test the accumulator after each step when debugging.
5. Use unit tests.

```js
function getTotalBalance(accounts) {
  return accounts.reduce((total, account) => {
    return total + account.balance;
  }, 0);
}
```

Tests:

```js
console.assert(
  getTotalBalance([
    { balance: 100 },
    { balance: 250 }
  ]) === 350
);

console.assert(getTotalBalance([]) === 0);

console.assert(
  getTotalBalance([{ balance: 0 }]) === 0
);
```

For debugging:

```js
const total = accounts.reduce((total, account) => {
  const nextTotal = total + account.balance;

  console.log({
    previousTotal: total,
    currentBalance: account.balance,
    nextTotal
  });

  return nextTotal;
}, 0);
```

### “What if an account is added?”

The function should not require special changes if it processes the array generically.

```js
const accounts = [
  { balance: 100 },
  { balance: 250 }
];

const firstTotal = getTotalBalance(accounts); // 350

accounts.push({ balance: 50 });

const secondTotal = getTotalBalance(accounts); // 400
```

A good answer should mention that the input data needs to satisfy the expected shape. If an account has a missing or invalid balance, decide how the application should handle it:

```js
function getTotalBalance(accounts) {
  return accounts.reduce((total, account) => {
    const balance = Number(account.balance);

    if (!Number.isFinite(balance)) {
      throw new Error("Invalid account balance");
    }

    return total + balance;
  }, 0);
}
```

### “What happens if you omit the initial value?”

```js
const numbers = [1, 2, 3];

const result = numbers.reduce((sum, number) => sum + number);

console.log(result); // 6
```

Without an initial value, the first element becomes the accumulator and iteration starts at the second element.

This can be dangerous with empty arrays:

```js
[].reduce((sum, number) => sum + number);
// TypeError
```

Therefore, provide an initial value whenever possible:

```js
[].reduce((sum, number) => sum + number, 0); // 0
```

### “Does `filter()` mutate the array?”

No. It creates a new array.

```js
const numbers = [1, 2, 3];

const filtered = numbers.filter((number) => number > 1);

console.log(numbers);  // [1, 2, 3]
console.log(filtered); // [2, 3]
```

However, if the array contains objects, the objects themselves are still references:

```js
const users = [{ active: true }];

const result = users.filter((user) => user.active);
result[0].active = false;

console.log(users[0].active); // false
```

The array was not mutated by `filter()`, but the shared object was mutated afterward.

---

## 3. `forEach()`

`forEach()` executes a callback once for every element. It does not produce a useful return value.

```js
const numbers = [1, 2, 3];

numbers.forEach((number) => {
  console.log(number);
});
```

Its return value is always `undefined`:

```js
const result = numbers.forEach((number) => number * 2);

console.log(result); // undefined
```

Use `forEach()` for side effects such as:

- Logging
- Updating an external system
- Calling a function
- Updating a separate data structure

Do not use it when you need a transformed result. Use `map()` instead:

```js
const doubled = numbers.map((number) => number * 2);
```

Do not use `forEach()` when you need to stop early. A regular `for...of` loop allows `break`:

```js
for (const number of numbers) {
  if (number === 2) {
    break;
  }

  console.log(number);
}
```

### Async caution

This does not wait for asynchronous callbacks:

```js
users.forEach(async (user) => {
  await saveUser(user);
});

console.log("Finished");
```

Use `for...of` if operations must happen sequentially:

```js
for (const user of users) {
  await saveUser(user);
}
```

Or use `Promise.all()` for parallel work:

```js
await Promise.all(
  users.map((user) => saveUser(user))
);
```

---

## 4. “Reducer” in JavaScript and Redux

A reducer is a function that receives a current state and an action, then returns the next state.

```js
function counterReducer(state = 0, action) {
  switch (action.type) {
    case "increment":
      return state + 1;

    case "decrement":
      return state - 1;

    default:
      return state;
  }
}
```

A reducer should generally be:

- Predictable
- Pure
- Free of side effects
- Based only on its inputs
- Returning a new state rather than mutating existing state

Example with an object:

```js
function userReducer(state = { name: "", loggedIn: false }, action) {
  switch (action.type) {
    case "login":
      return {
        ...state,
        name: action.payload,
        loggedIn: true
      };

    case "logout":
      return {
        ...state,
        name: "",
        loggedIn: false
      };

    default:
      return state;
  }
}
```

Incorrect mutation:

```js
state.loggedIn = true;
return state;
```

Preferred immutable update:

```js
return {
  ...state,
  loggedIn: true
};
```

### Relation between `reduce()` and reducers

The word “reducer” comes from the idea of reducing a sequence of actions into a current state:

```js
state = actions.reduce(reducer, initialState);
```

But an array’s `reduce()` method and a Redux reducer are not exactly the same thing. One is an array method; the other is an application-state pattern.

### “Diffusion”

“Diffusion” is not a standard JavaScript array concept. In an interview, it may have been a mishearing of one of these:

- **Destructuring:** extracting values from arrays or objects.
- **Spreading:** using `...` to copy or expand arrays and objects.
- **Dependency injection:** providing dependencies to a component or function.
- **Data diffusion:** an informal term for data spreading through a system.

Destructuring:

```js
const user = { name: "Ana", age: 30 };

const { name, age } = user;
```

Spreading:

```js
const original = [1, 2, 3];
const copy = [...original];

const updatedUser = {
  ...user,
  age: 31
};
```

---

## 5. Architectural improvements

For the question, “Architecturally, what would you do differently in your last project?”, avoid criticizing people. Discuss trade-offs, evidence, and improvements.

A strong answer could be:

> In my last project, I would improve the separation between presentation, business logic, and data access. Some components had too much responsibility: they rendered UI, made API calls, transformed responses, and handled business rules. I would move API calls into a service layer, keep business logic in reusable functions, and make components primarily responsible for presentation and user interaction. I would also improve automated test coverage around the business rules and introduce clearer boundaries between modules. I would make these changes incrementally rather than attempting a complete rewrite.

Possible architectural improvements:

- Separate UI, business logic, and API/data-access layers.
- Establish consistent error handling.
- Create shared validation utilities.
- Reduce duplicated code.
- Improve typing with TypeScript.
- Add unit and integration tests.
- Improve observability and logging.
- Define clearer component boundaries.
- Improve state-management structure.
- Add documentation for important decisions.
- Reduce unnecessary coupling between modules.
- Improve performance only where profiling shows a real problem.

A good architecture answer includes:

1. What the problem was.
2. Why it mattered.
3. What change you would make.
4. What trade-off the change involves.
5. How you would measure success.

---

## 6. “If you had a week, what would you change?”

A week is suitable for a focused, low-risk improvement.

Example answer:

> With one week, I would choose a contained improvement with a clear benefit, such as adding tests around the most important business logic, improving error handling for a high-traffic API flow, or removing duplicated validation. I would first measure the current behavior, make the change incrementally, run the existing test suite, and document the result.

Good one-week projects:

- Add tests for a critical module.
- Fix a recurring bug.
- Improve loading and error states.
- Refactor one highly complex component.
- Remove duplicated utility logic.
- Add input validation.
- Improve logging for a production issue.
- Optimize a proven bottleneck.

Avoid saying you would rewrite the entire application in a week.

---

## 7. “If you had a month, what would you change?”

A month allows a broader architectural improvement, but it should still be broken into deliverable stages.

Example answer:

> With a month, I would address the larger structural issue rather than just one symptom. I might introduce a clearer service layer, improve the state-management boundaries, migrate a risky area to TypeScript, or establish a more complete testing strategy. I would break that into milestones: document the current architecture, agree on the target design, implement it in one area, measure the result, and then expand it gradually. I would avoid a large rewrite without first proving the approach.

A strong answer mentions:

- Discovery and technical design.
- Agreement with the team.
- Incremental implementation.
- Backward compatibility.
- Testing.
- Metrics or success criteria.
- Risk management.

---

## 8. Unit testing walkthrough

A unit test tests one small unit of behavior, usually a function or module, in isolation.

Suppose you have:

```js
export function calculateTotal(items) {
  return items.reduce((total, item) => {
    return total + item.price * item.quantity;
  }, 0);
}
```

Using Jest or Vitest:

```js
import { describe, it, expect } from "vitest";
import { calculateTotal } from "./calculateTotal";

describe("calculateTotal", () => {
  it("calculates the total for multiple items", () => {
    const items = [
      { price: 10, quantity: 2 },
      { price: 5, quantity: 3 }
    ];

    expect(calculateTotal(items)).toBe(35);
  });

  it("returns zero for an empty list", () => {
    expect(calculateTotal([])).toBe(0);
  });

  it("handles a zero quantity", () => {
    expect(
      calculateTotal([{ price: 10, quantity: 0 }])
    ).toBe(0);
  });
});
```

### Unit-testing process

1. Identify the behavior to test.
2. Arrange the input and dependencies.
3. Act by calling the function.
4. Assert the expected result.
5. Add edge cases.
6. Run the test and investigate failures.
7. Keep the test independent and repeatable.

This is the Arrange–Act–Assert pattern:

```js
// Arrange
const input = [1, 2, 3];

// Act
const result = sum(input);

// Assert
expect(result).toBe(6);
```

### What should be tested?

Test:

- Normal cases.
- Empty input.
- Boundary values.
- Invalid input, if the function handles it.
- Error behavior.
- Different branches.
- Important business rules.

Avoid testing implementation details unnecessarily. This test is usually better:

```js
expect(calculateTotal(items)).toBe(35);
```

than testing exactly how many times `reduce()` internally called a callback.

### Mocking

Mock external dependencies, not usually simple internal logic.

```js
vi.mock("./api", () => ({
  fetchUser: vi.fn()
}));
```

Then test how your function behaves when the API:

- Returns valid data.
- Returns an error.
- Returns an empty response.
- Times out.

A good interview statement is:

> I use unit tests for deterministic logic and integration tests where modules need to work together. I mock external systems when I want to isolate the unit, but I avoid excessive mocking because tests can become coupled to implementation details.

---

## 9. `var`, `let`, and `const`

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function-scoped | Block-scoped | Block-scoped |
| Reassignable | Yes | Yes | No |
| Redeclarable in same scope | Usually yes | No | No |
| Hoisted | Yes, initialized as `undefined` | Yes, but temporal dead zone | Yes, but temporal dead zone |
| Recommended for new code | Rarely | When reassignment is needed | Default choice |

### Scope difference

```js
if (true) {
  var a = 1;
  let b = 2;
  const c = 3;
}

console.log(a); // 1
console.log(b); // ReferenceError
console.log(c); // ReferenceError
```

`let` and `const` are block-scoped. `var` is not.

### Reassignment

```js
var x = 1;
x = 2;

let y = 1;
y = 2;

const z = 1;
// z = 2; // TypeError
```

Use `const` by default. Use `let` when the variable must be reassigned. Avoid `var` in modern JavaScript unless you specifically need its behavior.

### Temporal dead zone

```js
console.log(value); // ReferenceError
let value = 10;
```

The variable is hoisted in the sense that its binding exists, but it cannot be accessed before its declaration.

---

## 10. `null`, `undefined`, and undeclared variables

### `undefined`

`undefined` generally means a variable exists but has no assigned value.

```js
let value;

console.log(value); // undefined
```

It can also result from:

```js
const user = {};
console.log(user.name); // undefined

function doSomething() {}
console.log(doSomething()); // undefined
```

### `null`

`null` is an intentional absence of a value.

```js
const selectedUser = null;
```

It often means: “There is deliberately no object or value here.”

```js
let user = null;

// Later:
user = { name: "Ana" };
```

A historical JavaScript quirk:

```js
typeof null; // "object"
```

This is a longstanding language bug. `null` is not actually an object.

### Undeclared

An undeclared variable has never been declared.

```js
console.log(total); // ReferenceError
```

In sloppy mode, assigning to an undeclared variable may create a global variable:

```js
total = 10;
```

In strict mode, this throws an error:

```js
"use strict";

total = 10; // ReferenceError
```

### Comparisons

```js
null == undefined;  // true
null === undefined; // false
```

Prefer strict equality:

```js
value === null
value === undefined
```

A concise null-or-undefined check is:

```js
value == null
```

But this uses intentional loose equality and should be explained clearly if used in an interview.

---

## 11. Conditional shorthand operators

### Logical AND: `&&`

Returns the first falsy value, or the final value if everything is truthy.

```js
const result = true && "hello";
console.log(result); // "hello"
```

Common conditional use:

```js
isLoggedIn && showDashboard();
```

This means `showDashboard()` runs only if `isLoggedIn` is truthy.

Be careful with numbers:

```js
const count = 0;

count && console.log("There are items");
// Nothing logs because 0 is falsy
```

In React, this can accidentally render `0`:

```jsx
{count && <List />}
```

A safer condition is:

```jsx
{count > 0 && <List />}
```

### Logical OR: `||`

Returns the first truthy value.

```js
const username = providedName || "Guest";
```

The fallback is used for all falsy values, including:

- `false`
- `0`
- `""`
- `null`
- `undefined`
- `NaN`

That can be undesirable:

```js
const volume = savedVolume || 50;
// If savedVolume is 0, volume becomes 50
```

### Nullish coalescing: `??`

Uses the fallback only when the left side is `null` or `undefined`.

```js
const volume = savedVolume ?? 50;

console.log(0 ?? 50);  // 0
console.log("" ?? "x"); // ""
console.log(null ?? 50); // 50
```

Use `??` when `0`, `false`, or an empty string are valid values.

You cannot freely mix `??` with `&&` or `||` without parentheses:

```js
const result = (a || b) ?? c;
```

### Ternary operator: `condition ? valueIfTrue : valueIfFalse`

```js
const message = isLoggedIn ? "Welcome" : "Please log in";
```

It is useful for assigning one of two values. Avoid deeply nested ternaries because they reduce readability.

### Optional chaining: `?.`

Although not in your list, it is commonly asked alongside these operators:

```js
const city = user?.address?.city;
```

If `user` or `address` is `null` or `undefined`, the expression returns `undefined` instead of throwing.

---

## 12. Can you push or pop from a `const` array?

Yes.

```js
const numbers = [1, 2, 3];

numbers.push(4);
numbers.pop();

console.log(numbers); // [1, 2, 3]
```

`const` prevents reassignment of the variable binding. It does not make the array immutable.

This is not allowed:

```js
numbers = [10, 20]; // TypeError
```

The variable still points to the same array:

```js
const numbers = [1, 2];

numbers.push(3); // allowed
// numbers = [1, 2, 3]; // not allowed
```

For an immutable update:

```js
const updatedNumbers = [...numbers, 3];
```

To prevent mutation at runtime:

```js
const numbers = Object.freeze([1, 2, 3]);

// numbers.push(4); // TypeError in strict mode
```

`Object.freeze()` is shallow, so nested objects may still be mutable.

---

## 13. Can properties be added or removed from a `const` object?

Yes.

```js
const user = {
  name: "Ana"
};

user.age = 30;
delete user.name;

console.log(user); // { age: 30 }
```

But the variable cannot be reassigned:

```js
user = { name: "Ben" }; // TypeError
```

Again, `const` protects the reference, not the contents of the object.

Immutable update:

```js
const updatedUser = {
  ...user,
  role: "admin"
};
```

To prevent direct changes:

```js
const user = Object.freeze({
  name: "Ana"
});

// user.age = 30; // Error in strict mode
```

### Strong short answer for interviews

> `const` means the variable binding cannot be reassigned. It does not make referenced objects or arrays immutable. Therefore, I can mutate a `const` array with `push()` or a `const` object by adding properties, but I cannot assign a completely new array or object to the variable.

---

## 14. Quick interview answers

**What is the difference between `filter()` and `reduce()`?**

> `filter()` returns a new array containing elements that satisfy a condition. `reduce()` accumulates array elements into a single result, which can be any type.

**What is the difference between `forEach()` and `map()`?**

> `forEach()` is for side effects and returns `undefined`. `map()` transforms every element and returns a new array.

**Why provide an initial value to `reduce()`?**

> It makes the accumulator’s type explicit, handles empty arrays safely, and makes the behavior more predictable.

**What is a pure reducer?**

> A reducer returns the same output for the same input and has no side effects. It should not mutate the existing state.

**When would you use `||` versus `??`?**

> I use `||` when any falsy value should trigger the fallback. I use `??` when only `null` or `undefined` should trigger it, because `0`, `false`, and `""` may be valid values.

**Does `const` make an object immutable?**

> No. It prevents reassignment of the reference, but the object’s properties can still be changed unless the object is frozen or updated immutably.

**What is the difference between `null` and `undefined`?**

> `undefined` commonly means no value has been assigned, while `null` is an explicit assignment representing no value.

**How would you improve a previous project?**

> I would identify a specific problem, explain its impact, propose an incremental improvement, mention the trade-offs, and define how I would measure success. For example, I might separate API access from UI components and add tests around critical business logic.
