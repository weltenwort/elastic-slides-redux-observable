class: center, middle, inverse

# Epics in Redux

redux-saga and redux-observable

---

# Redux Data Flow Recap: idealized

* unidirectional

* events originate from the user

```no-highlight
                                                     +------+
                                                     | User |
                                                     +--+---+
                                                        |
                    +-------+                   +-------v--------+
                    | State +-------------------> User Interface |
                    +---^---+                   +-------+--------+
                        |                               |
                        |                               |
                        |                               |
                   +----+----+                      +---v----+
                   | Reducer <----------------------+ Action |
                   +---------+                      +--------+
```

---

# Redux Data Flow Recap: (more) realistic

* unidirectional

* events originate from the user, timers, http requests, etc...

```no-highlight
                                                     +------+
                                                     | User |
                                                     +--+---+
                                                        |
                    +-------+                   +-------v--------+
                    | State +-------------------> User Interface |
                    +---^---+                   +-------+--------+
                        |                               |
                        |               +-------------+ |
                        |               |             | |
                   +----+----+    +-----+------+    +-v-v----+
                   | Reducer <----+ Middleware <----+ Action |
                   +---------+    +----+-+-----+    +-^-^----+
                                       | |            | |
                                       | | +-------+  | |
                                       | +-> Timer +--+ |
                                       |   +-------+    |
                                       |                |
                                       |  +---------+   |
                                       +--> Network +---+
                                          +---------+
```

---
class: center, middle

# Tying together async side effects

---
class: center, middle

# Thunks & Promises

middleware to chain async actions

synchronization? cancellation? testing?

---
class: center, middle

# Epics and Sagas

middleware to orchestrate async event flows and side-effects

`redux-saga` and `redux-observable`

---

# redux-saga

* middleware composed of generators of type `(): Iterable<Effect>`

* sagas yield "effects" like `take`, `put`, `select`, `race`, `throttle`, etc., which are executed by the redux-saga runtime

```typescript
function* watchFirstThreeTodosCreation() {
  for (let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED');
  }
  yield put({ type: 'SHOW_CONGRATULATION' });
}
```

```typescript
function* fetchPostsWithTimeout() {
  const { posts, timeout } = yield race({
    posts: call(fetchApi, '/posts'),
    timeout: call(delay, 1000)
  });
  if (posts) {
    yield put({ type: 'POSTS_RECEIVED', posts });
  } else {
    yield put({ type: 'TIMEOUT_ERROR' });
  }
}
```

---

# redux-observable

* middleware composed of functions of type

```typescript
(action$: Observable<Action>, store: Store, dependencies: any): Observable<Action>
```

* all actions in the returned observable are in turn dispatched to the store

```typescript
const watchFirstThreeTodosCreation = action$ =>
  action$
    .ofType$('TODO_CREATED')
    .take(3)
    .last()
    .mapTo({ type: 'SHOW_CONGRATULATION' });
```

```typescript
const fetchPostsWithTimeout = () =>
  switch(
    fetchApi('/posts')
      .map(posts => ({ type: 'POSTS_RECEIVED', posts })),
    of({ type: 'TIMEOUT_ERROR' })
      .delay(1000)
  );
```

---

# Both can be composed

* `redux-saga`:

```typescript
function* rootSaga() {
  yield spawn(saga1);
  yield spawn(saga2);
}
```

* `redux-observable`:

```typescript
import { combineEpics } from 'redux-observable';

const rootEpic = combineEpics(
  epic1,
  epic2
);
```

---

# Both are inserted as middleware

* `redux-saga`:

```typescript
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

import {saga, reducer} from './someModule'

const sagaMiddleware = createSagaMiddleware()
const store = createStore(reducer, applyMiddleware(sagaMiddleware))

sagaMiddleware.run(saga)
```

* `redux-observable`:

```typescript
import { createStore, applyMiddleware } from 'redux';
import { createEpicMiddleware } from 'redux-observable';

import { epic, reducer } from './someModule';

const epicMiddleware = createEpicMiddleware(epic);
const store = createStore(reducer, applyMiddleware(epicMiddleware));
```

---

# Testing sagas

* generators are externally controlled, i.e. stepped through via `iter.next()`

* effects results can be injected as the `next()` argument without executing it

```typescript
test('congratulates after the first three todos', () => {
  const gen = watchFirstThreeTodosCreation();

  for (let i = 0; i < 2; i++) {
    expect(gen.next({ type: 'TODO_CREATED' }).value)
      .toEqual(take('TODO_CREATED'));
  }

  expect(gen.next({ type: 'TODO_CREATED' }).value)
    .toEqual(put({ type: 'SHOW_CONGRATULATION' }));

  expect(gen.next().done).toBeTruthy();
});
```

---

# Testing epics

* mocks can be injected as dependencies

* time can be controlled via a virtual `TestScheduler`

* sequences can be asserted using marble diagrams (e.g. with `rxjs-marbles`)

```typescript
test('congratulates after the first three todos', marbles((marble) => {
  const values = {
    a: { type: 'TODO_CREATED' },
    b: { type: 'SHOW_CONGRATULATION' }
  };

  const input$ =     marble.hot('-a-a-a-a-a', values);
  const expected$ = marble.cold('-----b-|', values);
  const output$ = watchFirstThreeTodosCreation(input$);

  marble.expect(output$).toBeObservable(expected$);
}
```

---

# Choosing redux-observable

* easier to ensure that no actions are skipped

* more built-in time synchronization operators

* built-in http request cancellation

* observables already used in Kibana

---

# Example from the logging app

## Cancel older requests when jumping

```typescript
action$
  .filter(jumpToTarget.match)
* .switchMap(({ payload: target }) => {
    return fetchEntries(/* ... */);
  })
```

---

# Examples from the logging app

## Queue, filter and cancel reactions

```typescript
action$
  .filter(reportVisibleTextInterval.match)
  .filter(() => {
    const state = store.getState();
    return (
      !selectIsStreamEntryLoading(state, 'start')
      && !selectIsStreamEntryExhausted(state, 'start')
      && !selectHasStreamEntryLoadingFailed(state, 'start')
    );
  })
* .exhaustMap(({ payload }) => {
    return fetchEntries(/* ... */)
      .let(
        handleExtendStreamInterval({
          count: chunk_size,
          endpoint: 'start',
          target
        })
      )
*     .takeUntil(action$.filter(initializeStreamInterval.started.match));
  })
```

---

# Examples from the logging app

## Perform garbage collection when idle

```typescript
action$
  .filter(reportVisibleTextInterval.match)
* .debounceTime(VISIBLE_INTERVAL_SETTLE_TIMEOUT)
  .filter(() => {
    const state = store.getState();
    return (
      !selectIsStreamEntryLoading(state, 'start')
      && !selectIsStreamEntryLoading(state, 'end')
    );
  })
  .switchMap(({ payload: }) => {
    return [
      consolidateStreamInterval(/* ... */)
    ];
  })
```
