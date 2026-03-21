# Redux Saga with React — TypeScript Reference Guide

> A comprehensive reference for building React applications with Redux Toolkit (RTK), Redux Saga, and TypeScript.
> Testing uses the native Node.js test runner (`node:test`) with a BDD style (`describe`/`it`).

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Project Setup](#project-setup)
3. [RTK Slices with TypeScript](#rtk-slices-with-typescript)
4. [createAction — Decoupled Action Creators](#createaction--decoupled-action-creators)
5. [Sagas — Effects, Workers, and Watchers](#sagas--effects-workers-and-watchers)
6. [rootSaga](#rootsaga)
7. [rootReducer](#rootreducer)
8. [Store Configuration](#store-configuration)
9. [Connecting React Components](#connecting-react-components)
10. [Testing with Node.js Native Test Runner (BDD)](#testing-with-nodejs-native-test-runner-bdd)
11. [UnknownAction — The Modern Action Base Type](#unknownaction--the-modern-action-base-type)
12. [Common Gotchas](#common-gotchas)
13. [File Placement Guidelines](#file-placement-guidelines)
14. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Architecture Overview

```
React Component
  │  dispatch(action)
  ▼
Redux Store ──middleware──▶ Redux Saga (intercepts actions)
  │                              │
  │  reducer updates state       │  yields effects (call, put, select …)
  ▼                              ▼
RTK Slice Reducer          Side-effect execution (API calls, delays, etc.)
  │                              │
  │                              │  put(slice.actions.success(data))
  ▼                              ▼
Updated State ◀──────────────────┘
  │
  ▼
React re-renders via useSelector
```

Key pieces and how they connect:

|       Piece      |      Role                                                                            |
|------------------|--------------------------------------------------------------------------------------|
| **RTK Slice**    | Defines state shape, reducers, and auto-generated action creators                    |
| **createAction** | Standalone action creators used as saga triggers (not handled by a reducer directly) |
| **Worker Saga**  | Generator function that performs the side-effect logic                               |
| **Watcher Saga** | Listens for dispatched actions and forks worker sagas                                |
| **rootSaga**     | Combines all watcher sagas via `all` / `fork`                                        |
| **rootReducer**  | Combines all slice reducers via `combineSlices` or `combineReducers`                 |
| **Store**        | Created with `configureStore`, wired with saga middleware                            |

---

## Project Setup

```bash
npm install @reduxjs/toolkit react-redux redux-saga
npm install -D typescript @types/react @types/react-dom
```

### tsconfig.json — relevant compiler options

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "jsx": "react-jsx",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2022"
  }
}
```

---

## RTK Slices with TypeScript

A slice owns a piece of state, its reducers, and the action creators derived from those reducers.

```typescript
// src/features/users/usersSlice.ts

import { createSlice, type PayloadAction } from "@reduxjs/toolkit";

// ── Types ────────────────────────────────────────────────────────
export interface User {
  id: number;
  name: string;
  email: string;
}

export interface UsersState {
  list: User[];
  loading: boolean;
  error: string | null;
}

// ── Initial state ────────────────────────────────────────────────
const initialState: UsersState = {
  list: [],
  loading: false,
  error: null,
};

// ── Slice ────────────────────────────────────────────────────────
const usersSlice = createSlice({
  name: "users",
  initialState,
  reducers: {
    fetchUsersStart(state): void {
      state.loading = true;
      state.error = null;
    },
    fetchUsersSuccess(state, action: PayloadAction<User[]>): void {
      state.list = action.payload;
      state.loading = false;
    },
    fetchUsersFailure(state, action: PayloadAction<string>): void {
      state.loading = false;
      state.error = action.payload;
    },
  },
});

export const { fetchUsersStart, fetchUsersSuccess, fetchUsersFailure } =
  usersSlice.actions;

export default usersSlice.reducer;
```

### Why typed `PayloadAction<T>`?

`PayloadAction<T>` narrows `action.payload` to `T` at compile time. Without it you get `any`, which defeats the purpose of TypeScript.

---

## createAction — Decoupled Action Creators

`createAction` produces standalone actions that are **not** tied to any slice reducer. They serve as **saga trigger points** — dispatched by components, intercepted by watcher sagas.

```typescript
// src/features/users/usersActions.ts

import { createAction } from "@reduxjs/toolkit";

// Trigger: component dispatches this; a saga watches for it.
// The generic parameter types the payload.
export const fetchUsersRequested = createAction<void>("users/fetchRequested");

export const deleteUserRequested = createAction<{ userId: number }>(
  "users/deleteRequested"
);
```

### When to use `createAction` vs slice actions

|     Use case                           |     Mechanism                                   |
|----------------------------------------|-------------------------------------------------|
| State mutation (reducer logic)         | Slice actions (`usersSlice.actions.*`)          |
| Saga trigger (side-effect entry point) | `createAction` — no reducer handles it directly |

This separation keeps reducers pure and sagas responsible for orchestration.

---

## Sagas — Effects, Workers, and Watchers

### Core effects at a glance

```typescript
import {
  call,       // Invoke a function (blocking)
  put,        // Dispatch an action
  takeLatest, // Fork worker on latest matching action (cancels previous)
  takeEvery,  // Fork worker on every matching action
  select,     // Read from the Redux store
  fork,       // Non-blocking fork of a saga
  all,        // Run multiple effects in parallel
  take,       // Wait for a specific action (blocking)
  delay,      // Pause execution (like setTimeout)
  cancel,     // Cancel a forked task
  cancelled,  // Check if the current saga was cancelled
} from "redux-saga/effects";
```

### Typing saga return values

Redux Saga generators yield **effect descriptors**, not the actual values. TypeScript cannot infer the resolved value from a `yield`. You must annotate the return type explicitly.

```typescript
// ❌ TypeScript infers `unknown` for `response`
const response = yield call(api.fetchUsers);

// ✅ Cast the yield expression
const response = (yield call(api.fetchUsers)) as User[];

// ✅ Alternative: use a typed wrapper (see Gotchas section)
```

### Worker saga

```typescript
// src/features/users/usersSaga.ts

import { call, put } from "redux-saga/effects";
import type { SagaReturnType } from "redux-saga/effects";
import {
  fetchUsersStart,
  fetchUsersSuccess,
  fetchUsersFailure,
} from "./usersSlice";
import { fetchUsersRequested } from "./usersActions";
import * as api from "../../services/userApi";
import type { User } from "./usersSlice";

// ── Worker ───────────────────────────────────────────────────────
export function* fetchUsersSaga(): Generator {
  try {
    yield put(fetchUsersStart());

    // `call` creates a descriptor — the middleware executes the actual call.
    const users = (yield call(api.getUsers)) as User[];

    yield put(fetchUsersSuccess(users));
  } catch (error: unknown) {
    const message =
      error instanceof Error ? error.message : "Unknown error";
    yield put(fetchUsersFailure(message));
  }
}
```

### Watcher saga

```typescript
// src/features/users/usersSaga.ts  (continued)

import { takeLatest } from "redux-saga/effects";

export function* watchFetchUsers(): Generator {
  // takeLatest auto-cancels any in-flight fetchUsersSaga when a new
  // fetchUsersRequested action arrives.
  yield takeLatest(fetchUsersRequested.type, fetchUsersSaga);
}
```

### `takeLatest` vs `takeEvery` vs `take`

| Helper | Behaviour |
|---|---|
| `takeLatest` | Cancels previous running worker, runs only the latest |
| `takeEvery` | Runs a new worker for every dispatched action (no cancellation) |
| `take` | Blocks until one matching action is dispatched (manual loop needed) |

---

## rootSaga

Combines all watcher sagas into a single entry point.

```typescript
// src/sagas/rootSaga.ts

import { all, fork } from "redux-saga/effects";
import { watchFetchUsers } from "../features/users/usersSaga";
import { watchAuth } from "../features/auth/authSaga";

export default function* rootSaga(): Generator {
  yield all([
    fork(watchFetchUsers),
    fork(watchAuth),
    // Add more watchers here as features grow
  ]);
}
```

### Why `fork` inside `all`?

- `fork` starts each watcher as a **non-blocking** attached task.
- `all` waits for all forked tasks — since watchers run indefinitely, `rootSaga` stays alive for the app's lifetime.
- If any forked saga throws an unhandled error, the parent (`rootSaga`) is also cancelled. Wrap individual watchers in try/catch if you need fault isolation.

---

## rootReducer

```typescript
// src/store/rootReducer.ts

import { combineReducers } from "@reduxjs/toolkit";
import usersReducer from "../features/users/usersSlice";
import authReducer from "../features/auth/authSlice";

const rootReducer = combineReducers({
  users: usersReducer,
  auth: authReducer,
});

export default rootReducer;
```

---

## Store Configuration

```typescript
// src/store/store.ts

import { configureStore } from "@reduxjs/toolkit";
import createSagaMiddleware from "redux-saga";
import rootReducer from "./rootReducer";
import rootSaga from "../sagas/rootSaga";

// 1. Create the saga middleware instance
const sagaMiddleware = createSagaMiddleware();

// 2. Configure the store with RTK
const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      // Disable thunk — sagas handle all side effects.
      thunk: false,
      // redux-saga uses non-serializable effect objects internally.
      // Disable the serializable check to avoid false warnings.
      serializableCheck: false,
    }).concat(sagaMiddleware),
});

// 3. Run the root saga AFTER the store is created
sagaMiddleware.run(rootSaga);

// 4. Infer types from the store itself — the official recommended approach.
//    These update automatically as you add slices or middleware.
export type AppStore = typeof store;
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export { store };
```

### Why `Dispatch<UnknownAction>` is correct

`Dispatch<UnknownAction>` is the *input constraint*, not the output type. It means "accepts any object with a `type: string`". It does not erase payload types at the call site:

```typescript
// From redux source
interface Dispatch<A extends Action = UnknownAction> {
  <T extends A>(action: T): T
}

// When you call dispatch, TypeScript narrows T to the specific action type:
dispatch(fetchUsersRequested());  // T = { type: "users/fetchRequested" }
dispatch(fetchUsersSuccess([]));  // T = PayloadAction<User[]>
// Full payload type safety — UnknownAction is just the floor, not the ceiling.
```

RTK action creators (`createAction`, `slice.actions.*`) produce fully typed `PayloadAction<T>` objects. The type safety comes from the action creator, not from the dispatch constraint. You never write `dispatch({ type: "some-string" })` by hand — you always use action creators, which are already typed.

This is the community-accepted approach: `thunk: false`, `typeof store.dispatch` for `AppDispatch`, typed hooks via `.withTypes()`, and `Dispatch<UnknownAction>` as the resulting type. The RTK maintainers consider this correct and intentional.

### Typed hooks (recommended pattern)

The plain `useDispatch` and `useSelector` hooks from `react-redux` know nothing about your store. `useDispatch` returns a generic `Dispatch`, and `useSelector` types state as `unknown`. You'd have to annotate every call site manually:

```typescript
// ❌ Without typed hooks — repetitive and error-prone
const dispatch = useDispatch<AppDispatch>();
const users = useSelector((state: RootState) => state.users.list);
```

The typed hooks pattern creates pre-bound aliases that carry your `RootState` and `AppDispatch` types automatically. Define them once, import everywhere — no per-component annotations, no risk of forgetting a generic parameter.

```typescript
// src/store/hooks.ts

import { useDispatch, useSelector, useStore } from "react-redux";
import type { AppDispatch, RootState } from "./store";

// .withTypes() was added in react-redux v9.1.0.
// It returns a new hook function pre-bound to your store types.
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppStore = useStore.withTypes<AppStore>();
```

> The older pattern you may see in tutorials uses type-assertion aliases:
> ```typescript
> // ⚠️ Previous approach — still works, but .withTypes() is now preferred
> export const useAppDispatch: () => AppDispatch = useDispatch;
> export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
> ```
> Both produce the same result. `.withTypes()` is the approach recommended by the [official React-Redux TypeScript docs](https://react-redux.js.org/using-react-redux/usage-with-typescript) as of v9.1.0 — it reads more naturally and doesn't require importing `TypedUseSelectorHook`.

Why this matters:

- `useAppDispatch()` returns `AppDispatch`, so dispatching saga trigger actions is type-safe without casts.
- `useAppSelector(state => state.users)` infers `state` as `RootState` — autocomplete works, typos are caught at compile time.
- `useAppStore()` gives you the full store instance inside a component. You rarely need it — `useSelector` and `useDispatch` cover most cases. It's useful when you need to read state outside the React render cycle (e.g. in an event handler or effect where you want a one-shot snapshot without subscribing to re-renders).
- If the store shape changes (slices added/removed), every component using these hooks gets updated type checking for free.
- It's the pattern recommended by the official RTK TypeScript docs.

### Where `AppStore`, `RootState` and `AppDispatch` come from

These types are inferred from the store itself — not from the reducer, and not hand-written. This is the approach recommended by the [official Redux TypeScript Quick Start](https://redux.js.org/tutorials/typescript-quick-start).

```typescript
// In store.ts — all three types derived from the store:
export type AppStore = typeof store;
export type RootState = ReturnType<typeof store.getState>;
// → { users: UsersState; auth: AuthState; ... }

export type AppDispatch = typeof store.dispatch;
// → Dispatch<UnknownAction>  (with thunk: false)
```

Why from the store and not the reducer?

- Deriving both `RootState` and `AppDispatch` from the store keeps them co-located and consistent.
- If you pass reducers directly to `configureStore` (without a separate `rootReducer`), `ReturnType<typeof store.getState>` is the only way to get the state type.
- `AppStore` itself is useful for typing `renderWithStore` test helpers and SSR scenarios.

#### `useAppStore` — when and why

`useSelector` subscribes to the store and re-renders on every change to the selected slice. Sometimes you just want to read the current state once without triggering re-renders — that's where `useAppStore` comes in.

```tsx
// src/features/export/ExportButton.tsx

import { useAppStore } from "../../store/hooks";

export function ExportButton(): JSX.Element {
  const store = useAppStore();

  function handleExport(): void {
    // Read state at the moment of click — no subscription, no re-render.
    const { users } = store.getState();
    const csv = users.list
      .map((u) => `${u.id},${u.name},${u.email}`)
      .join("\n");
    downloadCsv(csv);
  }

  return (
    <button type="button" onClick={handleExport}>
      Export users
    </button>
  );
}
```

Use `useAppStore` when:
- You need a point-in-time snapshot (exports, analytics events, form submission payloads).
- You're inside a callback that shouldn't subscribe to state changes.
- You need to pass the store to a non-React utility (e.g. a saga's `runSaga` in tests).

Avoid it for rendering — if the component needs to reflect state changes, use `useAppSelector` instead.

---

## Connecting React Components

```tsx
// src/features/users/UserList.tsx

import { useEffect } from "react";
import { useAppDispatch, useAppSelector } from "../../store/hooks";
import { fetchUsersRequested } from "./usersActions";

export function UserList(): JSX.Element {
  const dispatch = useAppDispatch();
  const { list, loading, error } = useAppSelector((state) => state.users);

  useEffect(() => {
    dispatch(fetchUsersRequested());
  }, [dispatch]);

  if (loading) return <p role="status">Loading users…</p>;
  if (error) return <p role="alert">{error}</p>;

  return (
    <ul aria-label="User list">
      {list.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Data flow recap

1. Component dispatches `fetchUsersRequested()` (a `createAction` action).
2. Saga middleware intercepts it via `takeLatest` in `watchFetchUsers`.
3. `fetchUsersSaga` runs: dispatches `fetchUsersStart`, calls the API, dispatches `fetchUsersSuccess` or `fetchUsersFailure`.
4. The slice reducer handles those actions and updates state.
5. `useAppSelector` triggers a re-render with the new state.

---

## Testing with Node.js Native Test Runner (BDD)

Node.js 20+ ships a stable built-in test runner via `node:test`. It supports `describe`/`it` blocks (BDD style), mocking, and assertions out of the box — no extra dependencies.

### Running tests

```bash
# Run all test files matching the default glob (**/test/**/*.test.{ts,js})
node --import tsx --test 'src/**/*.test.ts'

# With coverage (experimental)
node --import tsx --test --experimental-test-coverage 'src/**/*.test.ts'
```

> `tsx` (or `ts-node/esm`) is needed to execute TypeScript directly. Install with `npm i -D tsx`.

### Key imports

```typescript
import { describe, it, mock, beforeEach, afterEach } from "node:test";
import assert from "node:assert/strict";
```

---

### 10.1 Component Tests with React Testing Library (Primary Approach)

RTL tests are the focal point of your test suite. They render real components inside a real Redux store with saga middleware, interact with the DOM the way users do, and assert on visible outcomes. Everything else (saga unit tests, reducer tests) is supplemental.

#### Setup: test dependencies

```bash
npm install -D @testing-library/react @testing-library/user-event jsdom global-jsdom
```

> When running with `node:test`, you need a DOM environment. Import `global-jsdom/register` before your tests to polyfill `document`, `window`, etc.

#### Test store factory

Create a reusable helper that spins up a fresh store with saga middleware for each test. This avoids state leaking between tests.

```typescript
// src/test-utils/renderWithStore.tsx

import { type ReactElement } from "react";
import { render, type RenderOptions } from "@testing-library/react";
import { Provider } from "react-redux";
import {
  configureStore,
  type EnhancedStore,
  type Middleware,
  type UnknownAction,
} from "@reduxjs/toolkit";
import createSagaMiddleware from "redux-saga";
import rootReducer from "../store/rootReducer";
import type { RootState } from "../store/store";
import rootSaga from "../sagas/rootSaga";

// ── Types ────────────────────────────────────────────────────────
interface RenderWithStoreOptions extends Omit<RenderOptions, "wrapper"> {
  preloadedState?: Partial<RootState>;
}

interface RenderWithStoreResult extends ReturnType<typeof render> {
  store: EnhancedStore;
  /** Chronological list of every action dispatched through the store. */
  actionLog: UnknownAction[];
}

// ── Action-logging middleware ────────────────────────────────────
function createActionLogger(): {
  middleware: Middleware;
  actionLog: UnknownAction[];
} {
  const actionLog: UnknownAction[] = [];

  const middleware: Middleware = () => (next) => (action) => {
    actionLog.push(action as UnknownAction);
    return next(action);
  };

  return { middleware, actionLog };
}

// ── Factory ──────────────────────────────────────────────────────
export function renderWithStore(
  ui: ReactElement,
  { preloadedState, ...renderOptions }: RenderWithStoreOptions = {}
): RenderWithStoreResult {
  const sagaMiddleware = createSagaMiddleware();
  const { middleware: loggerMiddleware, actionLog } = createActionLogger();

  const store = configureStore({
    reducer: rootReducer,
    preloadedState: preloadedState as RootState,
    middleware: (getDefaultMiddleware) =>
      getDefaultMiddleware({ serializableCheck: false })
        .concat(sagaMiddleware)
        .concat(loggerMiddleware),
  });

  sagaMiddleware.run(rootSaga);

  function Wrapper({ children }: { children: React.ReactNode }): JSX.Element {
    return <Provider store={store}>{children}</Provider>;
  }

  return {
    ...render(ui, { wrapper: Wrapper, ...renderOptions }),
    store,
    actionLog,
  };
}
```

`actionLog` is a plain array that captures every action in dispatch order. It sits after the saga middleware in the chain, so saga-dispatched actions (`put(...)`) are captured too. Use it to assert that the right actions fired, in the right order, with the right payloads — without needing to spy on `dispatch`.

#### Example: testing the UserList component end-to-end

This test renders the component, lets the saga run against a mocked API, and asserts on what the user sees.

```typescript
// src/features/users/__tests__/UserList.test.tsx

import "global-jsdom/register";
import { describe, it, mock, afterEach } from "node:test";
import assert from "node:assert/strict";
import { screen, waitFor } from "@testing-library/react";
import { renderWithStore } from "../../../test-utils/renderWithStore";
import { UserList } from "../UserList";
import * as api from "../../../services/userApi";

describe("UserList", () => {
  afterEach(() => {
    mock.restoreAll();
  });

  describe("when the API returns users", () => {
    it("should render the user names after loading", async () => {
      // Arrange — mock the API so the saga resolves with known data
      mock.method(api, "getUsers", async () => [
        { id: 1, name: "Alice", email: "alice@test.io" },
        { id: 2, name: "Bob", email: "bob@test.io" },
      ]);

      // Act — render with a real store + saga middleware
      const { actionLog } = renderWithStore(<UserList />);

      // Assert — loading state appears first
      assert.ok(screen.getByRole("status"));

      // Assert — users appear after the saga completes
      await waitFor(() => {
        assert.ok(screen.getByText("Alice"));
        assert.ok(screen.getByText("Bob"));
      });

      // Assert — verify the full action sequence (full action objects, not just types)
      assert.deepStrictEqual(actionLog, [
        { type: "users/fetchRequested" },
        { type: "users/fetchUsersStart" },
        {
          type: "users/fetchUsersSuccess",
          payload: [
            { id: 1, name: "Alice", email: "alice@test.io" },
            { id: 2, name: "Bob", email: "bob@test.io" },
          ],
        },
      ]);
    });
  });

  describe("when the API fails", () => {
    it("should render the error message", async () => {
      mock.method(api, "getUsers", async () => {
        throw new Error("Network error");
      });

      const { actionLog } = renderWithStore(<UserList />);

      await waitFor(() => {
        assert.ok(screen.getByRole("alert"));
        assert.ok(screen.getByText("Network error"));
      });

      // Verify the failure action carries the error message
      assert.deepStrictEqual(actionLog, [
        { type: "users/fetchRequested" },
        { type: "users/fetchUsersStart" },
        { type: "users/fetchUsersFailure", payload: "Network error" },
      ]);
    });
  });

  describe("when preloaded state already has users", () => {
    it("should render them immediately without loading", () => {
      renderWithStore(<UserList />, {
        preloadedState: {
          users: {
            list: [{ id: 3, name: "Charlie", email: "c@test.io" }],
            loading: false,
            error: null,
          },
        },
      });

      // No loading indicator — data is already in the store
      assert.equal(screen.queryByRole("status"), null);
      assert.ok(screen.getByText("Charlie"));
    });
  });
});
```

#### Example: asserting dispatched actions from user interaction

Sometimes you want to verify that a user action dispatches the right Redux action. Access the store from the render helper and inspect its state or spy on dispatch.

```typescript
// src/features/users/__tests__/UserActions.test.tsx

import "global-jsdom/register";
import { describe, it, mock, afterEach } from "node:test";
import assert from "node:assert/strict";
import { screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { renderWithStore } from "../../../test-utils/renderWithStore";
import { DeleteUserButton } from "../DeleteUserButton";

// Assume DeleteUserButton dispatches deleteUserRequested({ userId })
// when clicked, and the saga calls the API then removes the user.

describe("DeleteUserButton", () => {
  afterEach(() => {
    mock.restoreAll();
  });

  it("should dispatch delete action and remove the user from the list", async () => {
    const user = userEvent.setup();

    const { store, actionLog } = renderWithStore(<DeleteUserButton userId={42} />, {
      preloadedState: {
        users: {
          list: [{ id: 42, name: "Doomed User", email: "d@test.io" }],
          loading: false,
          error: null,
        },
      },
    });

    // Act — click the delete button
    await user.click(screen.getByRole("button", { name: /delete/i }));

    // Assert — verify the delete action was dispatched with correct payload
    const deleteAction = actionLog.find((a) => a.type === "users/deleteRequested");
    assert.deepStrictEqual(deleteAction, {
      type: "users/deleteRequested",
      payload: { userId: 42 },
    });

    // Assert — check the store state was updated by the saga
    const state = store.getState();
    assert.equal(
      state.users.list.find((u: { id: number }) => u.id === 42),
      undefined
    );
  });
});
```

#### Example: testing a `useAppStore` consumer

```typescript
// src/features/export/__tests__/ExportButton.test.tsx

import "global-jsdom/register";
import { describe, it, mock, afterEach } from "node:test";
import assert from "node:assert/strict";
import { screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { renderWithStore } from "../../../test-utils/renderWithStore";
import { ExportButton } from "../ExportButton";

describe("ExportButton", () => {
  afterEach(() => {
    mock.restoreAll();
  });

  it("should read the current users from the store on click", async () => {
    const user = userEvent.setup();

    // Arrange — preload users into the store
    const { store } = renderWithStore(<ExportButton />, {
      preloadedState: {
        users: {
          list: [
            { id: 1, name: "Alice", email: "alice@test.io" },
            { id: 2, name: "Bob", email: "bob@test.io" },
          ],
          loading: false,
          error: null,
        },
      },
    });

    // Act — click export
    await user.click(screen.getByRole("button", { name: /export/i }));

    // Assert — store state was read (not mutated) by the handler.
    // The component uses store.getState() for a one-shot read,
    // so the state should remain unchanged.
    const state = store.getState();
    assert.equal(state.users.list.length, 2);
  });
});
```

#### Example: testing with `screen` queries (BDD style)

RTL's `screen` object is the recommended way to query the DOM. Pair it with BDD `describe`/`it` blocks for readable, behaviour-focused tests.

```typescript
describe("UserList (query patterns)", () => {
  it("should use accessible roles for loading and error states", async () => {
    mock.method(api, "getUsers", async () => []);
    renderWithStore(<UserList />);

    // getByRole — throws if not found (good for assertions)
    const loading = screen.getByRole("status");
    assert.ok(loading);

    await waitFor(() => {
      // queryByRole — returns null if not found (good for absence checks)
      assert.equal(screen.queryByRole("status"), null);
    });
  });

  it("should render an accessible list with aria-label", async () => {
    mock.method(api, "getUsers", async () => [
      { id: 1, name: "Eve", email: "eve@test.io" },
    ]);
    renderWithStore(<UserList />);

    await waitFor(() => {
      const list = screen.getByRole("list", { name: /user list/i });
      assert.ok(list);
    });
  });
});
```

#### When to use RTL vs other test types

| Test type | Use for | Priority |
|---|---|---|
| RTL component tests | User-visible behaviour, integration with store + sagas | **Primary** |
| Saga step-by-step tests | Complex saga logic, branching, error paths | Supplemental |
| Reducer unit tests | Edge cases in state transitions | Supplemental |
| `createAction` tests | Verifying action type strings / payloads | Supplemental |

---

### 10.2 Testing a Slice Reducer (Pure Function — No Mocking Needed)

Reducers are pure functions: given state + action → new state. Perfect for straightforward BDD assertions.

```typescript
// src/features/users/__tests__/usersSlice.test.ts

import { describe, it } from "node:test";
import assert from "node:assert/strict";
import reducer, {
  fetchUsersStart,
  fetchUsersSuccess,
  fetchUsersFailure,
  type UsersState,
} from "../usersSlice";

describe("usersSlice reducer", () => {
  const initialState: UsersState = {
    list: [],
    loading: false,
    error: null,
  };

  describe("when fetchUsersStart is dispatched", () => {
    it("should set loading to true and clear error", () => {
      const state = reducer(initialState, fetchUsersStart());
      assert.equal(state.loading, true);
      assert.equal(state.error, null);
    });
  });

  describe("when fetchUsersSuccess is dispatched", () => {
    it("should populate the list and stop loading", () => {
      const users = [{ id: 1, name: "Alice", email: "alice@test.io" }];
      const prev: UsersState = { ...initialState, loading: true };
      const state = reducer(prev, fetchUsersSuccess(users));

      assert.equal(state.loading, false);
      assert.deepStrictEqual(state.list, users);
    });
  });

  describe("when fetchUsersFailure is dispatched", () => {
    it("should store the error message and stop loading", () => {
      const prev: UsersState = { ...initialState, loading: true };
      const state = reducer(prev, fetchUsersFailure("Network error"));

      assert.equal(state.loading, false);
      assert.equal(state.error, "Network error");
    });
  });
});
```

---

### 10.3 Testing Sagas — Step-by-Step Generator Assertions

Sagas are generators. You can advance them with `.next()` and assert each yielded effect descriptor. This is the **unit-test** approach recommended by the official Redux Saga docs.

```typescript
// src/features/users/__tests__/usersSaga.test.ts

import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { call, put } from "redux-saga/effects";
import { fetchUsersSaga } from "../usersSaga";
import {
  fetchUsersStart,
  fetchUsersSuccess,
  fetchUsersFailure,
} from "../usersSlice";
import * as api from "../../../services/userApi";

describe("fetchUsersSaga", () => {
  describe("when the API call succeeds", () => {
    it("should dispatch start, call the API, then dispatch success", () => {
      const gen = fetchUsersSaga();

      // Step 1: should dispatch fetchUsersStart
      const step1 = gen.next();
      assert.deepStrictEqual(step1.value, put(fetchUsersStart()));

      // Step 2: should call api.getUsers
      const step2 = gen.next();
      assert.deepStrictEqual(step2.value, call(api.getUsers));

      // Step 3: provide mock response, should dispatch fetchUsersSuccess
      const mockUsers = [{ id: 1, name: "Bob", email: "bob@test.io" }];
      const step3 = gen.next(mockUsers);
      assert.deepStrictEqual(
        step3.value,
        put(fetchUsersSuccess(mockUsers))
      );

      // Step 4: generator should be done
      const step4 = gen.next();
      assert.equal(step4.done, true);
    });
  });

  describe("when the API call fails", () => {
    it("should dispatch start then dispatch failure with the error message", () => {
      const gen = fetchUsersSaga();

      // Advance past start
      gen.next(); // put(fetchUsersStart())
      gen.next(); // call(api.getUsers)

      // Throw an error into the generator (simulates API failure)
      const step = gen.throw(new Error("Server down"));
      assert.deepStrictEqual(
        step.value,
        put(fetchUsersFailure("Server down"))
      );

      const done = gen.next();
      assert.equal(done.done, true);
    });
  });
});
```

### Why step-by-step?

- No real middleware or store needed.
- Each `yield` produces a plain effect descriptor (a JS object). You compare objects, not behaviour.
- Extremely fast — no async, no network.

---

### 10.4 Mocking API Services with `node:test` mock

When you want to test a saga (or any module) that imports a real API module, use `mock.module` or `mock.fn` from `node:test`.

```typescript
// src/features/users/__tests__/usersSaga.integration.test.ts

import { describe, it, mock, beforeEach } from "node:test";
import assert from "node:assert/strict";

describe("fetchUsersSaga (integration with mocked API)", () => {
  // mock.fn creates a mock function you can inspect later
  const mockGetUsers = mock.fn(async (): Promise<
    Array<{ id: number; name: string; email: string }>
  > => {
    return [{ id: 1, name: "Mocked User", email: "mock@test.io" }];
  });

  beforeEach(() => {
    mockGetUsers.mock.resetCalls();
  });

  it("should call the API function exactly once per saga run", async () => {
    // In a real integration test you'd run the saga with `runSaga`
    // from redux-saga. Here we demonstrate the mock mechanics.
    await mockGetUsers();

    assert.equal(mockGetUsers.mock.callCount(), 1);
  });

  it("should propagate the resolved value", async () => {
    const result = await mockGetUsers();
    assert.deepStrictEqual(result, [
      { id: 1, name: "Mocked User", email: "mock@test.io" },
    ]);
  });
});
```

### Integration-style saga test with `runSaga`

For tests that exercise the full saga pipeline (middleware resolves effects for real), use `runSaga` from `redux-saga`:

```typescript
// src/features/users/__tests__/usersSaga.runSaga.test.ts

import { describe, it, mock, beforeEach } from "node:test";
import assert from "node:assert/strict";
import { runSaga } from "redux-saga";
import type { UnknownAction } from "@reduxjs/toolkit";
import { fetchUsersSaga } from "../usersSaga";
import * as api from "../../../services/userApi";
import type { User } from "../usersSlice";

describe("fetchUsersSaga (runSaga integration)", () => {
  let dispatched: UnknownAction[];

  beforeEach(() => {
    dispatched = [];
  });

  it("should dispatch success when API resolves", async () => {
    const fakeUsers: User[] = [
      { id: 1, name: "Test", email: "t@test.io" },
    ];

    // Mock the API at the module level
    const getUsersMock = mock.method(api, "getUsers", async () => fakeUsers);

    await runSaga(
      {
        dispatch: (action: UnknownAction) => {
          dispatched.push(action);
        },
        getState: () => ({ users: { list: [], loading: false, error: null } }),
      },
      fetchUsersSaga
    ).toPromise();

    // Verify full dispatched actions including payloads
    assert.deepStrictEqual(dispatched, [
      { type: "users/fetchUsersStart" },
      {
        type: "users/fetchUsersSuccess",
        payload: [{ id: 1, name: "Test", email: "t@test.io" }],
      },
    ]);

    // Verify mock was called
    assert.equal(getUsersMock.mock.callCount(), 1);

    // Restore original
    getUsersMock.mock.restore();
  });

  it("should dispatch failure when API rejects", async () => {
    const getUsersMock = mock.method(api, "getUsers", async () => {
      throw new Error("Timeout");
    });

    await runSaga(
      {
        dispatch: (action: UnknownAction) => {
          dispatched.push(action);
        },
        getState: () => ({ users: { list: [], loading: false, error: null } }),
      },
      fetchUsersSaga
    ).toPromise();

    assert.deepStrictEqual(dispatched, [
      { type: "users/fetchUsersStart" },
      { type: "users/fetchUsersFailure", payload: "Timeout" },
    ]);

    getUsersMock.mock.restore();
  });
});
```

---

### 10.5 Testing `createAction` Actions

Since `createAction` returns a plain action creator, testing is trivial:

```typescript
// src/features/users/__tests__/usersActions.test.ts

import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { fetchUsersRequested, deleteUserRequested } from "../usersActions";

describe("usersActions", () => {
  describe("fetchUsersRequested", () => {
    it("should create an action with the correct type", () => {
      const action = fetchUsersRequested();
      assert.equal(action.type, "users/fetchRequested");
    });
  });

  describe("deleteUserRequested", () => {
    it("should include the userId in the payload", () => {
      const action = deleteUserRequested({ userId: 42 });
      assert.deepStrictEqual(action.payload, { userId: 42 });
    });
  });
});
```

---

### 10.6 Where Mocking Makes the Most Sense

| What to mock | Why |
|---|---|
| API service functions (`api.getUsers`) | Avoid real HTTP calls; control responses |
| `Date.now`, `Math.random` | Deterministic output for snapshots |
| Third-party SDKs (analytics, auth providers) | Isolate your logic from external services |
| `window.localStorage` / `sessionStorage` | Not available in Node.js; must be mocked |
| Timers (`setTimeout`, `setInterval`) | Use `mock.timers` from `node:test` for fast-forwarding |

Things you should **not** mock:

- Slice reducers (they're pure — test with real inputs).
- Redux Saga effect creators (`call`, `put`) — assert against their descriptors instead.

---

## UnknownAction — The Modern Action Base Type

Starting with Redux 5.0 and RTK 2.0, the old `AnyAction` type is deprecated and replaced by `UnknownAction`. This is a significant TypeScript-level change that affects how you type reducers, middleware, sagas, and any code that handles generic actions.

### What changed and why

```typescript
// Redux 4 / RTK 1.x — AnyAction (deprecated)
// Allows accessing ANY property on action without type errors.
// This is unsafe: action.foo.bar.baz compiles fine even if those don't exist.
import { AnyAction } from "redux"; // ⚠️ deprecated

// Redux 5 / RTK 2.x — UnknownAction
// Only guarantees `action.type` exists. Everything else requires narrowing.
import { UnknownAction } from "@reduxjs/toolkit";
```

`AnyAction` extended `Action` with an index signature `[extraProps: string]: any`, which let you access arbitrary properties without TypeScript complaining. `UnknownAction` removes that index signature — you must narrow the type before accessing anything beyond `.type`.

### The type definition

```typescript
// From @reduxjs/toolkit (re-exported from redux)
interface Action<T extends string = string> {
  type: T;
}

// UnknownAction = Action with no extra index signature
// You can read .type, but nothing else without narrowing.
type UnknownAction = Action<string>;
```

### Common usage patterns

#### 1. Typing a hand-written reducer (outside of `createSlice`)

If you write a reducer manually (rare with RTK, but it happens), use `UnknownAction`:

```typescript
import { type UnknownAction } from "@reduxjs/toolkit";

interface CounterState {
  value: number;
}

const initialState: CounterState = { value: 0 };

function counterReducer(
  state: CounterState = initialState,
  action: UnknownAction
): CounterState {
  // action.payload doesn't exist on UnknownAction — you must narrow first
  switch (action.type) {
    case "counter/increment":
      return { value: state.value + 1 };
    case "counter/add": {
      // Narrow with a type guard before accessing payload
      if (hasPayload<number>(action)) {
        return { value: state.value + action.payload };
      }
      return state;
    }
    default:
      return state;
  }
}

// Reusable type guard for actions with a payload
function hasPayload<T>(action: UnknownAction): action is UnknownAction & { payload: T } {
  return "payload" in action;
}
```

#### 2. Typing dispatched actions in saga `runSaga` tests

When collecting dispatched actions in a `runSaga` test, type the array as `UnknownAction[]`:

```typescript
import { type UnknownAction } from "@reduxjs/toolkit";

let dispatched: UnknownAction[];

// In your runSaga config:
dispatch: (action: UnknownAction) => {
  dispatched.push(action);
},
```

This replaces the old `Action[]` or `AnyAction[]` pattern and is what RTK 2.x expects.

#### 3. Custom middleware

```typescript
import {
  type Middleware,
  type UnknownAction,
  isAction,
} from "@reduxjs/toolkit";
import type { RootState } from "./store";

const loggingMiddleware: Middleware<{}, RootState> = (storeApi) => (next) => (action) => {
  // `action` is `unknown` in RTK 2.x middleware signatures.
  // Use the isAction type guard to narrow it.
  if (isAction(action)) {
    console.log("Dispatching:", action.type);
  }
  return next(action);
};
```

#### 4. Type guards with `isAction` and action matchers

RTK provides `isAction` as a built-in type guard, plus matchers from `createSlice` and `createAction`:

```typescript
import { isAction, type UnknownAction } from "@reduxjs/toolkit";
import { fetchUsersSuccess } from "./features/users/usersSlice";

function handleAction(action: unknown): void {
  // Guard 1: is it even an action?
  if (!isAction(action)) return;

  // action is now UnknownAction (has .type)
  console.log(action.type);

  // Guard 2: is it a specific action?
  if (fetchUsersSuccess.match(action)) {
    // action is now PayloadAction<User[]> — fully typed
    console.log(action.payload.length, "users loaded");
  }
}
```

The `.match()` method on RTK action creators is the idiomatic way to narrow from `UnknownAction` to a specific `PayloadAction<T>`.

#### 5. Saga workers receiving the triggering action

When a watcher passes the triggering action to a worker, type the parameter explicitly:

```typescript
import { type PayloadAction } from "@reduxjs/toolkit";
import { takeEvery, call, put } from "redux-saga/effects";

interface DeleteUserPayload {
  userId: number;
}

function* deleteUserSaga(
  action: PayloadAction<DeleteUserPayload>
): Generator {
  const { userId } = action.payload; // Fully typed
  yield call(api.deleteUser, userId);
  yield put(deleteUserSuccess(userId));
}

function* watchDeleteUser(): Generator {
  // takeEvery passes the matched action to the worker
  yield takeEvery(deleteUserRequested.type, deleteUserSaga);
}
```

If you used `UnknownAction` here instead, you'd need to narrow before accessing `.payload`. Using `PayloadAction<T>` directly is cleaner when you know the action shape.

### Migration from `AnyAction`

| Old (RTK 1.x / Redux 4) | New (RTK 2.x / Redux 5) |
|---|---|
| `import { AnyAction } from "redux"` | `import { UnknownAction } from "@reduxjs/toolkit"` |
| `action.anything` compiles | `action.anything` is a type error — narrow first |
| Index signature `[key: string]: any` | No index signature — only `.type` is guaranteed |
| No type guards needed | Use `isAction()`, `.match()`, or custom guards |

### When you'll encounter `UnknownAction` in practice

- Typing `dispatch` callback arrays in `runSaga` tests
- Writing custom middleware
- Hand-written reducers outside `createSlice`
- Generic utility functions that accept "any action"
- Migrating older codebases from Redux 4 to Redux 5

If you use `createSlice` and `PayloadAction<T>` consistently (which you should), you'll rarely need to interact with `UnknownAction` directly — RTK handles the narrowing for you inside slice reducers. It mainly surfaces in glue code, tests, and middleware.

---

## Common Gotchas

### 1. Forgetting to run `sagaMiddleware.run(rootSaga)` after store creation

```typescript
// ❌ This silently does nothing — sagas never start
const sagaMiddleware = createSagaMiddleware();
sagaMiddleware.run(rootSaga); // Too early — store doesn't exist yet
const store = configureStore({ /* ... */ });

// ✅ Correct order
const sagaMiddleware = createSagaMiddleware();
const store = configureStore({ /* ... */ });
sagaMiddleware.run(rootSaga); // After store is created
```

### 2. `yield` returns `unknown` in TypeScript

Generator functions in TypeScript type all `yield` expressions as `unknown`. You must cast:

```typescript
// ❌ Type error: Object is of type 'unknown'
const users = yield call(api.getUsers);
users.map(/* ... */); // Error

// ✅ Explicit cast
const users = (yield call(api.getUsers)) as User[];
```

There is no built-in way to make TypeScript infer yield types from saga effects. The `typed-redux-saga` package offers a `call` wrapper that improves this, but it adds a dependency.

### 3. Using `takeLatest` when you need `takeEvery`

`takeLatest` cancels the previous in-flight worker when a new action arrives. This is great for search/autocomplete but dangerous for fire-and-forget operations like "add to cart" where every dispatch should complete.

```typescript
// Search — use takeLatest (only the latest query matters)
yield takeLatest("search/requested", searchSaga);

// Add to cart — use takeEvery (every click should persist)
yield takeEvery("cart/addItem", addItemSaga);
```

### 4. Non-serializable values in actions

RTK's `serializableCheck` middleware will warn if you put non-serializable values (class instances, functions, Promises) in actions or state. When using saga middleware, you may need to disable or configure this check:

```typescript
middleware: (getDefaultMiddleware) =>
  getDefaultMiddleware({
    serializableCheck: {
      // Ignore specific action types or paths if needed
      ignoredActions: ["persist/PERSIST"],
    },
  }).concat(sagaMiddleware),
```

### 5. Generator function signature: `Generator` vs `SagaIterator`

Both work, but they communicate different things:

```typescript
import type { SagaIterator } from "redux-saga";

// Option A: Generic Generator (works, less specific)
function* mySaga(): Generator { /* ... */ }

// Option B: SagaIterator (communicates intent, slightly stricter)
function* mySaga(): SagaIterator { /* ... */ }
```

`SagaIterator` is an alias for `Generator<StrictEffect, void, any>`. Use it when you want to signal "this is a saga" in your type signatures.

### 6. Error handling — uncaught errors kill the saga tree

An unhandled error in a forked saga propagates up and cancels the parent (and siblings). Always wrap workers in try/catch:

```typescript
function* workerSaga(): Generator {
  try {
    // ... side effects
  } catch (error: unknown) {
    // Handle gracefully — dispatch a failure action
    yield put(someFailureAction(getErrorMessage(error)));
  }
}

// Utility for safe error message extraction
function getErrorMessage(error: unknown): string {
  if (error instanceof Error) return error.message;
  return String(error);
}
```

### 7. `select` returns the full state — type it

```typescript
import { select } from "redux-saga/effects";
import type { RootState } from "../../store/store";

function* mySaga(): Generator {
  // ❌ result is `unknown`
  const token = yield select((state) => state.auth.token);

  // ✅ Cast the yield
  const token = (yield select(
    (state: RootState) => state.auth.token
  )) as string | null;
}
```

### 8. Testing gotcha — `gen.next(value)` feeds the value to the *previous* yield

When stepping through a generator, the argument to `.next(value)` becomes the resolved value of the `yield` that just paused. This is a common source of off-by-one confusion:

```typescript
const gen = fetchUsersSaga();
gen.next();           // Executes up to first yield, returns its effect
gen.next();           // Feeds `undefined` to first yield, executes to second yield
gen.next(mockUsers);  // Feeds `mockUsers` to second yield (the `call` result)
```

### 9. `node:test` mock cleanup

Always restore mocks to avoid test pollution:

```typescript
import { describe, it, mock, afterEach } from "node:test";

describe("my suite", () => {
  afterEach(() => {
    mock.restoreAll(); // Restores all mocked methods/modules
  });

  // ... tests
});
```

### 10. Saga middleware order matters

`sagaMiddleware` must come after RTK's default middleware (thunk, serializable check, etc.) so that actions flow through the standard pipeline first:

```typescript
// ✅ .concat appends saga middleware at the end
middleware: (getDefaultMiddleware) =>
  getDefaultMiddleware({ serializableCheck: false }).concat(sagaMiddleware),

// ❌ Prepending can cause unexpected behavior
middleware: (getDefaultMiddleware) =>
  [sagaMiddleware, ...getDefaultMiddleware()],
```

---

## File Placement Guidelines

These are not rigid folder rules — they're placement principles for the Redux Saga–specific files discussed in this document. Adapt them to whatever project structure you already have.

### Store wiring lives next to your app entry point

`store.ts`, `rootReducer.ts`, `rootSaga.ts`, and `hooks.ts` are app-level glue. They don't belong to any single feature — they wire features together. Place them alongside `App.tsx` (or wherever your top-level component lives), since that's where the `<Provider store={store}>` wrapper is rendered. Keeping them close makes the dependency obvious: `App.tsx` imports `store.ts`, which imports `rootReducer.ts` and `rootSaga.ts`.

```
App.tsx            ← renders <Provider store={store}>
store.ts           ← configureStore + sagaMiddleware.run(rootSaga)
rootReducer.ts     ← combineReducers, exports RootState
rootSaga.ts        ← all/fork of watcher sagas
hooks.ts           ← useAppDispatch, useAppSelector, useAppStore
```

If your project already has an `app/` or `src/` root folder for the entry point, these files naturally land there. The point is proximity to `App.tsx`, not a specific folder name.

### Slice, actions, and saga stay together per feature

For each feature that uses sagas, you'll have three Redux Saga–related files. Keep them in the same place as the components they serve:

| File | What it contains | Why it's separate |
|---|---|---|
| `usersSlice.ts` | State shape, reducers, slice actions | Pure state transitions — no side effects, easy to test |
| `usersActions.ts` | `createAction` saga triggers | Decouples "what the UI requests" from "how the reducer updates" |
| `usersSaga.ts` | Worker + watcher sagas | Side-effect orchestration — imports from both slice and actions |

These three files form a unit. The saga imports from the slice (to dispatch state-changing actions) and from the actions file (to watch for triggers). Keeping them together means you can understand the full data flow of a feature without jumping across the project.

If a feature has no side effects, you only need the slice. Don't create empty saga or actions files preemptively.

### `rootSaga` and `rootReducer` are aggregation points

Every time you add a new feature with a saga, two files need an update:

1. `rootSaga.ts` — add a `fork(watchNewFeature)` to the `all([...])` block.
2. `rootReducer.ts` — add the new slice reducer to `combineReducers`.

This is the only cross-feature coupling in the saga setup. It's intentional — these files are the single place where you can see every feature that's wired into the store.

### Tests sit next to what they test

Place `UserList.test.tsx` next to `UserList.tsx`, `usersSaga.test.ts` next to `usersSaga.ts`, and so on. Short imports, easy to find, and when you delete a feature you delete its tests with it.

The one exception is `renderWithStore.tsx` (the shared test store factory). It's used across all feature tests, so place it in a shared location — a `test-utils/` folder next to your app entry point, or wherever your project keeps shared utilities.

### API service modules

The functions your sagas `call` (like `api.getUsers`) should live in their own module, separate from sagas and slices. This is what you mock in tests. Place them wherever your project keeps service/client code — the key is that sagas import from them, not the other way around.

### What to avoid

- **Saga and slice in the same file** — they have different responsibilities and different test strategies. Mixing them makes both harder to test and reason about.
- **A top-level `sagas/` folder separate from features** — this forces you to jump between `sagas/usersSaga.ts` and `features/users/usersSlice.ts` to understand one feature. Keep them together.
- **Components importing `store.ts` directly** — components should use `useAppDispatch` / `useAppSelector` from `hooks.ts` and dispatch actions from their feature's actions file. If a component needs the raw store, something is off (except for the `useAppStore` snapshot pattern described earlier).
- **Circular imports between features** — if `features/users/usersSaga.ts` imports from `features/auth/authSlice.ts`, you've coupled two features. Use shared actions or a mediator saga in `rootSaga.ts` instead.

---

## Quick Reference Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│  SLICE (usersSlice.ts)                                          │
│    createSlice → reducer + actions (fetchUsersStart, etc.)      │
│    Export: default reducer, named actions                       │
├─────────────────────────────────────────────────────────────────┤
│  ACTIONS (usersActions.ts)                                      │
│    createAction → saga triggers (fetchUsersRequested, etc.)     │
│    Not handled by any reducer directly                          │
├─────────────────────────────────────────────────────────────────┤
│  SAGA (usersSaga.ts)                                            │
│    Worker: fetchUsersSaga — call API, put success/failure       │
│    Watcher: watchFetchUsers — takeLatest(trigger, worker)       │
├─────────────────────────────────────────────────────────────────┤
│  ROOT SAGA (rootSaga.ts)                                        │
│    all([ fork(watcher1), fork(watcher2), ... ])                 │
├─────────────────────────────────────────────────────────────────┤
│  ROOT REDUCER (rootReducer.ts)                                  │
│    combineReducers({ users: usersReducer, auth: authReducer })  │
│    Export: RootState type                                       │
├─────────────────────────────────────────────────────────────────┤
│  STORE (store.ts)                                               │
│    configureStore + createSagaMiddleware                        │
│    sagaMiddleware.run(rootSaga) — AFTER store creation          │
│    Export: AppDispatch type, store                              │
├─────────────────────────────────────────────────────────────────┤
│  HOOKS (hooks.ts)                                               │
│    useAppDispatch = useDispatch<AppDispatch>                    │
│    useAppSelector = useSelector<RootState>                      │
├─────────────────────────────────────────────────────────────────┤
│  COMPONENT                                                      │
│    dispatch(fetchUsersRequested()) → saga → reducer → re-render │
└─────────────────────────────────────────────────────────────────┘
```

---

## Document Constraints (Preserve for Future Reference Docs)

This document was built under the following constraints. Apply them when creating similar advanced React reference guides:

1. **Language**: TypeScript — all function signatures, parameters, and return types must be explicitly typed.
2. **State management**: Redux Toolkit (RTK) slices + Redux Saga for side effects.
3. **Testing framework**: Node.js native test runner (`node:test`, stable in Node 20+).
4. **Testing style**: BDD (`describe`/`it` blocks) with `node:assert/strict`.
5. **Component testing (primary)**: React Testing Library (`@testing-library/react`) with `screen` queries and `userEvent` interactions. RTL tests are the focal point — they render components inside a real Redux store with saga middleware and assert on user-visible behaviour. All other test types (saga unit tests, reducer tests, action tests) are supplemental.
6. **Test store helper**: Required — provide a `renderWithStore` factory that creates a fresh `configureStore` + `sagaMiddleware` per test, accepts `preloadedState`, and wraps the component in `<Provider>`.
7. **DOM environment**: Use `jsdom` via `global-jsdom/register` for Node.js-based component tests.
8. **Mocking**: Use `mock.fn`, `mock.method`, and `mock.restoreAll` from `node:test`. Mock external services and I/O; do not mock pure functions or effect creators.
9. **Gotchas section**: Required — document TypeScript-specific pitfalls, common misconfigurations, and testing foot-guns.
10. **Architecture diagram**: Include a visual flow showing how components, store, middleware, sagas, and reducers connect.
11. **Code examples**: Must be self-contained and runnable. Use realistic (but generic) domain models.
12. **Accessibility**: React component examples should include appropriate ARIA attributes.
13. **No external test runner dependencies**: Avoid Jest, Mocha, Vitest, or any third-party test runner. RTL and `userEvent` are allowed as they are DOM utilities, not test runners.

---

*Sources consulted: [Redux Saga official docs](https://redux-saga.js.org), [Redux Toolkit TypeScript usage](https://redux-toolkit.js.org/usage/usage-with-typescript), [Node.js test runner docs](https://nodejs.org/docs/v20.18.3/api/test.html). Content was rephrased for compliance with licensing restrictions.*