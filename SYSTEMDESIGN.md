# React + TypeScript Systems Design Interview Guide

A strong mid-to-senior answer connects **product requirements, frontend architecture, runtime behavior, performance, reliability, security, and maintainability**. Do not begin by naming libraries. Begin by clarifying the problem and making explicit tradeoffs.

## 1. A reliable interview framework

Use this sequence for almost any frontend systems-design question:

1. **Clarify requirements**
   - Who are the users?
   - What are the core user journeys?
   - Is the application read-heavy, write-heavy, collaborative, or offline-capable?
   - What are the latency, availability, accessibility, and SEO requirements?
   - What scale matters: users, records, requests, bundle size, or rendered rows?

2. **Define the critical user experience**
   - Initial page load
   - Navigation
   - Search/filtering
   - Editing and saving
   - Error recovery
   - Mobile and slow-network behavior

3. **Sketch the architecture**
   - Browser
   - CDN/edge
   - React application
   - API or backend-for-frontend
   - Database and external services
   - Caches, queues, analytics, and observability

4. **Define data ownership**
   - What belongs to the server?
   - What is local UI state?
   - What can be cached?
   - What must be synchronized or conflict-resolved?

5. **Explain rendering and performance**
   - CSR, SSR, static generation, streaming, or hybrid rendering
   - Code splitting
   - Caching
   - Virtualization
   - Request cancellation and deduplication

6. **Cover failure modes**
   - API timeout
   - Partial failure
   - Stale data
   - Duplicate submissions
   - Offline edits
   - Permission changes
   - Schema evolution

7. **Discuss tradeoffs and evolution**
   - What would you ship first?
   - What changes at 10× scale?
   - What would you intentionally avoid?

A useful opening is:

> “I’ll first clarify the user flows and quality attributes, then define the data model and API boundary, choose a rendering strategy, describe client state and caching, and finally cover performance, reliability, security, and observability.”

---

## 2. Core architecture

A typical React + TypeScript application can be divided into these layers:

```text
Presentation
  Pages, layouts, components, accessibility

Application
  Use cases, orchestration, navigation, permissions

Domain
  Business rules, entities, value objects, validation

Data access
  API clients, repositories, cache adapters, serializers

Infrastructure
  HTTP, logging, analytics, storage, feature flags
```

A practical feature-oriented structure:

```text
src/
  app/
    router.tsx
    providers.tsx
  features/
    orders/
      api.ts
      model.ts
      hooks.ts
      components/
      pages/
      tests/
  shared/
    ui/
    http/
    validation/
    types/
  lib/
    telemetry.ts
```

Prefer **feature boundaries** over a single global folder for components, hooks, and services. This makes ownership clearer and limits accidental coupling.

A common senior-level principle:

> Components should express UI intent; data-fetching, domain rules, and transport details should live behind explicit boundaries.

Avoid both extremes:

- Putting API calls directly in every component
- Creating excessive abstraction layers before the domain requires them

---

## 3. React rendering model

### CSR

The browser downloads JavaScript and renders the application.

**Advantages**

- Highly interactive
- Simple mental model
- Good for authenticated dashboards

**Costs**

- Slower first meaningful content on slow devices
- More JavaScript shipped
- Poorer SEO unless supplemented with pre-rendering

### SSR

The server generates HTML for a request, and the browser hydrates it.

**Advantages**

- Faster visible content
- Better SEO
- Useful for personalized but indexable pages

**Costs**

- Server rendering complexity
- Hydration cost
- Potential server/client markup mismatches

Hydration attaches client behavior to server-rendered HTML; it does not simply “render the page again.”

### Static generation

HTML is generated at build time.

**Good for**

- Documentation
- Marketing pages
- Product catalogs with infrequent changes

### Streaming

The server sends usable parts of the UI as they become available.

**Good for**

- Pages with slow secondary sections
- Large dashboards
- Reducing time to first meaningful content

### Server and client components

Where supported by the framework, server components can fetch server-side data and avoid shipping their implementation JavaScript to the browser. Interactive components remain client components because they need state, effects, browser APIs, or event handlers. A server component should not be treated as a security boundary by itself; authorization must still occur on the server. <citation src="1"></citation>

A useful decision:

- **Server-rendered/server component:** static or data-display-heavy UI
- **Client component:** interaction, local state, browser APIs, event handlers
- **Hybrid page:** server-render the shell and data-heavy content; isolate interactive leaves

---

## 4. State design

The most important state question is not “Which state library should we use?” It is:

> “Who owns this data, and how long does it remain valid?”

### State categories

| State type | Examples | Recommended location |
|---|---|---|
| Local UI state | Modal open, selected tab, input focus | Component state |
| Form state | Draft values, validation errors | Form library or component state |
| URL state | Search query, filters, pagination | URL/search params |
| Server state | Users, orders, notifications | Query/cache layer |
| Cross-feature client state | Theme, session summary, feature flags | Context or external store |
| Persistent client state | Preferences, drafts | Storage abstraction |
| Real-time state | Presence, collaboration cursors | WebSocket/subscription layer |

Server state is remotely owned, can become stale, and needs caching, refetching, deduplication, and invalidation. Avoid copying it into a second global store unless you intentionally need a draft, derived projection, or conflict-resolution model. <citation src="1"></citation>

### State placement heuristic

Keep state as close as possible to where it is used.

Move it upward only when:

- Multiple siblings need it
- It must survive navigation
- It must be represented in the URL
- It must be shared across independently mounted features

Use Context for low-frequency cross-cutting concerns such as theme, locale, or dependency injection. Context is not automatically an efficient high-frequency state store: changes can cause all consumers in the provider subtree to update.

### Example: typed reducer

```tsx
type FilterState = {
  query: string;
  status: "all" | "open" | "closed";
};

type FilterAction =
  | { type: "queryChanged"; query: string }
  | { type: "statusChanged"; status: FilterState["status"] }
  | { type: "reset" };

function filterReducer(
  state: FilterState,
  action: FilterAction
): FilterState {
  switch (action.type) {
    case "queryChanged":
      return { ...state, query: action.query };
    case "statusChanged":
      return { ...state, status: action.status };
    case "reset":
      return { query: "", status: "all" };
  }
}
```

Discriminated unions make illegal action shapes difficult to represent and allow TypeScript to narrow the action based on its `type`. TypeScript’s narrowing is driven by runtime checks such as `typeof`, equality comparisons, `in`, and discriminant fields. <citation src="5,7"></citation>

---

## 5. API and data contracts

A senior answer should distinguish between:

- **Transport types:** what the API sends
- **Domain types:** what the application means
- **View-model types:** what a component needs to render

Do not assume that a TypeScript interface validates runtime JSON. Types disappear at runtime. Validate untrusted API responses at the boundary.

```ts
type User = {
  id: string;
  name: string;
  role: "admin" | "member";
};

type ApiResponse<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: string; message: string } };

async function getUser(id: string): Promise<ApiResponse<User>> {
  const response = await fetch(`/api/users/${encodeURIComponent(id)}`);

  if (!response.ok) {
    return {
      ok: false,
      error: {
        code: `HTTP_${response.status}`,
        message: "Request failed",
      },
    };
  }

  const raw: unknown = await response.json();

  // In production, validate raw with a runtime schema validator.
  return { ok: true, data: raw as User };
}
```

### API design considerations

Discuss:

- Pagination: cursor-based pagination is usually more stable for changing datasets
- Filtering and sorting
- Partial responses or field selection
- Cache headers and ETags
- Idempotency keys for retryable writes
- Request correlation IDs
- Versioning and backward compatibility
- Consistent error envelopes
- Authorization at the resource level

For a write operation, model states explicitly:

```ts
type SaveState =
  | { status: "idle" }
  | { status: "saving" }
  | { status: "saved"; savedAt: string }
  | { status: "error"; message: string };
```

This is clearer than several loosely related booleans such as `isSaving`, `hasError`, and `isSaved`.

---

## 6. Data fetching and caching

A robust client data layer should answer:

- When is data fetched?
- How long is it fresh?
- Can requests be deduplicated?
- What happens during refetch?
- How are writes invalidated?
- Can stale data be displayed?
- Can requests be cancelled?

Recommended behavior:

```text
Component mounts
  → check cache
  → render cached data if available
  → fetch if stale
  → update cache
  → notify subscribers
```

### Query keys

Query keys should include every input that changes the result:

```ts
const queryKey = ["orders", { customerId, status, cursor }];
```

If `status` changes but the key does not, the UI may display incorrect cached data.

### Request cancellation

```ts
async function fetchSearch(
  query: string,
  signal: AbortSignal
): Promise<SearchResult[]> {
  const response = await fetch(
    `/api/search?q=${encodeURIComponent(query)}`,
    { signal }
  );

  if (!response.ok) throw new Error("Search failed");
  return response.json();
}
```

Cancel obsolete searches to avoid race conditions where an older response overwrites a newer one.

### Optimistic updates

Use optimistic updates when:

- The operation is likely to succeed
- Reversal is understandable
- The UI benefits substantially from low perceived latency

You need:

1. A temporary client update
2. A request with an idempotency strategy
3. Reconciliation with the server response
4. Rollback or error presentation
5. Conflict handling if another client changed the record

For overlapping edits, do not blindly restore an old snapshot after a failure; that can erase a newer successful change. Track operations or use server versions and reconcile by identity/order. <citation src="1"></citation>

---

## 7. TypeScript topics interviewers expect

### `unknown` versus `any`

Use `unknown` for untrusted values:

```ts
function parse(value: unknown): string {
  if (typeof value !== "string") {
    throw new Error("Expected a string");
  }
  return value;
}
```

- `any` disables useful checking
- `unknown` requires narrowing before use

### Union types

```ts
type Result<T> =
  | { status: "success"; value: T }
  | { status: "failure"; error: Error };

function unwrap<T>(result: Result<T>): T {
  if (result.status === "success") return result.value;
  throw result.error;
}
```

### Generics

Use generics when a relationship between inputs and outputs matters:

```ts
function mapById<T extends { id: string }>(
  items: T[]
): Record<string, T> {
  return Object.fromEntries(items.map(item => [item.id, item]));
}
```

Generics provide reusable APIs while preserving the caller’s specific type. TypeScript supports constraints such as `T extends { id: string }` to preserve required relationships. <citation src="8"></citation>

### Utility types

```ts
type User = {
  id: string;
  name: string;
  email: string;
  role: string;
};

type UserPatch = Partial<Pick<User, "name" | "email">>;
type PublicUser = Omit<User, "email">;
```

Useful utilities include:

- `Pick`
- `Omit`
- `Partial`
- `Required`
- `Readonly`
- `Record`
- `ReturnType`
- `Parameters`
- `Awaited`

TypeScript provides these as standard type transformations. <citation src="6"></citation>

### Type-level versus runtime safety

Explain this explicitly:

> TypeScript catches many developer mistakes at compile time, but it cannot prove that network data, local storage, user input, or third-party JavaScript matches the declared type.

Use runtime validation at boundaries and convert external data into trusted domain objects.

### Component props

Prefer precise props:

```tsx
type ButtonProps =
  | {
      variant: "link";
      href: string;
      onClick?: never;
    }
  | {
      variant: "button";
      onClick: () => void;
      href?: never;
    };

function Action(props: ButtonProps) {
  if (props.variant === "link") {
    return <a href={props.href}>Open</a>;
  }

  return <button onClick={props.onClick}>Submit</button>;
}
```

This prevents impossible combinations such as a component that is simultaneously a link and a button.

---

## 8. Performance design

Discuss performance in terms of measurable bottlenecks, not “React is slow.”

### Main sources of frontend cost

- JavaScript download and parse time
- Unnecessary component renders
- Large DOM trees
- Expensive calculations
- Network waterfalls
- Image and font loading
- Main-thread blocking
- Large lists
- Excessive hydration
- Memory leaks

### High-value techniques

**Reduce work**

- Split routes and heavy features
- Lazy-load editors, charts, and modals
- Virtualize large lists
- Avoid rendering hidden large subtrees
- Use pagination or incremental loading

**Reduce repeated work**

- Keep state local
- Use stable keys
- Memoize only measured expensive work
- Pass primitive or stable props to memoized components
- Use selectors that return stable or narrowly scoped values

**Improve network behavior**

- Start independent requests in parallel
- Prefetch likely next routes
- Cache immutable assets aggressively
- Deduplicate requests
- Compress responses
- Use responsive images

**Protect responsiveness**

- Debounce expensive search
- Use cancellation
- Move heavy computation to a worker when appropriate
- Defer non-critical work
- Use transitions for non-urgent UI updates

### `useMemo` and `useCallback`

They are optimization tools, not correctness tools.

Bad reasoning:

> “I should wrap every function in `useCallback`.”

Better reasoning:

> “This callback is passed to a memoized child and changes frequently enough to cause expensive work, so I will stabilize it and verify with profiling.”

### Virtualized list example

```tsx
type Row = { id: string; title: string };

type RowProps = {
  row: Row;
  onSelect: (id: string) => void;
};

const RowView = React.memo(function RowView({
  row,
  onSelect,
}: RowProps) {
  return (
    <button onClick={() => onSelect(row.id)}>
      {row.title}
    </button>
  );
});
```

For thousands of rows, this alone is insufficient: render only the visible window and maintain stable item identity.

---

## 9. Reliability and failure handling

A production design should distinguish:

- Loading
- Empty
- Partial data
- Recoverable error
- Fatal error
- Unauthorized
- Offline
- Stale-but-usable data

### Error boundaries

Error boundaries handle rendering failures in a subtree. They do not replace handling for failed fetches or rejected event-handler promises. Use both:

```text
Error boundary
  → catches render failure
Query/error state
  → handles API failure
Form state
  → handles validation failure
Global telemetry
  → records diagnostic context
```

### Retry policy

Retry only when appropriate:

- Retry network failures and some 5xx responses
- Avoid retrying validation or authorization failures
- Use exponential backoff
- Add jitter
- Cap retry attempts
- Respect idempotency for writes

### Offline behavior

For a read-heavy app:

- Show cached data
- Indicate stale/offline state
- Queue safe mutations only if the product supports it

For offline writes:

```text
User action
  → local operation log
  → optimistic UI
  → sync queue
  → retry with backoff
  → conflict resolution
  → mark resolved or request user action
```

Do not claim “offline support” without defining conflict behavior.

---

## 10. Security

Frontend security is not just hiding buttons.

Cover:

- Authentication and session expiry
- Server-side authorization
- XSS prevention
- CSRF strategy where cookies are used
- Secure handling of tokens
- Content Security Policy
- Input validation and output encoding
- Dependency and supply-chain risk
- Avoiding sensitive data in URLs, logs, or client storage
- Rate limiting and abuse prevention on the backend

A disabled button, hidden route, or client-side role check does not enforce authorization. Every server operation must authenticate the caller, authorize the specific action/resource, validate input, and define duplicate/retry behavior. <citation src="1"></citation>

---

## 11. Accessibility

Make accessibility part of the architecture rather than a final checklist:

- Semantic HTML before ARIA
- Keyboard navigation
- Visible focus states
- Labels and error announcements
- Correct dialog focus management
- Sufficient color contrast
- Reduced-motion support
- Screen-reader-friendly loading and status updates
- Accessible virtualized lists
- Touch target sizing

A component library should encode accessible defaults so individual feature teams do not repeatedly solve the same problems.

---

## 12. Testing strategy

Use the lowest-cost test that provides confidence.

| Test | Best for |
|---|---|
| Unit | Pure reducers, formatters, validation, domain rules |
| Component | User interactions and rendering behavior |
| Integration | Component + router + data layer |
| Contract | API request/response compatibility |
| End-to-end | Critical user journeys |
| Performance | Load time, interaction latency, rendering cost |
| Accessibility | Keyboard and semantic behavior |

Prefer behavior-oriented tests:

```tsx
it("shows a validation error when submitting an empty title", async () => {
  render(<CreateProjectForm />);

  await userEvent.click(screen.getByRole("button", { name: /create/i }));

  expect(
    await screen.findByText("Title is required")
  ).toBeVisible();
});
```

Avoid tests that lock in implementation details such as internal state names or component structure.

---

## 13. Real-time collaboration design

For chat, collaborative documents, dashboards, or presence:

```text
Initial HTTP snapshot
  → WebSocket connection
  → subscribe to resource
  → receive ordered events
  → update local cache
  → reconnect with last event/version
  → resync if history is unavailable
```

Discuss:

- Event ordering
- Reconnection
- Heartbeats
- Backpressure
- Duplicate events
- Missed events
- Authorization on subscriptions
- Conflict resolution
- Presence expiry
- Whether to use last-write-wins, version checks, OT, or CRDTs

For a simple editable record, optimistic concurrency may be enough:

```ts
type UpdateRequest = {
  id: string;
  version: number;
  patch: Record<string, unknown>;
};
```

The server rejects the update if the version is stale, and the client presents a merge or refresh flow.

---

## 14. Example design: scalable searchable order dashboard

### Requirements

- Search and filter orders
- Paginate through a large dataset
- Open an order detail view
- Update order status
- Support slow networks
- Enforce role-based access
- Provide usable empty and error states

### Architecture

```text
Browser
  ├─ Router and URL state
  ├─ Server-rendered dashboard shell
  ├─ Client filter controls
  ├─ Query/cache layer
  └─ Accessible virtualized table
        ↓
Backend-for-frontend
  ├─ Authentication
  ├─ Authorization
  ├─ Query normalization
  ├─ Pagination and filtering
  └─ API aggregation
        ↓
Order service → Database
```

### URL state

```text
/orders?status=open&query=acme&cursor=...
```

This enables:

- Deep links
- Browser navigation
- Shareable filtered views
- Refresh persistence

### API

```text
GET /api/orders?status=open&query=acme&cursor=abc

{
  "items": [...],
  "nextCursor": "def",
  "serverTime": "..."
}
```

```text
PATCH /api/orders/:id/status
Headers:
  Idempotency-Key: ...
Body:
  { "status": "shipped", "version": 7 }
```

### Client behavior

- Cache each filter combination
- Debounce search input
- Cancel obsolete searches
- Preserve previous results while fetching the next page
- Optimistically update status only if rollback is clear
- Invalidate or reconcile the affected order after success
- Show stale data with a refresh affordance during temporary failures

### Scaling concerns

- Cursor pagination instead of loading all orders
- Virtualized rows
- Server-side filtering and sorting
- Query indexes on common filter fields
- CDN caching only for data safe to cache
- Per-user authorization before returning results
- Telemetry for query latency and client render time

---

## 15. Common interview mistakes

- Jumping directly to Redux, Zustand, or another library
- Treating all state as one category
- Copying server data into multiple stores
- Assuming TypeScript validates API responses
- Using `useEffect` for derived values
- Ignoring URL state for filters and pagination
- Saying “use memoization” without identifying the bottleneck
- Ignoring loading, empty, error, and stale states
- Treating client-side authorization as security
- Forgetting duplicate submissions and retries
- Ignoring accessibility
- Designing for infinite scale without identifying actual constraints
- Giving a perfect architecture without an incremental delivery plan

---

## 16. Rapid-fire questions and strong answers

**How do you choose between local state, Context, and an external store?**  
Use local state by default. Use Context for low-frequency cross-cutting dependencies. Use an external store when many distant consumers need coordinated client state, selective subscriptions, or event/history semantics. Keep server state in a query/cache layer rather than treating it as ordinary UI state.

**When should data be in the URL?**  
When it affects navigation, sharing, reload persistence, search, filtering, sorting, or pagination.

**How do you prevent stale search results from overwriting current results?**  
Use a query key containing the search parameters and cancel obsolete requests with `AbortController`. Also ensure response application checks the current request identity.

**When would you use SSR?**  
For SEO-sensitive or first-render-sensitive pages. For highly interactive authenticated applications, use a hybrid approach and avoid shipping unnecessary server-rendering complexity to every screen.

**What does `React.memo` solve?**  
It can skip rerendering a component when its props are equal. It does not prevent rerenders caused by the component’s own state, context changes, unstable props, or parent-level architectural problems.

**How do you model API errors?**  
Use a normalized error shape with a stable machine-readable code, user-safe message, retryability, and correlation ID. Keep diagnostic details out of user-facing messages.

**How do you handle optimistic updates?**  
Update the visible cache immediately, send an idempotent request, reconcile with the authoritative response, and either roll back or mark the operation failed. For concurrent edits, use versions or operation-based reconciliation.

**What changes at 10× scale?**  
Usually list rendering, query volume, bundle size, cache strategy, backend pagination/indexing, real-time fanout, and observability—not necessarily the entire component architecture.

---

## 17. Final interview checklist

Before ending your design, confirm that you covered:

- Requirements and quality attributes
- Main user flows
- Component and feature boundaries
- Rendering strategy
- API and data contracts
- State ownership
- Caching and invalidation
- Loading, empty, error, and stale states
- Performance bottlenecks and measurements
- Accessibility
- Authentication and authorization
- Testing
- Observability
- Offline or real-time behavior, if relevant
- Tradeoffs
- Incremental rollout and future scaling

The strongest answers are not the ones with the most technologies. They are the ones that clearly identify **ownership, consistency, failure behavior, and tradeoffs** at every boundary.
