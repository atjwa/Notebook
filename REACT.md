# React Practical Interview Review Guide

For a “light” React interview, expect a small, realistic UI rather than a difficult algorithm problem. The interviewer will usually evaluate whether you can understand an existing codebase, model state correctly, build accessible components, handle edge cases, debug efficiently, and explain your decisions.

CoderPad’s React environment supports JavaScript, JSX, and TypeScript, with IntelliSense across the project. Its interview environment lets you write code in one pane, run it, and inspect output, so practice the workflow of making small changes and validating them frequently. <citation src="1,3,5"></citation>

## 1. What the interview is likely to test

Prioritize these areas:

| Area | What you should demonstrate |
|---|---|
| Component design | Break the UI into sensible, focused components |
| Props and state | Keep state minimal and place it at the correct level |
| Rendering | Correctly render arrays, conditional content, and empty states |
| Event handling | Handle clicks, form submissions, keyboard actions, and input changes |
| Effects | Fetch data or synchronize with external systems without creating loops |
| Async behavior | Loading, success, error, cancellation, and retry states |
| Debugging | Read errors, isolate the issue, and test incrementally |
| Accessibility | Use semantic HTML, labels, buttons, and keyboard-friendly controls |
| Code quality | Clear names, small functions, maintainable structure |
| Communication | Explain tradeoffs and narrate your reasoning |

A typical practical task might be a todo list, searchable list, user directory, shopping cart, autocomplete, pagination component, or small dashboard. CoderPad’s own React interview examples include a todo list with adding and completing items. <citation src="1"></citation>

---

## 2. The React fundamentals you should know cold

### Components and props

A component should generally:

- Receive data through props.
- Render UI from those props.
- Notify its parent about changes through callback props.
- Avoid mutating props directly.

```jsx
function UserCard({ user, onSelect }) {
  return (
    <article>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={() => onSelect(user.id)}>
        View profile
      </button>
    </article>
  );
}
```

Be ready to explain:

- Why `user` is a prop rather than local state.
- Why the parent owns the selection behavior.
- Why the callback is passed down instead of modifying shared state from the child.

### State

Use state for information that:

1. Can change over time, and
2. Affects what is rendered.

```jsx
const [query, setQuery] = useState("");
const [todos, setTodos] = useState([]);
```

Avoid storing values that can be calculated from existing state or props.

```jsx
// Usually unnecessary state
const [completedCount, setCompletedCount] = useState(0);

// Prefer derived data
const completedCount = todos.filter((todo) => todo.completed).length;
```

This avoids synchronization bugs.

### State updates are immutable

Do not mutate arrays or objects in state:

```jsx
// Incorrect
todos.push(newTodo);
setTodos(todos);
```

Instead:

```jsx
setTodos((previousTodos) => [
  ...previousTodos,
  newTodo,
]);
```

For an object:

```jsx
setForm((previousForm) => ({
  ...previousForm,
  email: value,
}));
```

For updating one array item:

```jsx
setTodos((previousTodos) =>
  previousTodos.map((todo) =>
    todo.id === id
      ? { ...todo, completed: !todo.completed }
      : todo
  )
);
```

For removing an item:

```jsx
setTodos((previousTodos) =>
  previousTodos.filter((todo) => todo.id !== id)
);
```

Use the functional updater form when the new value depends on the previous value.

---

## 3. Rendering lists correctly

Always provide a stable key:

```jsx
{todos.map((todo) => (
  <li key={todo.id}>{todo.title}</li>
))}
```

Avoid using the array index as a key when items can be inserted, removed, sorted, or reordered:

```jsx
// Risky
{todos.map((todo, index) => (
  <TodoItem key={index} todo={todo} />
))}
```

Why this matters: React uses keys to associate rendered elements with items. Unstable keys can cause incorrect component state, focus problems, and surprising updates.

Also handle empty collections explicitly:

```jsx
{todos.length === 0 ? (
  <p>No tasks yet.</p>
) : (
  <ul>
    {todos.map((todo) => (
      <TodoItem key={todo.id} todo={todo} />
    ))}
  </ul>
)}
```

Be cautious with truthiness:

```jsx
// This fails to render 0
{count && <span>{count}</span>}
```

Use:

```jsx
{count > 0 && <span>{count}</span>}
```

---

## 4. Forms and controlled inputs

A controlled input gets its value from React state:

```jsx
function TodoForm({ onAdd }) {
  const [title, setTitle] = useState("");

  function handleSubmit(event) {
    event.preventDefault();

    const trimmedTitle = title.trim();

    if (!trimmedTitle) {
      return;
    }

    onAdd(trimmedTitle);
    setTitle("");
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="todo-title">Task</label>

      <input
        id="todo-title"
        value={title}
        onChange={(event) => setTitle(event.target.value)}
      />

      <button type="submit">Add task</button>
    </form>
  );
}
```

Know how to handle:

- `event.preventDefault()`
- Empty or whitespace-only input
- Submit with the Enter key
- Resetting the form after success
- Validation errors
- Labels and accessible input names
- Disabled states while submitting

For multiple fields:

```jsx
const [form, setForm] = useState({
  name: "",
  email: "",
});

function handleChange(event) {
  const { name, value } = event.target;

  setForm((previousForm) => ({
    ...previousForm,
    [name]: value,
  }));
}
```

---

## 5. `useEffect`: understand the purpose, not just the syntax

An effect is for synchronizing with something outside React, such as:

- Network requests
- Browser APIs
- Timers
- Subscriptions
- Manually controlled non-React systems

It is not a general-purpose place for ordinary calculations.

### Fetching data

```jsx
function UsersList() {
  const [users, setUsers] = useState([]);
  const [status, setStatus] = useState("idle");
  const [error, setError] = useState(null);

  useEffect(() => {
    let ignore = false;

    async function loadUsers() {
      setStatus("loading");
      setError(null);

      try {
        const response = await fetch("/api/users");

        if (!response.ok) {
          throw new Error("Failed to fetch users");
        }

        const data = await response.json();

        if (!ignore) {
          setUsers(data);
          setStatus("success");
        }
      } catch (err) {
        if (!ignore) {
          setError(err);
          setStatus("error");
        }
      }
    }

    loadUsers();

    return () => {
      ignore = true;
    };
  }, []);

  if (status === "loading") {
    return <p>Loading users…</p>;
  }

  if (status === "error") {
    return <p role="alert">{error.message}</p>;
  }

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

Understand the dependency array:

```jsx
useEffect(() => {
  // Runs after every render
});
```

```jsx
useEffect(() => {
  // Runs once after initial mount
}, []);
```

```jsx
useEffect(() => {
  // Runs initially and whenever userId changes
}, [userId]);
```

Common mistakes:

- Omitting dependencies.
- Putting unstable objects or functions in dependencies unnecessarily.
- Calling a state setter in an effect that immediately retriggers the same effect.
- Forgetting cleanup.
- Using an effect to derive data that could be calculated during rendering.

### Prefer request cancellation when appropriate

For fetch requests, use `AbortController` when the request should actually be canceled:

```jsx
useEffect(() => {
  const controller = new AbortController();

  async function loadData() {
    try {
      const response = await fetch("/api/items", {
        signal: controller.signal,
      });

      const data = await response.json();
      setItems(data);
    } catch (error) {
      if (error.name !== "AbortError") {
        setError(error);
      }
    }
  }

  loadData();

  return () => controller.abort();
}, []);
```

---

## 6. State modeling patterns

### Keep state minimal

Suppose you have:

```jsx
const [todos, setTodos] = useState([]);
const [filter, setFilter] = useState("all");
```

Derive the visible todos:

```jsx
const visibleTodos = todos.filter((todo) => {
  if (filter === "active") return !todo.completed;
  if (filter === "completed") return todo.completed;
  return true;
});
```

Do not separately store `visibleTodos`, because then `todos` and `visibleTodos` can become inconsistent.

### Lift state up

If two sibling components need the same data, place the state in their nearest common parent.

```jsx
function TodoApp() {
  const [todos, setTodos] = useState([]);

  return (
    <>
      <TodoForm onAdd={addTodo} />
      <TodoList todos={todos} onToggle={toggleTodo} />
    </>
  );
}
```

The form does not need to own the full todo list. The list does not need to know how todos are created.

### Use `useReducer` for complex transitions

For a light interview, `useState` is usually sufficient. Use `useReducer` when state has multiple related transitions:

```jsx
const initialState = {
  status: "idle",
  data: [],
  error: null,
};

function reducer(state, action) {
  switch (action.type) {
    case "load":
      return { ...state, status: "loading", error: null };

    case "success":
      return { status: "success", data: action.data, error: null };

    case "error":
      return { ...state, status: "error", error: action.error };

    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}
```

Be able to explain that a reducer centralizes state transitions and makes the allowed events explicit.

---

## 7. A practical implementation strategy

When you receive the prompt, use this sequence.

### Step 1: Clarify the requirements

Ask concise questions if the prompt leaves important ambiguity:

- What should happen when the list is empty?
- Should search be case-insensitive?
- Should duplicate items be allowed?
- What should happen on a failed request?
- Is persistence required?
- Should filtering happen as the user types?
- Are there accessibility expectations?

Do not spend several minutes asking questions that do not affect implementation.

### Step 2: Identify the state

Write down only the changing values.

For a todo app:

```text
todos
newTodoTitle
filter
```

For a user search page:

```text
query
users
status
error
```

### Step 3: Sketch the component structure

For example:

```text
TodoApp
├── TodoForm
├── TodoFilters
└── TodoList
    └── TodoItem
```

Do not over-engineer. Three clear components are better than ten trivial wrapper components.

### Step 4: Build the simplest working path

Implement in this order:

1. Static layout
2. Main state
3. Primary interaction
4. Rendering the updated state
5. Empty state
6. Validation
7. Loading and error states
8. Accessibility and polish

This ensures you have a working solution even if time runs short.

### Step 5: Run early

Do not write the whole solution before running it. Run after:

- The initial render
- Adding the first event handler
- Adding state updates
- Adding list rendering
- Adding async behavior

CoderPad’s candidate guide specifically encourages using the run output for debugging, and its Reset control clears the output while preserving code. <citation src="3"></citation>

### Step 6: Test manually

Try:

- Empty input
- Whitespace-only input
- One item
- Multiple items
- Duplicate values
- Removing the first, middle, and last item
- Rapid clicks
- Slow or failed requests
- Empty search results
- Keyboard submission
- Refreshing or remounting the component

### Step 7: Explain tradeoffs

Useful commentary sounds like:

> “I’m keeping the full todo array in the parent because both the form and list need access to it. The filtered list is derived rather than stored separately, which avoids synchronization problems.”

---

## 8. Common practical exercises

### Exercise 1: Todo list

Requirements:

- Add a todo.
- Mark a todo complete.
- Delete a todo.
- Filter by all, active, and completed.
- Show an empty state.

Core solution shape:

```jsx
function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [title, setTitle] = useState("");
  const [filter, setFilter] = useState("all");

  function addTodo(event) {
    event.preventDefault();

    const trimmedTitle = title.trim();

    if (!trimmedTitle) {
      return;
    }

    setTodos((previousTodos) => [
      ...previousTodos,
      {
        id: crypto.randomUUID(),
        title: trimmedTitle,
        completed: false,
      },
    ]);

    setTitle("");
  }

  function toggleTodo(id) {
    setTodos((previousTodos) =>
      previousTodos.map((todo) =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  }

  function deleteTodo(id) {
    setTodos((previousTodos) =>
      previousTodos.filter((todo) => todo.id !== id)
    );
  }

  const visibleTodos = todos.filter((todo) => {
    if (filter === "active") return !todo.completed;
    if (filter === "completed") return todo.completed;
    return true;
  });

  return (
    <main>
      <h1>Todos</h1>

      <form onSubmit={addTodo}>
        <label htmlFor="todo">New todo</label>
        <input
          id="todo"
          value={title}
          onChange={(event) => setTitle(event.target.value)}
        />
        <button type="submit">Add</button>
      </form>

      <select
        value={filter}
        onChange={(event) => setFilter(event.target.value)}
        aria-label="Filter todos"
      >
        <option value="all">All</option>
        <option value="active">Active</option>
        <option value="completed">Completed</option>
      </select>

      {visibleTodos.length === 0 ? (
        <p>No todos found.</p>
      ) : (
        <ul>
          {visibleTodos.map((todo) => (
            <li key={todo.id}>
              <label>
                <input
                  type="checkbox"
                  checked={todo.completed}
                  onChange={() => toggleTodo(todo.id)}
                />
                {todo.title}
              </label>

              <button
                type="button"
                onClick={() => deleteTodo(todo.id)}
              >
                Delete
              </button>
            </li>
          ))}
        </ul>
      )}
    </main>
  );
}
```

Potential follow-up questions:

- How would you persist this?
- How would you avoid duplicate todos?
- How would you add editing?
- How would you optimize a list with thousands of items?
- Where should filtering logic live?

### Exercise 2: Searchable user list

Requirements:

- Fetch users.
- Show loading and error states.
- Filter by name.
- Show “no matches.”
- Avoid case-sensitive search bugs.

```jsx
const normalizedQuery = query.trim().toLowerCase();

const matchingUsers = users.filter((user) =>
  user.name.toLowerCase().includes(normalizedQuery)
);
```

Discuss whether the filtering should be client-side or server-side. For a small loaded dataset, client-side filtering is simple. For a large dataset, server-side search may be more appropriate.

### Exercise 3: Autocomplete

Important concerns:

- Debounce input.
- Handle stale requests.
- Show loading state.
- Handle no results.
- Support keyboard navigation.
- Close the menu when an option is selected.

For a light interview, you may not need to fully implement keyboard navigation, but mention it if time is limited.

### Exercise 4: Shopping cart

State might look like:

```jsx
[
  {
    id: "p1",
    name: "Keyboard",
    price: 80,
    quantity: 2,
  }
]
```

Derived values:

```jsx
const subtotal = cart.reduce(
  (total, item) => total + item.price * item.quantity,
  0
);
```

Important edge cases:

- Quantity cannot go below one.
- Removing an item updates totals.
- Adding an existing product increments quantity.
- Prices should not be accidentally treated as strings.
- Currency formatting should be centralized.

### Exercise 5: Pagination

Track:

```jsx
const [page, setPage] = useState(1);
const [pageSize, setPageSize] = useState(10);
```

Be ready to explain:

- How you calculate the offset.
- What happens when the page size changes.
- How you disable Previous and Next buttons.
- What happens when a filter reduces the number of pages.
- Whether pagination is client-side or server-side.

---

## 9. JavaScript topics worth reviewing

A React interview often tests JavaScript indirectly.

### Array methods

Know these well:

```js
map
filter
reduce
find
some
every
includes
sort
```

Remember that `sort()` mutates the array:

```js
const sorted = [...items].sort(compareItems);
```

### Closures

Understand why this can be problematic:

```jsx
setCount(count + 1);
setCount(count + 1);
```

Both updates may use the same captured value. Prefer:

```jsx
setCount((previousCount) => previousCount + 1);
setCount((previousCount) => previousCount + 1);
```

### Equality and coercion

Prefer strict equality:

```js
value === 0
```

Be aware that values from inputs are strings:

```jsx
const quantity = Number(event.target.value);
```

### Optional chaining and nullish coalescing

```js
const city = user?.address?.city ?? "Unknown";
```

Know the difference between `||` and `??`:

```js
const pageSize = providedPageSize ?? 10;
```

This preserves valid falsy values such as `0`, while `||` would replace them.

### Async JavaScript

Know how to:

- Return and await promises.
- Catch errors.
- Check `response.ok`.
- Avoid updating state after a request is no longer relevant.
- Handle parallel requests with `Promise.all`.

---

## 10. Accessibility checklist

Even in a short interview, accessibility can distinguish a strong implementation.

Use:

- `<button>` for actions, not clickable `<div>` elements.
- `<label>` connected to inputs.
- `<form>` for forms.
- `<main>`, `<nav>`, `<section>`, and headings where appropriate.
- `type="button"` for non-submit buttons inside forms.
- `aria-live` or `role="alert"` for important dynamic messages.
- Visible focus indicators.
- Meaningful button text.

Good:

```jsx
<button type="button" onClick={handleDelete}>
  Delete task
</button>
```

Less effective:

```jsx
<div onClick={handleDelete}>×</div>
```

If an icon-only button is necessary:

```jsx
<button
  type="button"
  aria-label="Delete task"
  onClick={handleDelete}
>
  ×
</button>
```

---

## 11. Performance topics

For a small coding exercise, correctness matters more than premature optimization. Still, know the concepts.

### Avoid unnecessary state

Derived values should normally be calculated:

```jsx
const total = items.reduce(...);
```

rather than synchronized through an effect.

### `useMemo`

Use it when a calculation is expensive and its dependencies are stable:

```jsx
const filteredItems = useMemo(
  () => expensiveFilter(items, query),
  [items, query]
);
```

Do not use it automatically for every calculation.

### `useCallback`

Useful when:

- A callback is passed to a memoized child.
- The callback causes unnecessary child re-renders.
- The function identity matters to a dependency array.

```jsx
const handleSelect = useCallback((id) => {
  setSelectedId(id);
}, []);
```

Do not add `useCallback` everywhere without a measurable reason.

### `React.memo`

Can prevent a child from re-rendering when its props have not changed, but it will not help if you create new object or function props every render.

### Large lists

For thousands of rows, discuss:

- Pagination
- Virtualization
- Server-side filtering
- Debounced search
- Avoiding unnecessary row renders

---

## 12. Debugging checklist

When something does not work:

1. Read the exact error message.
2. Identify the component and line involved.
3. Check whether the component renders at all.
4. Log the relevant state and props.
5. Verify the event handler is actually called.
6. Check whether the state update is immutable.
7. Check list keys.
8. Inspect the network response and status.
9. Check effect dependencies.
10. Reduce the problem to the smallest failing case.

### Common bugs

#### State is not updating

Possible causes:

- Mutating the existing array or object.
- Setting the same reference.
- Reading stale state.
- Updating the wrong item ID.

#### Infinite effect loop

Often caused by:

```jsx
useEffect(() => {
  setData(transform(data));
}, [data]);
```

The effect updates `data`, which triggers the effect again. Derive the transformed value during render or revise the state model.

#### Input cannot be edited

This usually means the input has a `value` but no matching `onChange` update.

```jsx
<input value={name} onChange={(e) => setName(e.target.value)} />
```

#### List items appear to swap state

Check whether you used array indices as keys for a reorderable list.

#### Fetch appears to run too often

Check:

- The dependency array.
- Whether an object or function is recreated on every render.
- Whether development behavior is causing extra effect execution.
- Whether the effect is needed at all.

---

## 13. Testing mindset

Even if you are not asked to write tests, think in terms of behavior.

For a todo app, test:

- Adding a valid todo.
- Rejecting empty input.
- Completing a todo.
- Deleting a todo.
- Filtering results.
- Rendering the empty state.
- Preserving all other todos during an update.

A useful distinction:

- Test what the user can observe.
- Avoid testing implementation details unnecessarily.

For example, “click Add and see the new task” is more meaningful than testing whether a particular internal setter was called.

---

## 14. TypeScript essentials

If the pad uses TypeScript, know how to type props and state.

```tsx
type Todo = {
  id: string;
  title: string;
  completed: boolean;
};

type TodoItemProps = {
  todo: Todo;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
};

function TodoItem({
  todo,
  onToggle,
  onDelete,
}: TodoItemProps) {
  // ...
}
```

For an input event:

```tsx
function handleChange(
  event: React.ChangeEvent<HTMLInputElement>
) {
  setTitle(event.target.value);
}
```

For a form event:

```tsx
function handleSubmit(
  event: React.FormEvent<HTMLFormElement>
) {
  event.preventDefault();
}
```

Avoid using `any` unless there is a strong reason. If time is short, prioritize correct behavior over perfect type sophistication.

---

## 15. A 60-minute practice plan

| Time | Activity |
|---:|---|
| 5 min | Read requirements and clarify assumptions |
| 5 min | Identify state and sketch components |
| 15 min | Build the main UI and happy path |
| 10 min | Add validation and edge cases |
| 10 min | Add loading/error/empty states |
| 5 min | Improve accessibility |
| 5 min | Test manually |
| 5 min | Refactor and explain tradeoffs |

A good practice constraint is to complete a small app in 30–40 minutes, then spend the remaining time improving it rather than trying to build an overly ambitious solution.

---

## 16. What to say while coding

Narrate decisions without describing every keystroke.

Useful examples:

- “I’ll keep the source list in state and derive the filtered list.”
- “This state belongs in the parent because both siblings need it.”
- “I’m using a functional update because this value depends on the previous state.”
- “I’m adding an explicit empty state so the UI is understandable when there are no results.”
- “I’ll implement the happy path first, then add loading and error handling.”
- “I’m using a stable ID for the key because the list can be reordered or deleted.”
- “I’m checking `response.ok`; `fetch` does not reject merely because the server returns a 404.”
- “I’ll run this now to catch syntax and rendering issues before adding more logic.”

If you get stuck:

> “I’m going to reduce this to the smallest failing interaction, verify the state before and after the event, and then add the rest back incrementally.”

That demonstrates a disciplined debugging process.

---

## 17. CoderPad-specific preparation

Before the interview:

- Open the exact interview link early.
- Use a supported, updated browser.
- Test your microphone and camera if a call is included.
- Practice in CoderPad’s sandbox.
- Confirm whether JavaScript, JSX, or TypeScript is expected.
- Ask whether external documentation or AI assistance is allowed.
- Know where the Run, Reset, Settings, and instructions controls are.
- Practice using keyboard shortcuts rather than relying on a full local IDE.

CoderPad’s preparation guide provides a sandbox that approximates the interview experience, although it does not include every collaborative feature. It also notes that the interviewer may enable AI Assist, in which case you should clarify what usage is permitted before using it. Instructions can also be opened in a separate window, which may be helpful for a two-screen setup. <citation src="3"></citation>

During the interview:

- Read the entire prompt before coding.
- Keep the instructions visible.
- Run code frequently.
- Ask for clarification when behavior is ambiguous.
- Do not silently struggle for a long time.
- Mention what you would improve if time allowed.
- If a tool behaves unexpectedly, explain the issue and continue with a reasonable workaround.

---

## 18. Final high-priority review list

If you have limited preparation time, review these first:

1. Controlled inputs and form submission.
2. Immutable array and object updates.
3. Stable list keys.
4. Derived state versus stored state.
5. `useEffect` dependencies and cleanup.
6. Loading, error, and empty states.
7. Fetching and checking `response.ok`.
8. Parent-child communication through props and callbacks.
9. Array methods: `map`, `filter`, `reduce`, and `find`.
10. Accessibility basics.
11. Debugging by logging state, props, and events.
12. Explaining your design as you work.

The strongest “light practical” candidate is not the person who writes the most code. It is the person who produces a working solution, handles the obvious edge cases, makes sensible tradeoffs, and communicates clearly while debugging.
