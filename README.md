# @taktikorg/hic-aliquid

[![npm](https://img.shields.io/npm/v/@taktikorg/hic-aliquid)](https://npm.im/@taktikorg/hic-aliquid)
[![build](https://github.com/taktikorg/hic-aliquid/workflows/build/badge.svg)](https://github.com/taktikorg/hic-aliquid/actions/workflows/build.yml)
[![publish](https://github.com/taktikorg/hic-aliquid/workflows/publish/badge.svg)](https://github.com/taktikorg/hic-aliquid/actions/workflows/publish.yml)
[![codecov](https://codecov.io/gh/iyegoroff/@taktikorg/hic-aliquid/branch/main/graph/badge.svg?token=YC314L3ZF7)](https://codecov.io/gh/iyegoroff/@taktikorg/hic-aliquid)
[![Type Coverage](https://img.shields.io/badge/dynamic/json.svg?label=type-coverage&prefix=%E2%89%A5&suffix=%&query=$.typeCoverage.atLeast&uri=https%3A%2F%2Fraw.githubusercontent.com%2Fiyegoroff%2F@taktikorg/hic-aliquid%2Fmain%2Fpackage.json)](https://github.com/plantain-00/type-coverage)
![Libraries.io dependency status for latest release](https://img.shields.io/librariesio/release/npm/@taktikorg/hic-aliquid/0.0.32)
[![bundlejs](https://deno.bundlejs.com/?q=@taktikorg/hic-aliquid@0.0.32,@taktikorg/hic-aliquid@0.0.32&treeshake=[*],[{+default+}]&badge=)](https://bundlejs.com/?q=@taktikorg/hic-aliquid)
[![npm](https://img.shields.io/npm/l/@taktikorg/hic-aliquid.svg?t=1495378566926)](https://www.npmjs.com/package/@taktikorg/hic-aliquid)

useReducer with effects

## Getting started

```
npm i @taktikorg/hic-aliquid
```

## Description

This hook is a basic approach to split view/logic/effects in React. It works in [StrictMode](https://reactjs.org/docs/strict-mode.html) and is easy to [test](test/index.spec.tsx#L5-L27). It is designed to be framework-agnostic and was tested with [react](/test/react.spec.tsx) and [preact](/test/preact.spec.tsx).

## Tutorial

This is going to be a Counter.

```ts
import React, { useRef, useState, useEffect, useLayoutEffect } from 'react'
import { UpdateMap, createBacklash } from '@taktikorg/hic-aliquid'

// A framework should provide react-like hooks
const useBacklash = createBacklash({ useRef, useState, useEffect, useLayoutEffect })

// State can be anything,
type State = number

// but an Action is always a record of tuples, where the key
// is the name of an action and value is a list of arguments.
type Action = {
  inc: []
  dec: []
}

// init is a pure (in react terms) function that has no arguments
// and just returns the initial state wrapped in array.
const init = () => [0] as const

// Unlike the standard useReducer, update/reducer is not a function with
// a switch statement inside, it is an object where each key is an action
// name and each value is a reducer that takes a state, rest action
// elements (if any) and returns next state wrapped in array. There is
// a helper UpdateMap type, that checks the shape of update object and
// makes writing types by hand optional.
const update: UpdateMap<State, Action> = {
  inc: (state) => [state + 1],

  dec: (state) => [state - 1]
}

export const Counter = () => {
  // In this example useBacklash hook takes init & update functions and
  // returns a tuple containing state & actions. Note that 'init' & 'update'
  // arguments of useBacklash is 'initial' and changing these things won't
  // affect the behavior of the hook. Also the actions object is guaranteed
  // to remain the same during rerenders just like useReducer's dispatch
  // function.
  const [state, actions] = useBacklash(init, update)

  return (
    <>
      <div>{state}</div>
      <button onClick={actions.inc}>inc</button>
      <button onClick={actions.dec}>dec</button>
    </>
  )
}
```

Passing arguments to init function.

```ts
// Let's change the init function to have a single parameter
const init = (count: number) => [count] as const

// ...

export const Counter = () => {
  // Inside useBacklash body init function is called only once,
  // so it is ok to inline it.
  const [state, actions] = useBacklash(() => init(5), update)

  // ...
}
```

For now `useBacklash` was used just as a fancy `useReducer` that returns an actions object instead of dispatch function. It doesn't make much sense to use it like this instead of `useReducer`. So let's make that counter persistent and see how `useBacklash` helps to handle side effects.

```ts
// We are going to use localStorage to store the state of the Counter.
// Since I/O is a side effect it can not be called directly from the init
// function. To model the situation 'state is not set yet' State type will
// be extended with 'loading' string literal.
type State = 'loading' | number

// Additional action 'loaded' will notify that Counter state is loaded.
type Action = {
  loaded: [count: number]
  inc: []
  dec: []
}

const key = 'counter_key'

// init and each update property functions return
// the value of Command type - [State, ...Effect[]]
export const init = (): Command<State, Action> => [
  'loading',
  // The next function is a side effect that will be called by useBacklash
  // internally. Here it has single parameter - the same actions object
  // that is returned from useBacklash call.
  ({ loaded }) => loaded(Number(localStorage.getItem(key)) || 0)
  // Additional can be added after the first one
  // and all of them will run in order.
]

export const update: UpdateMap<State, Action> = {
  // The second parameter is a value that was passed to the 'loaded' action
  // a few lines earlier.
  loaded: (_, count) => [count],

  inc: (state) => {
    // If someone manages to call 'inc' before the state is loaded,
    // just do nothing, that's the normal strategy for this example.
    if (state === 'loading') {
      return [state]
    }

    const next = state + 1

    // Like the init function an update returns a Command
    return [next, () => localStorage.setItem(key, `${next}`)]
  },

  dec: (state) => {
    if (state === 'loading') {
      return [state]
    }

    // This line is the only difference between 'inc' and 'dec'.
    // Probably I will refactor it someday...
    const next = state - 1

    return [next, () => localStorage.setItem(key, `${next}`)]
  }
}

export const Counter = () => {
  const [state, actions] = useBacklash(init, update)

  return state === 'loading' ? null : (
    <>
      <div>{state}</div>
      <button onClick={actions.inc}>inc</button>
      <button onClick={actions.dec}>dec</button>
    </>
  )
}
```

Sample test.

```ts
import { act, renderHook } from '@testing-library/react'
import { useBacklash } from '../src'
import { init, update } from '../src/Counter'

describe.only('Counter', () => {
  test('state should equal 1 after inc', () => {
    const { result } = renderHook(() => useBacklash(init, update))

    act(() => {
      result.current[1].inc()
    })

    expect(result.current[0]).toEqual(1)
  })
})
```

When running this test with `jest` in `jsdom` test environment everything works as expected. But let's imagine that we don't have access to `localStorage` in our test environment. In this case test will fail with error: `ReferenceError: localStorage is not defined`. To avoid these kind of errors, `useBacklash` has an optional third parameter - `injects`. This parameter's value will be passed as a second argument to every effect function.

```diff
  import React from 'react'
  import { Command, UpdateMap, useBacklash } from '../'

  type State = 'loading' | number

  type Action = {
    loaded: [count: number]
    inc: []
    dec: []
  }

+ type Injects = {
+   readonly getItem: Storage['getItem']
+   readonly setItem: Storage['setItem']
+ }

  const key = 'counter_key'

- export const init = (): Command<State, Action> => [
+ export const init = (): Command<State, Action, Injects> => [
    'loading',
-   ({ loaded }) => loaded(Number(localStorage.getItem(key)) || 0)
+   ({ loaded }, { getItem }) => loaded(Number(getItem(key)) || 0)
  ]

- export const update: UpdateMap<State, Action> = {
+ export const update: UpdateMap<State, Action, Injects> = {
    loaded: (_, count) => [count],

    inc: (state) => {
      if (state === 'loading') {
        return [state]
      }

      const next = state + 1

-     return [next, () => localStorage.setItem(key, `${next}`)]
+     return [next, (_, { setItem }) => setItem(key, `${next}`)]
    },

    dec: (state) => {
      if (state === 'loading') {
        return [state]
      }

      const next = state - 1

-     return [next, () => localStorage.setItem(key, `${next}`)]
+     return [next, (_, { setItem }) => setItem(key, `${next}`)]
    }
  }

  export const Counter = () => {
-   const [state, actions] = useBacklash(init, update)
+   // Updating 'injects' doesn't trigger rerenders, so it is safe to inline it.
+   const [state, actions] = useBacklash(init, update, {
+     getItem: ((...args) => localStorage.getItem(...args)) as Storage['getItem'],
+     setItem: ((...args) => localStorage.setItem(...args)) as Storage['setItem']
+   })

    return state === 'loading' ? null : (
      <>
        <div>{state}</div>
        <button onClick={actions.inc}>inc</button>
        <button onClick={actions.dec}>dec</button>
      </>
    )
}
```

Now the test can be rewritten with mocked `localStorage`:

```ts
test('state should equal 1 after inc', () => {
  let storage: string | null = null

  const { result } = renderHook(() =>
    useBacklash(init, update, {
      getItem: (_: string) => storage,
      setItem: (_: string, value: string) => {
        storage = `${value}`
      }
    })
  )

  act(() => {
    result.current[1].inc()
  })

  expect(result.current[0]).toEqual(1)
  expect(storage).toEqual('1')
})
```

# Trivia

It was developed as a boilerplate-free substitute of [ts-elmish](https://github.com/iyegoroff/ts-elmish) project. While it doesn't support effect composition or complex effect creators, it should be easier to grasp and have enough power to handle important parts of the UI-logic for any component.
