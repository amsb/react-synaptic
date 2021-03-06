# Evoke
Straightforward action-driven state management for straightforward apps. 

<div align="center">
  <br><br>
  <img src="https://raw.githubusercontent.com/amsb/react-evoke/master/logo.png" alt="Evoke" width="400">
  <br>
  <br><br>
</div>

Built on React [Suspense](https://www.youtube.com/watch?v=6g3g0Q_XVb4) and [Immer](https://github.com/mweststrate/immer), Evoke provides a simple framework for dispatching asynchronous state updating actions and accessing that state throughout the application. It is a lightweight library for shared application state management in the spirit of Command Query Responsibility Segregation (CQRS), flux, redux, etc.

## Overview

React Evoke enables components to read existing shared state synchronously (internally using React's built-in Context feature) or use React Suspense to suspend the render and initialize asynchronously if the state doesn't yet exist. Here's an example:

```javascript
function App() {
  return (
    <Store
      actions={{ loadQuote, nextQuote }}
      initializers={{ quotes: "loadQuote" }}
      initialState={{ quoteId: 1 }}
    >
      <ErrorBoundary fallback={ErrorMessage}>
        <Suspense fallback={<p>Loading...</p>}>
          <CurrentQuote />
        </Suspense>
      </ErrorBoundary>
    </Store>
  );
}

function CurrentQuote() {
  const [quoteId] = useStore("quoteId")
  const [quote, { nextQuote }] = useStore("quotes", quoteId)

  return (
    <div>
      <h4>{quote.title}</h4>
      <p>{quote.description}</p>
      <button onClick={() => nextQuote()}>Next Quote</button>
    </div>
  )
}

// show this when loadQuote throws an error
function ErrorMessage({ state, error, clearError }) {
  return (
    <>
      <h1 style={{ color: "red" }}>{error.message}</h1>
      <button onClick={() => clearError()}>Try Again</button>
      <pre>{JSON.stringify(state, null, 2)}</pre>
    </>
  )
}

// define an action to load quote data
async function loadQuote(store, quoteId) {
  const quote = await fetchQuote(quoteId)
  await store.update(state => {
    if (!state.quotes) {
      state.quotes = {}
    }
    state.quotes[quoteId] = quote
  })
  return { quoteId }
}

// define action the move to next quote
async function nextQuote(store) {
  await store.update(state => {
    state.quoteId = state.quoteId + 1
    if (state.quoteId >= MAX_QUOTE_ID) {
      state.quoteId = 1
    }
  })
}

```

On first render, the quote data for `quoteId` `1` isn't yet available, so the render is suspended while the `loadQuote(quoteId)` initializer action fetches it and updates the Store state. Once the asynchronous action resolves, React Suspense retries the render which succeeds because the quote data was loaded by the `loadQuote` action. The `ErrorBoundary` will catch errors like network errors thrown by `fetchQuote` and show the `ErrorMessage`.

You can [browse a simple yet complete example](https://github.com/amsb/react-evoke/blob/master/examples/nutshell/src/index.js), or walk through how to use Evoke block by block below.

## Components

### Store
Using Evoke involves three primary building blocks:

1. A **Store** for shared state
2. Asynchronous **actions** for updating shared state
3. Access shared state with **useStore** hook or **UseStore** component

The `Store` *component* holds shared application **state** and a registry of **actions** for modifying that state:

```javascript
import createStore from "react-evoke";
const { Store, UseStore } = createStore();

ReactDOM.render(
  <Store
    actions={{
      loadQuote,
      nextQuote
    }}
    initialState={{
      quoteId: 1
    }}
    initializers={{
      quotes: "loadQuote"
    }}
  >
    <App />
  </Store>,
  document.getElementById("root")
);
```

In basic usage, the `Store` component takes three props: `actions`, `initialState`, and `initializers`:

* **actions** *(required)* An object declaring asynchronous action handlers.
* **initializers** *(optional)* An object that maps top-level state name to an initialization action.
* **initialState** *(optional)* An object containing the initial state.


### Actions
The application defines **actions** for performing shared state updates. Actions are defined as functions with the signature `(store, ...payload) => Promise` (the payload can be zero, one or more arguments). You can use async/await or explicitly return a `Promise`. Here is an example action to perform an async fetch:

```javascript
async function loadQuote(store, quoteId) {
  const quote = await fetchQuote(quoteId);
  await store.update(state => {
    // update immer draft state with direct mutation
    state.quotes[quoteId] = quote;
  });
}
```

Action handlers update the state of the store with the `store.update` method. The update method takes a *function of the current state* as its argument. The **updater function must MUTATE the provided state object** ([examples](https://github.com/mweststrate/immer#example-patterns)) which is an Immer draft state that will produce an *immutable* update to the React component state managed by `Store`.

Actions can make asynchronous calls like fetching data from a remote server, update the store state as many times as desired, or dispatch other actions using the pattern `store.actions.someAction(payload)` (either `await`-ed or not as needed).

To keep your code organized, you might want to define your actions in a separate module that you import and feed to the `Store` component:

```javascript
import createStore from "react-evoke";
const { Store, UseStore } = createStore();
import * as actions from "./actions"

ReactDOM.render(
  <Store actions={actions}>
    <App />
  </Store>,
  document.getElementById("root")
);
```

### useStore
The shared state and the actions come together through the `useStore` hook (or the `UseStore` component). Evoke manages state updates using React Context so that components using state under `name` will update if that shared state changes.

```javascript
function CurrentQuote() {
  const [quoteId] = useStore("quoteId")
  const [quote, { nextQuote }] = useStore("quotes", quoteId)

  return (
    <div>
      <h4>{quote.title}</h4>
      <p>{quote.description}</p>
      <button onClick={() => nextQuote()}>Next Quote</button>
    </div>
  )
}
```

#### API
The `useStore` hook can be called in a number of different ways to access the data and actions you need:

* Use the entire object or collection identified by `name`:  
  `[object] = useStore(name)`

* Use the entire object or collection identified by `name` along with one or more `actions`:  
  `[object, { action1, action2 }] = useStore(name)`

* Use the value of `item` from collection `name`:  
  `[value] = useStore(name, item)`

* Use the value of `item` from collection `name` along with one or more `actions`:  
  `[value, { action1, action2 }] = useStore(name, item)`

* Use one or more actions without any data:  
  `{ action1, action2 } = useStore()`


#### UseStore Component
The `UseStore` component works just like the hook but uses the function-as-a-child render prop pattern: 

```javascript
function QuoteView({ quoteId }) {
  return (
    <UseStore name="quoteId">
      {quoteId => (
        <UseStore name="quotes" item={quoteId}>
          {(quote, { nextQuote }) => (
            <div>
              <h4>{quote.title}</h4>
              <p>{quote.description}</p>
              <button onClick={() => nextQuote()}>Next Quote</button>
            </div>
          )}
        </UseStore>
      )}
    </UseStore>
  );
}
```


### Initializers

What if `quoteId` hasn't been loaded into `quotes` yet? You can declare a `Store` `initializer` to tell Evoke to first execute the `loadQuote(quoteId)` action if that `item` isn't available:

```javascript
  <Store
    actions={{
      loadQuote,
      nextQuote
    }}
    initializers={{
      quotes: "loadQuote"
    }}
    initialState={{
      quoteId: 1
    }}
  >
    <App />
  </Store>,
```

To use this feature, you will also need to insert at least one [`Suspense`](https://reactjs.org/docs/code-splitting.html#suspense) component with a `fallback` somewhere in the component tree **below** the `Store` and **above** the first `UseStore` component or the first component using the `useStore` hook. The Suspense component will suspend rendering of its children while the item is being initialized.

```javascript
import createStore from "react-evoke";
const { Store, UseStore } = createStore();
import actions from "./actions"

ReactDOM.render(
  <Store actions={actions} initializers={...}>
    <Suspense fallback={<p>Loading...</p>}>
      <App />
    </Suspense>
  </Store>,
  document.getElementById("root")
);
```

#### API
The asynchronous action configured in `Store.initializers` will be dispatched when calls to `useState(name)` or `useState(name, item)` attempt to retrieve data that isn't available. The action will be dispatched with `(name, undefined)` or `(name, item)` depending on what usage triggered the initialization so the action handler will need to be able to initialize both the collection (when the second argument for the item is `undefined`) or a specific `item`. A specific `item` may be initialized before the collection, so the action handler needs to be attentive to creating an empty collection before setting the first item.

> NOTE: If the collection has been assigned a value other than `null` or `undefined` through the `initialState` property for the `Store` or otherwise, then Evoke will consider that collection to have data and therefore will not dispatch its initializer.

### ErrorBoundary

Initializers (and other code) can throw errors during render. The `createStore` function also returns an `ErrorBoundary` component to help in handling these errors. Here's an example of how it gets used:

```javascript
const { Store, ErrorBoundary, useStore } = createStore();

function ErrorMessage({ state, actions, error, clearError }) {
  return (
    <>
      <h1 style={{ color: "red" }}>{error.message}</h1>
      <button onClick={() => clearError()}>Try Again</button>
      <pre>{JSON.stringify(state, null, 2)}</pre>
    </>
  );
}

function App() {
  const [quoteId] = useStore("quoteId");
  return (
    <ErrorBoundary fallback={ErrorMessage}>
      <Suspense
        maxDuration={1000}
        fallback={
          <p style={{ color: "blue", fontWeight: "bold" }}>Loading...</p>
        }
      >
        <QuoteView quoteId={quoteId} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

The `ErrorBoundary` component must be a descendant of the `Store` component. The `fallback` prop provides a component which takes `{ state, actions, error, clearError }` as props, where:

* `state` is the current state of the Store which must be treated as **read-only**.
* `actions` are the callable async actions defined by the store.
* `error` is the error object that was thrown.
* `clearError` is an argument-less function that clears the error (and effectively "retries") *when the error was thrown by an initializer*.

## Advanced Usage

In addition to the basic `actions`, `initialState`, and `initializers`, the `Store` supports a few more advanced (but optional!) features that can be configured through its props.

### meta
The `Store` can optionally be configured with an additional `meta` prop. The `meta` prop can be set to an object containing data that is not part of the application's view state (as changing it will not trigger an update). This is a good place to stash API objects and the like for later use in executing actions. The `meta` object is passed to the actions through the `store` argument.

### derivedState
Sometimes it is helpful to structure your store data in a way that makes it easier to maintain but less convenient to work with in your application. Evoke makes it easy to have it both ways by defining `derivedState` for the `Store`. Once defined, you can use `derivedState` just like regular state in your `useStore` hooks. Neat!

As an example, consider the situation where you have a collection of people and their friends. You can easily maintain normalized data in the `Store` while conveniently accessing the list of friends for a particular person.

```javascript
<Store
  ...
  derivedState={{
    friendsList: (getState, personId) => getState("people", personId)["friendIds"].map(
        friendId => getState("people")[friendId]
      )
  }}
>
  ...
</Store>
```

But what if the "people" collection hasn't been initialized yet? No problem! If `getState` discovers that the requested data isn't initialized, it will kick-off the *Suspense* mediated initialization process and try again. **Initialization will occur sequentially in the order of evaluation**.

As with actions, you can keep your code organized by moving  state deriving functions to a module and importing it for use in your `Store`.

> NOTE: `derivedState` is synchronous! This is not the place to put asynchronous work like data fetching. that's what actions are for.

#### Memoization

But wait, there's more! What if your `derivedState` is expensive to compute? Memoization to the rescue! This API is necessarily more complex but typically won't be needed. Simple maps, reductions, etc. are fast enough and can be as fast or faster than memoization if that is the only computation being performed. Remember, memoization has additional overhead to check if the previous value can be used.

The essence of the memoized form is to separate the process into two parts by declaring the data selection independently from the computation.  You do this by declaring a two-element array for the `derivedState`: `[(getState, item) => result, result => derivedValue]`. Here's an example:

```javascript
<Store
  ...
  derivedState={{
    expensiveData: [
      (getState, dataId) => getState("someData", dataId),
      someData => expensiveCalculation(someData)
    ]
  }}
>
  ...
</Store>
```

> NOTE: The memoization maintains a cache size of 1 with the primary intent of eliminating re-calculation when re-rendering the same or similar state.

### middleware
The Store can be enhanced through middleware. Right from the get-go, you'll probably want to
use the built-in logging middleware:

```javascript
import { consoleLogging } from "react-evoke"

<Store
  ...
  middleware={[consoleLogging]}
>
  ...
</Store>
```

## Caveats

* This library uses React Context's `unstable_observedBits` internally to limit consumer updates to only those "subscribing" to the modified substate. This unstable/experimental feature of React may be removed/replaced in future version of React. Hopefully it will be replaced by something even better.
* This library makes use of [`Suspense`](https://reactjs.org/docs/code-splitting.html#suspense) in a way that isn't (yet) officially sanctioned.

## Rationale

Why create yet another React state management library? React provides an unopinionated open architecture ecosystem that makes it a flexible application development foundation. While there is the tendency to want the "one true solution", the reality is that different applications and teams have different needs and preferences. This, unfortunately, creates a bit of barrier to newcomers, but it creates a lot of adaptability for application development. React provides an effective local state management solution, but until recently didn't much help for shared state management when your needs extend beyond lifting state up and passing down props. There is a spectrum of community frameworks available spanning the gamut from reactive (mobx) to reductive (redux). The two frameworks at the extremes are both amazing in their own right, but there remain many opportunities for different trade-offs in the middle to suit different use cases. React Evoke is one such middle dweller. With the new class Context API and recently released Suspense features, React now provides a firm foundation to create lightweight solutions to accommodate different needs and preferences.  If you like the trade-offs it makes and the style of its API, feel free to use it in your own projects!

In particular, React Evoke was designed for applications that want to use the CQRS/redux style primarily for relatively coarse-grained interactions with remote services while embracing React's primitives. As a result, it is easy to define async actions that get/post from/to remote services, but the framework is not well suited for making every key-stroke or local interaction an "action" since every action creates and resolves a promise. In practice, this means using local state to control form and other micro-interactions while using Evoke to process the end result. Furthermore, React Evoke forgoes the (sometimes very helpful) formalism of separating remote query/mutation from local query/mutation by embedding the local mutation operation inside the action instead of separating the two as would be done with redux. If these trade-offs make sense for your app, so might React Evoke!


## Legal

Released under MIT license.

Copyright &copy; 2019 Alexander M. Sauer-Budge
