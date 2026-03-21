# Redux-Observable with React — TypeScript Reference Guide

> A comprehensive reference for building React applications with Redux Toolkit (RTK), Redux-Observable, RxJS, and TypeScript.
> Testing uses the native Node.js test runner (`node:test`) with a BDD style (`describe`/`it`).

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Project Setup](#project-setup)
3. [RTK Slices with TypeScript](#rtk-slices-with-typescript)
4. [createAction — Decoupled Action Creators](#createaction--decoupled-action-creators)
5. [Epics — The Core Abstraction](#epics--the-core-abstraction)
6. [rootEpic](#rootepic)
7. [rootReducer](#rootreducer)
8. [Store Configuration](#store-configuration)
9. [Where `RootState` and `AppDispatch` Come From](#where-rootstate-and-appdispatch-come-from)
10. [Typed Hooks](#typed-hooks)
11. [Connecting React Components](#connecting-react-components)
12. [Testing with Node.js Native Test Runner (BDD)](#testing-with-nodejs-native-test-runner-bdd)
13. [UnknownAction — The Modern Action Base Type](#unknownaction--the-modern-action-base-type)
14. [Common Gotchas](#common-gotchas)
15. [File Placement Guidelines](#file-placement-guidelines)
16. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Architecture Overview

```
React Component
  │  dispatch(action)
  ▼
Redux Store ──middleware──▶ Redux-Observable (epic middleware)
  │                              │
  │  reducer updates state       │  action$ stream → RxJS operators → output actions
  ▼                              ▼
RTK Slice Reducer          Side-effect execution (API calls, timers, WebSockets, etc.)
  │                              │
  │                              │  map to slice actions (success / failure)
  ▼                              ▼
Updated State ◀──────────────────┘
  │
  ▼
React re-renders via useSelector
```

Key pieces and how they connect:

| Piece | Role |
|---|---|
| **RTK Slice** | Defines state shape, reducers, and auto-generated action creators |
| **createAction** | Standalone action creators used as epic triggers (not handled by a reducer directly) |
| **Epic** | Function that receives an Observable of actions and returns an Observable of actions |
| **rootEpic** | Combines all epics via `combineEpics` |
| **rootReducer** | Combines all slice reducers via `combineReducers` |
| **Store** | Created with `configureStore`, wired with epic middleware |

### How epics differ from sagas

| Aspect | Redux Saga | Redux-Observable |
|---|---|---|
| Abstraction | Generator functions yielding effect descriptors | RxJS Observable pipelines |
| Cancellation | `takeLatest` auto-cancels; manual `cancel` effect | `switchMap` auto-unsubscribes; `takeUntil` for manual |
| Concurrency | `takeEvery`, `takeLatest`, `fork` | `mergeMap`, `switchMap`, `concatMap`, `exhaustMap` |
| Testing | Step-by-step generator `.next()` assertions | Marble diagrams or subscribe-and-collect |
| Learning curve | Generators + saga effects API | RxJS operators (broader ecosystem) |

---

## Project Setup

```bash
npm install @reduxjs/toolkit react-redux redux-observable rxjs
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

Slices work identically to the saga guide — they own state, reducers, and auto-generated action creators.

```typescript
// usersSlice.ts

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

---

## createAction — Decoupled Action Creators

Same pattern as with sagas — `createAction` produces standalone actions that serve as epic trigger points.

```typescript
// usersActions.ts

import { createAction } from "@reduxjs/toolkit";

export const fetchUsersRequested = createAction<void>("users/fetchRequested");

export const deleteUserRequested = createAction<{ userId: number }>(
  "users/deleteRequested"
);
```

Epics filter the `action$` stream for these trigger actions. Slice actions handle state mutations.

---

## Epics — The Core Abstraction

An epic is a function with this signature:

```typescript
type Epic<
  Input extends Action = Action,
  Output extends Input = Input,
  State = void,
  Dependencies = any
> = (
  action$: Observable<Input>,
  state$: StateObservable<State>,
  dependencies: Dependencies
) => Observable<Output>;
```

- `action$` — an Observable stream of every action dispatched to the store.
- `state$` — a `StateObservable` that emits the current state after each reducer run. Access the latest value with `state$.value`.
- `dependencies` — an optional injection object (API clients, etc.) passed when creating the middleware.

Epics run after reducers. You cannot "swallow" an action — it always reaches the reducer first, then the epic sees it.

### Core RxJS operators for epics

```typescript
import { filter, map, switchMap, mergeMap, concatMap, exhaustMap,
         catchError, debounceTime, takeUntil, retry } from "rxjs/operators";
import { of, from, EMPTY } from "rxjs";
```

| Operator | Epic use case |
|---|---|
| `filter` | Match specific action types (or use `ofType`) |
| `switchMap` | Cancel previous in-flight request on new trigger (like `takeLatest`) |
| `mergeMap` | Run requests in parallel (like `takeEvery`) |
| `concatMap` | Queue requests sequentially |
| `exhaustMap` | Ignore new triggers while one is in-flight (good for form submits) |
| `catchError` | Handle errors without killing the epic stream |
| `debounceTime` | Debounce rapid triggers (search autocomplete) |
| `takeUntil` | Cancel on a specific action (e.g. navigation away) |

### `ofType` — the action filter

redux-observable provides `ofType` as a pipeable operator that narrows the action type:

```typescript
import { ofType } from "redux-observable";

// Filters action$ to only actions matching fetchUsersRequested.type
// and narrows the TypeScript type accordingly.
action$.pipe(
  ofType(fetchUsersRequested.type)
)
```

When using RTK `createAction`, you can also pass the action creator directly (redux-observable v2+):

```typescript
action$.pipe(ofType(fetchUsersRequested))
```

### Worker epic

```typescript
// usersEpic.ts

import { type Epic, ofType } from "redux-observable";
import { switchMap, map, catchError } from "rxjs/operators";
import { of, from } from "rxjs";
import type { RootState } from "./store";
import type { UnknownAction } from "@reduxjs/toolkit";
import { fetchUsersRequested } from "./usersActions";
import {
  fetchUsersStart,
  fetchUsersSuccess,
  fetchUsersFailure,
} from "./usersSlice";
import * as api from "./userApi";
import type { User } from "./usersSlice";

export const fetchUsersEpic: Epic<UnknownAction, UnknownAction, RootState> = (
  action$
) =>
  action$.pipe(
    ofType(fetchUsersRequested.type),
    switchMap(() =>
      // Emit start action, then the API call result
      from(api.getUsers()).pipe(
        map((users: User[]) => fetchUsersSuccess(users)),
        catchError((error: Error) =>
          of(fetchUsersFailure(error.message))
        )
      )
    )
  );
```

Note: unlike the saga version, the `fetchUsersStart` action needs to be dispatched separately. A common pattern is to either:

1. Dispatch it from the component alongside the trigger, or
2. Use `startWith` inside the epic:

```typescript
import { startWith } from "rxjs/operators";

export const fetchUsersEpic: Epic<UnknownAction, UnknownAction, RootState> = (
  action$
) =>
  action$.pipe(
    ofType(fetchUsersRequested.type),
    switchMap(() =>
      from(api.getUsers()).pipe(
        map((users: User[]) => fetchUsersSuccess(users)),
        catchError((error: Error) =>
          of(fetchUsersFailure(error.message))
        ),
        startWith(fetchUsersStart())
      )
    )
  );
```

### `switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`

| Operator | Behaviour | Saga equivalent |
|---|---|---|
| `switchMap` | Cancels previous inner Observable on new emission | `takeLatest` |
| `mergeMap` | Runs all inner Observables concurrently | `takeEvery` |
| `concatMap` | Queues — waits for previous to complete before starting next | No direct equivalent |
| `exhaustMap` | Ignores new emissions while inner Observable is active | `takeLeading` |

### Dependency injection

Instead of importing API modules directly, inject them via the middleware's `dependencies` option. This makes testing easier — you swap the dependency object instead of mocking modules.

```typescript
// usersEpic.ts (with dependency injection)

interface EpicDependencies {
  api: {
    getUsers: () => Promise<User[]>;
  };
}

export const fetchUsersEpic: Epic<
  UnknownAction,
  UnknownAction,
  RootState,
  EpicDependencies
> = (action$, _state$, { api }) =>
  action$.pipe(
    ofType(fetchUsersRequested.type),
    switchMap(() =>
      from(api.getUsers()).pipe(
        map((users: User[]) => fetchUsersSuccess(users)),
        catchError((error: Error) =>
          of(fetchUsersFailure(error.message))
        ),
        startWith(fetchUsersStart())
      )
    )
  );
```

---

## rootEpic

Combines all epics into a single entry point, analogous to `rootSaga`.

```typescript
// rootEpic.ts

import { combineEpics } from "redux-observable";
import { fetchUsersEpic } from "./usersEpic";
import { authEpic } from "./authEpic";

const rootEpic = combineEpics(
  fetchUsersEpic,
  authEpic
  // Add more epics here as features grow
);

export default rootEpic;
```

`combineEpics` merges the output Observables of all epics into a single stream. Every action emitted by any epic gets dispatched to the store.

---

## rootReducer

```typescript
// rootReducer.ts

import { combineReducers } from "@reduxjs/toolkit";
import usersReducer from "./usersSlice";
import authReducer from "./authSlice";

const rootReducer = combineReducers({
  users: usersReducer,
  auth: authReducer,
});

export default rootReducer;
```

---

## Store Configuration

```typescript
// store.ts

import { configureStore } from "@reduxjs/toolkit";
import { createEpicMiddleware } from "redux-observable";
import type { UnknownAction } from "@reduxjs/toolkit";
import rootReducer from "./rootReducer";
import rootEpic from "./rootEpic";
import * as api from "./userApi";

// ── Types for dependency injection ───────────────────────────────
export interface EpicDependencies {
  api: typeof api;
}

// 1. Create the epic middleware with typed generics.
//    We use ReturnType<typeof rootReducer> here because the store
//    (and therefore RootState) doesn't exist yet at this point.
const epicMiddleware = createEpicMiddleware<
  UnknownAction,
  UnknownAction,
  ReturnType<typeof rootReducer>,
  EpicDependencies
>({
  dependencies: { api },
});

// 2. Configure the store with RTK
const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      // Disable thunk — epics handle all side effects.
      thunk: false,
      // redux-observable uses non-serializable Observable objects internally.
      serializableCheck: false,
    }).concat(epicMiddleware),
});

// 3. Run the root epic AFTER the store is created
epicMiddleware.run(rootEpic);

// 4. Infer types from the store itself — the official recommended approach.
//    These are the canonical types the rest of the app should import.
export type AppStore = typeof store;
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export { store };
```

### Why `Dispatch<UnknownAction>` is correct

`Dispatch<UnknownAction>` is the *input constraint*, not the output type. When you call `dispatch(fetchUsersRequested())`, TypeScript narrows to the specific action type at the call site — full payload type safety. The `UnknownAction` constraint just means "accepts any object with a `type` string". The type safety comes from the action creators, not from the dispatch constraint.

This is the community-accepted approach. See the [saga guide](./redux-saga-react-guide.md#why-dispatchunknownaction-is-correct) for the full explanation.

---

## Typed Hooks

The plain `useDispatch` and `useSelector` hooks from `react-redux` know nothing about your store. The typed hooks pattern creates pre-bound aliases that carry your `RootState` and `AppDispatch` types automatically.

```typescript
// hooks.ts

import { useDispatch, useSelector, useStore } from "react-redux";
import type { AppDispatch, RootState } from "./store";

// .withTypes() was added in react-redux v9.1.0.
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppStore = useStore.withTypes<AppStore>();
```

- `useAppDispatch()` — returns `AppDispatch`, so dispatching epic trigger actions is type-safe.
- `useAppSelector(state => state.users)` — infers `state` as `RootState`, autocomplete works.
- `useAppStore()` — gives the full store instance for one-shot reads (exports, analytics) without subscribing to re-renders.

Place this file next to `store.ts` and `App.tsx`.

---

## Where `AppStore`, `RootState` and `AppDispatch` Come From

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

---

## Connecting React Components

```tsx
// UserList.tsx

import { useEffect } from "react";
import { useAppDispatch, useAppSelector } from "./hooks";
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

1. Component dispatches `fetchUsersRequested()`.
2. Reducer ignores it (no case for this action type).
3. Epic middleware passes it through `action$`. The `fetchUsersEpic` matches it via `ofType`.
4. The epic's `switchMap` calls the API, emits `fetchUsersStart()`, then `fetchUsersSuccess(users)` or `fetchUsersFailure(message)`.
5. Those actions hit the slice reducer, updating state.
6. `useAppSelector` triggers a re-render.

---

## Testing with Node.js Native Test Runner (BDD)

### Running tests

```bash
node --import tsx --test 'src/**/*.test.ts'
```

### Key imports

```typescript
import { describe, it, mock, beforeEach, afterEach } from "node:test";
import assert from "node:assert/strict";
```

### Test dependencies for RTL

```bash
npm install -D @testing-library/react @testing-library/user-event jsdom global-jsdom
```

---

### 12.1 Component Tests with React Testing Library (Primary Approach)

RTL tests are the focal point. They render components inside a real store with epic middleware and assert on user-visible behaviour.

#### Test store factory

```typescript
// renderWithStore.tsx

import { type ReactElement } from "react";
import { render, type RenderOptions } from "@testing-library/react";
import { Provider } from "react-redux";
import {
  configureStore,
  type EnhancedStore,
  type Middleware,
  type UnknownAction,
} from "@reduxjs/toolkit";
import { createEpicMiddleware } from "redux-observable";
import rootReducer from "./rootReducer";
import type { RootState } from "./store";
import rootEpic from "./rootEpic";
import * as api from "./userApi";
import type { EpicDependencies } from "./store";

// ── Types ────────────────────────────────────────────────────────
interface RenderWithStoreOptions extends Omit<RenderOptions, "wrapper"> {
  preloadedState?: Partial<RootState>;
  dependencies?: Partial<EpicDependencies>;
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
  { preloadedState, dependencies, ...renderOptions }: RenderWithStoreOptions = {}
): RenderWithStoreResult {
  const epicMiddleware = createEpicMiddleware<
    UnknownAction,
    UnknownAction,
    RootState,
    EpicDependencies
  >({
    dependencies: { api, ...dependencies } as EpicDependencies,
  });

  const { middleware: loggerMiddleware, actionLog } = createActionLogger();

  const store = configureStore({
    reducer: rootReducer,
    preloadedState: preloadedState as RootState,
    middleware: (getDefaultMiddleware) =>
      getDefaultMiddleware({ serializableCheck: false })
        .concat(epicMiddleware)
        .concat(loggerMiddleware),
  });

  epicMiddleware.run(rootEpic);

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

The `dependencies` option lets you inject mock API clients per test — no need for `mock.method` on the real module.

#### Example: testing UserList end-to-end

```typescript
// UserList.test.tsx

import "global-jsdom/register";
import { describe, it, mock, afterEach } from "node:test";
import assert from "node:assert/strict";
import { screen, waitFor } from "@testing-library/react";
import { renderWithStore } from "./renderWithStore";
import { UserList } from "./UserList";

describe("UserList", () => {
  afterEach(() => {
    mock.restoreAll();
  });

  describe("when the API returns users", () => {
    it("should render user names after loading", async () => {
      const mockApi = {
        getUsers: async () => [
          { id: 1, name: "Alice", email: "alice@test.io" },
          { id: 2, name: "Bob", email: "bob@test.io" },
        ],
      };

      const { actionLog } = renderWithStore(<UserList />, {
        dependencies: { api: mockApi },
      });

      assert.ok(screen.getByRole("status"));

      await waitFor(() => {
        assert.ok(screen.getByText("Alice"));
        assert.ok(screen.getByText("Bob"));
      });

      // Verify the full action sequence
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
      const mockApi = {
        getUsers: async () => {
          throw new Error("Network error");
        },
      };

      const { actionLog } = renderWithStore(<UserList />, {
        dependencies: { api: mockApi },
      });

      await waitFor(() => {
        assert.ok(screen.getByRole("alert"));
        assert.ok(screen.getByText("Network error"));
      });

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

      assert.equal(screen.queryByRole("status"), null);
      assert.ok(screen.getByText("Charlie"));
    });
  });
});
```

#### When to use RTL vs other test types

| Test type | Use for | Priority |
|---|---|---|
| RTL component tests | User-visible behaviour, integration with store + epics | **Primary** |
| Epic Observable tests | Complex stream logic, timing, cancellation | Supplemental |
| Reducer unit tests | Edge cases in state transitions | Supplemental |
| `createAction` tests | Verifying action type strings / payloads | Supplemental |

---

### 12.2 Testing a Slice Reducer (Supplemental)

Identical to the saga guide — reducers are pure functions.

```typescript
// usersSlice.test.ts

import { describe, it } from "node:test";
import assert from "node:assert/strict";
import reducer, {
  fetchUsersStart,
  fetchUsersSuccess,
  fetchUsersFailure,
  type UsersState,
} from "./usersSlice";

describe("usersSlice reducer", () => {
  const initialState: UsersState = { list: [], loading: false, error: null };

  describe("when fetchUsersSuccess is dispatched", () => {
    it("should populate the list and stop loading", () => {
      const users = [{ id: 1, name: "Alice", email: "alice@test.io" }];
      const prev: UsersState = { ...initialState, loading: true };
      const state = reducer(prev, fetchUsersSuccess(users));

      assert.equal(state.loading, false);
      assert.deepStrictEqual(state.list, users);
    });
  });
});
```

---

### 12.3 Testing Epics — Observable Subscribe-and-Collect

The most practical way to unit-test an epic: feed it a mock `action$`, subscribe to the output, and collect emitted actions.

```typescript
// usersEpic.test.ts

import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { Subject } from "rxjs";
import { StateObservable } from "redux-observable";
import type { UnknownAction } from "@reduxjs/toolkit";
import { fetchUsersEpic } from "./usersEpic";
import { fetchUsersRequested } from "./usersActions";
import {
  fetchUsersStart,
  fetchUsersSuccess,
  fetchUsersFailure,
} from "./usersSlice";
import type { RootState } from "./store";

function createMockState(overrides: Partial<RootState> = {}): StateObservable<RootState> {
  const state: RootState = {
    users: { list: [], loading: false, error: null },
    auth: { token: null },
    ...overrides,
  } as RootState;

  return new StateObservable(new Subject<RootState>(), state);
}

describe("fetchUsersEpic", () => {
  describe("when fetchUsersRequested is dispatched and API succeeds", () => {
    it("should emit start then success with the user data", async () => {
      const fakeUsers = [{ id: 1, name: "Test", email: "t@test.io" }];
      const deps = {
        api: { getUsers: async () => fakeUsers },
      };

      const action$ = new Subject<UnknownAction>();
      const state$ = createMockState();
      const output: UnknownAction[] = [];

      await new Promise<void>((resolve) => {
        fetchUsersEpic(action$, state$, deps).subscribe({
          next: (action) => output.push(action),
          complete: () => resolve(),
        });

        action$.next(fetchUsersRequested());
        action$.complete();
      });

      assert.deepStrictEqual(output, [
        fetchUsersStart(),
        fetchUsersSuccess(fakeUsers),
      ]);
    });
  });

  describe("when the API fails", () => {
    it("should emit start then failure with the error message", async () => {
      const deps = {
        api: {
          getUsers: async () => {
            throw new Error("Timeout");
          },
        },
      };

      const action$ = new Subject<UnknownAction>();
      const state$ = createMockState();
      const output: UnknownAction[] = [];

      await new Promise<void>((resolve) => {
        fetchUsersEpic(action$, state$, deps).subscribe({
          next: (action) => output.push(action),
          complete: () => resolve(),
        });

        action$.next(fetchUsersRequested());
        action$.complete();
      });

      assert.deepStrictEqual(output, [
        fetchUsersStart(),
        fetchUsersFailure("Timeout"),
      ]);
    });
  });
});
```

### Why subscribe-and-collect over marble testing?

Marble testing (`TestScheduler`) is powerful for time-dependent logic (debounce, throttle, delays). But for most epics that just do an async call and map the result, subscribe-and-collect is simpler, requires no extra setup, and reads more naturally in BDD blocks. Use marble tests when timing is the thing you're testing.

---

### 12.4 Testing Epics — Marble Diagrams (Time-Dependent Logic)

When your epic uses `debounceTime`, `delay`, `throttleTime`, or other time-based operators, use RxJS's `TestScheduler` to virtualize time.

```typescript
// searchEpic.test.ts

import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { TestScheduler } from "rxjs/testing";
import { StateObservable } from "redux-observable";
import { Subject } from "rxjs";
import type { UnknownAction } from "@reduxjs/toolkit";
import { searchEpic } from "./searchEpic";
import type { RootState } from "./store";

// Assume searchEpic debounces by 300ms then calls an API
// action$.pipe(
//   ofType(searchRequested.type),
//   debounceTime(300),
//   switchMap(action => from(api.search(action.payload)).pipe(...))
// )

describe("searchEpic (marble test)", () => {
  function createScheduler(): TestScheduler {
    return new TestScheduler((actual, expected) => {
      assert.deepStrictEqual(actual, expected);
    });
  }

  it("should debounce rapid search requests by 300ms", () => {
    createScheduler().run(({ hot, cold, expectObservable }) => {
      // Three rapid searches, only the last should survive debounce
      const action$ = hot<UnknownAction>("--a-b-c--------|", {
        a: { type: "search/requested", payload: "re" },
        b: { type: "search/requested", payload: "rea" },
        c: { type: "search/requested", payload: "reac" },
      });

      const state$ = new StateObservable(
        new Subject<RootState>(),
        { users: { list: [], loading: false, error: null } } as RootState
      );

      const deps = {
        api: {
          search: (_query: string) => cold<string[]>("--r|", { r: ["React"] }),
        },
      };

      const output$ = searchEpic(action$, state$, deps);

      // 300ms debounce after 'c' at frame 6, then 2-frame API response
      expectObservable(output$).toBe("-----------s----|", {
        s: { type: "search/success", payload: ["React"] },
      });
    });
  });
});
```

### Marble syntax quick reference

| Symbol | Meaning |
|---|---|
| `-` | One virtual time frame (typically 1ms in `TestScheduler`) |
| `a`, `b`, `c` | Emitted values (mapped via the values object) |
| `\|` | Observable completion |
| `#` | Observable error |
| `^` | Subscription point (hot observables) |
| `(abc)` | Grouped synchronous emissions |

---

### 12.5 Where Mocking Makes the Most Sense

| What to mock | Why |
|---|---|
| API service functions (via `dependencies`) | Avoid real HTTP calls; control responses |
| `Date.now`, `Math.random` | Deterministic output |
| WebSocket connections | Not available in test environment |
| Timers | Use `TestScheduler` to virtualize time instead of real `setTimeout` |

Things you should not mock:

- Slice reducers — they're pure, test with real inputs.
- RxJS operators — assert on the output stream, not the internals.
- `ofType` — it's a simple filter, not a side effect.

---

## UnknownAction — The Modern Action Base Type

Same as in the saga guide — Redux 5 / RTK 2 replaced `AnyAction` with `UnknownAction`. Only `.type` is guaranteed; everything else requires narrowing.

```typescript
import { type UnknownAction, isAction } from "@reduxjs/toolkit";

// Use UnknownAction for epic type parameters
const myEpic: Epic<UnknownAction, UnknownAction, RootState> = (action$) =>
  action$.pipe(/* ... */);

// Use isAction() and .match() for narrowing
if (fetchUsersSuccess.match(action)) {
  // action is now PayloadAction<User[]>
}
```

For full details, type guards, and migration from `AnyAction`, see the [Redux Saga guide's UnknownAction section](./redux-saga-react-guide.md#unknownaction--the-modern-action-base-type).

---

## Common Gotchas

### 1. Forgetting to run `epicMiddleware.run(rootEpic)` after store creation

```typescript
// ❌ Too early — store doesn't exist yet
const epicMiddleware = createEpicMiddleware();
epicMiddleware.run(rootEpic);
const store = configureStore({ /* ... */ });

// ✅ Correct order
const epicMiddleware = createEpicMiddleware();
const store = configureStore({ /* ... */ });
epicMiddleware.run(rootEpic);
```

### 2. Killing the epic stream with an uncaught error

If an error propagates to the top-level Observable of an epic, that epic dies permanently — no more actions will be processed by it. Always contain errors inside the inner Observable:

```typescript
// ❌ catchError on the outer pipe — epic dies on first error
action$.pipe(
  ofType(fetchUsersRequested.type),
  switchMap(() => from(api.getUsers())),
  map((users) => fetchUsersSuccess(users)),
  catchError((err) => of(fetchUsersFailure(err.message))) // Epic stops after this
);

// ✅ catchError inside switchMap — epic survives and handles next action
action$.pipe(
  ofType(fetchUsersRequested.type),
  switchMap(() =>
    from(api.getUsers()).pipe(
      map((users) => fetchUsersSuccess(users)),
      catchError((err) => of(fetchUsersFailure(err.message))) // Only inner stream dies
    )
  )
);
```

This is the single most common redux-observable bug. If `catchError` is on the outer pipe, the epic completes after the first error and silently stops working.

### 3. Epics run after reducers

Unlike middleware that can intercept actions before they reach reducers, epics see actions after the reducer has already processed them. You cannot prevent an action from reaching the reducer with an epic. If you need to transform an action before the reducer sees it, use a regular middleware.

### 4. `state$.value` vs subscribing to `state$`

```typescript
// ✅ Read current state snapshot (most common)
const token = state$.value.auth.token;

// ⚠️ Subscribing to state$ creates a new stream of state changes.
// Rarely needed — use state$.value for point-in-time reads.
state$.pipe(
  map((state) => state.auth.token),
  distinctUntilChanged()
);
```

### 5. TypeScript generics on `createEpicMiddleware`

If you don't provide generics, the middleware defaults to `Action` for input/output, which won't match `UnknownAction` from RTK 2. Always type it:

```typescript
// ❌ Defaults to Action — type mismatch with RTK 2 slices
const epicMiddleware = createEpicMiddleware();

// ✅ Explicit generics
const epicMiddleware = createEpicMiddleware<
  UnknownAction,
  UnknownAction,
  RootState,
  EpicDependencies
>();
```

### 6. `ofType` with `createAction` — string vs action creator

```typescript
// Both work, but passing the string is more explicit and always safe:
action$.pipe(ofType(fetchUsersRequested.type))  // ✅ string literal
action$.pipe(ofType(fetchUsersRequested))        // ✅ works in redux-observable v2+

// ❌ Don't pass the action creator's .toString() — it's the same as .type
// but reads confusingly
action$.pipe(ofType(fetchUsersRequested.toString()))
```

### 7. Multiple emissions from one epic invocation

Unlike sagas where each `put` is a separate yield, an epic's inner Observable can emit multiple actions synchronously. Be aware that all emitted actions hit the reducer before the next action from `action$` is processed:

```typescript
// This emits start, then success — both dispatch before the epic
// processes any new incoming action.
switchMap(() =>
  from(api.getUsers()).pipe(
    map((users) => fetchUsersSuccess(users)),
    startWith(fetchUsersStart())
  )
)
```

### 8. Importing `Observable` types for epic signatures

When typing epics explicitly, you need the right imports:

```typescript
import { type Epic } from "redux-observable";
import type { UnknownAction } from "@reduxjs/toolkit";
import type { RootState } from "./store";

// Full explicit signature
const myEpic: Epic<UnknownAction, UnknownAction, RootState, EpicDependencies> = (
  action$,
  state$,
  deps
) => { /* ... */ };
```

The four generic parameters are: `Input`, `Output`, `State`, `Dependencies`. If you omit them, TypeScript may infer overly broad types.

### 9. `node:test` mock cleanup

Always restore mocks to avoid test pollution:

```typescript
import { afterEach, mock } from "node:test";

afterEach(() => {
  mock.restoreAll();
});
```

### 10. Don't dispatch inside `subscribe`

```typescript
// ❌ Anti-pattern — bypasses the epic pipeline
myEpic(action$, state$, deps).subscribe((action) => {
  store.dispatch(action); // Manual dispatch — breaks the reactive model
});

// ✅ The epic middleware handles dispatching automatically.
// Just return actions from your epic — the middleware subscribes and dispatches.
```

---

## File Placement Guidelines

These are placement principles for redux-observable–specific files. Adapt them to your existing project structure.

### Store wiring lives next to your app entry point

`store.ts`, `rootReducer.ts`, `rootEpic.ts`, and `hooks.ts` are app-level glue. Place them alongside `App.tsx`, since that's where `<Provider store={store}>` is rendered.

```
App.tsx            ← renders <Provider store={store}>
store.ts           ← configureStore + epicMiddleware.run(rootEpic)
rootReducer.ts     ← combineReducers, exports RootState
rootEpic.ts        ← combineEpics
hooks.ts           ← useAppDispatch, useAppSelector, useAppStore
```

### Slice, actions, and epic stay together per feature

| File | What it contains |
|---|---|
| `usersSlice.ts` | State shape, reducers, slice actions |
| `usersActions.ts` | `createAction` epic triggers |
| `usersEpic.ts` | Epic(s) for this feature |

Keep them next to the components they serve. The epic imports from both the slice and the actions file.

### `rootEpic` is the aggregation point

Every time you add a feature with an epic, update `rootEpic.ts` to include it in `combineEpics`. This is the single place where you can see every epic wired into the store.

### Tests sit next to what they test

`UserList.test.tsx` next to `UserList.tsx`, `usersEpic.test.ts` next to `usersEpic.ts`. The shared `renderWithStore.tsx` goes in a common test-utils location.

### What to avoid

- **Epic and slice in the same file** — different responsibilities, different test strategies.
- **A top-level `epics/` folder separate from features** — forces jumping between directories to understand one feature.
- **Components importing `store.ts` directly** — use typed hooks from `hooks.ts`.
- **Circular imports between features** — if one feature's epic needs another feature's state, read it via `state$.value` rather than importing the other feature's internals.

---

## Quick Reference Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│  SLICE (usersSlice.ts)                                          │
│    createSlice → reducer + actions (fetchUsersStart, etc.)      │
│    Export: default reducer, named actions                       │
├─────────────────────────────────────────────────────────────────┤
│  ACTIONS (usersActions.ts)                                      │
│    createAction → epic triggers (fetchUsersRequested, etc.)     │
│    Not handled by any reducer directly                          │
├─────────────────────────────────────────────────────────────────┤
│  EPIC (usersEpic.ts)                                            │
│    action$.pipe(ofType(...), switchMap(...))                     │
│    Emits slice actions (start, success, failure)                │
├─────────────────────────────────────────────────────────────────┤
│  ROOT EPIC (rootEpic.ts)                                        │
│    combineEpics(fetchUsersEpic, authEpic, ...)                  │
├─────────────────────────────────────────────────────────────────┤
│  ROOT REDUCER (rootReducer.ts)                                  │
│    combineReducers({ users: usersReducer, auth: authReducer })  │
│    Export: RootState type                                       │
├─────────────────────────────────────────────────────────────────┤
│  STORE (store.ts)                                               │
│    configureStore + createEpicMiddleware                        │
│    epicMiddleware.run(rootEpic) — AFTER store creation          │
│    Export: AppDispatch type, store                              │
├─────────────────────────────────────────────────────────────────┤
│  HOOKS (hooks.ts)                                               │
│    useAppDispatch = useDispatch.withTypes<AppDispatch>()        │
│    useAppSelector = useSelector.withTypes<RootState>()          │
├─────────────────────────────────────────────────────────────────┤
│  COMPONENT                                                      │
│    dispatch(fetchUsersRequested()) → epic → reducer → re-render │
└─────────────────────────────────────────────────────────────────┘
```

### RxJS operator → saga equivalent

```
switchMap   ≈  takeLatest     (cancel previous)
mergeMap    ≈  takeEvery      (run all concurrently)
concatMap   ≈  (no direct)    (queue sequentially)
exhaustMap  ≈  takeLeading    (ignore while busy)
catchError  ≈  try/catch      (error handling)
startWith   ≈  put(action)    (emit before async work)
debounceTime ≈ delay + cancel (debounce triggers)
takeUntil   ≈  cancel(task)   (cancel on signal)
```

---

## Document Constraints (Preserve for Future Reference Docs)

This document was built under the following constraints. Apply them when creating similar advanced React reference guides:

1. **Language**: TypeScript — all function signatures, parameters, and return types must be explicitly typed.
2. **State management**: Redux Toolkit (RTK) slices + Redux-Observable with RxJS for side effects.
3. **Testing framework**: Node.js native test runner (`node:test`, stable in Node 20+).
4. **Testing style**: BDD (`describe`/`it` blocks) with `node:assert/strict`.
5. **Component testing (primary)**: React Testing Library (`@testing-library/react`) with `screen` queries and `userEvent` interactions. RTL tests are the focal point — they render components inside a real Redux store with epic middleware and assert on user-visible behaviour. All other test types (epic unit tests, reducer tests, action tests) are supplemental.
6. **Test store helper**: Required — provide a `renderWithStore` factory that creates a fresh `configureStore` + `epicMiddleware` per test, accepts `preloadedState` and `dependencies`, returns `store` and `actionLog`, and wraps the component in `<Provider>`.
7. **DOM environment**: Use `jsdom` via `global-jsdom/register` for Node.js-based component tests.
8. **Mocking**: Use `mock.fn`, `mock.method`, and `mock.restoreAll` from `node:test`. Prefer dependency injection via epic `dependencies` over module-level mocking where possible.
9. **Gotchas section**: Required — document TypeScript-specific pitfalls, common misconfigurations, and testing foot-guns.
10. **Architecture diagram**: Include a visual flow showing how components, store, middleware, epics, and reducers connect.
11. **Code examples**: Must be self-contained and runnable. Use realistic (but generic) domain models.
12. **Accessibility**: React component examples should include appropriate ARIA attributes.
13. **No external test runner dependencies**: Avoid Jest, Mocha, Vitest, or any third-party test runner. RTL and `userEvent` are allowed as they are DOM utilities, not test runners.

---

*Sources consulted: [Redux-Observable official docs](https://redux-observable.js.org), [RxJS marble testing guide](https://rxjs.dev/guide/testing/marble-testing), [Redux Toolkit TypeScript usage](https://redux-toolkit.js.org/usage/usage-with-typescript), [Node.js test runner docs](https://nodejs.org/docs/v20.18.3/api/test.html). Content was rephrased for compliance with licensing restrictions.*
