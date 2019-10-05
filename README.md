# react-stored
The power and simplicity of `useState`, but with persistence across browser sessions, global sync'd access without context, speed and memory optimisation, anti-ill form checks, and other cool stuff.

Ever dreamed of such of feature but couldn't come up with a 100% clean and satisfying solution ? Well, this package is for you.

#### It is designed to :

1. provide you with a **reliable key-based** `useState`-like feature called `useStore`,
2. with tree-wide **auto-sync** with similar `useStore` calls,
3. **without** context,
4. with **persistence** accross page reloads, tabs, sessions,
5. with **configurable safety asserts** on deserialization and **default fallbacks**,
6. with **no unnecessary** rerender, ever,
7. with **simplicity** and **cool configuration** options.

## Quick demo

Say you have the two following components far away from one another in the tree :
```javascript
import React from 'react'
import { useStore } from 'react-stored'

function FirstComponent() {
  const [counter, setCounter] = useStore('counter', 0)
  return <>
    <h1>Counter : {counter}</h1>
    <button onClick={() => setCounter(counter + 1)}>
      Increment
    </button>
  </>
}
```

```javascript
import React from 'react'
import { useStore } from 'react-stored'

function SecondComponent() {
  const [counter, setCounter] = useStore('counter', 0)
  return <>
    <h1>Counter : {counter}</h1>
    <button onClick={() => setCounter(0)}>
      Reset
    </button>
  </>
}
```

Since they share the same key (`'counter'`), they actually seemlessly share the _same value_ and keep one another in sync _no matter_ how hard you try to unsync them. **Even better** : if you refresh the page, nothing changes. The values are persistent. You better like them, because they ain't going anywhere unless you change the key or... the key _prefix_ (see config section below).

The second argument to `useStore`, the number `0` in this case, represents the default counter value, as no persistent save cound be found the first time around.

## Reference

### 1- The `useStore` hook

This is the cornerstone of this package. It 'connects' you to a specific store slot, identified by key, and returns the value at that location as well as an update function. It's overall feel mimics `useState`. It also listens to any outside change, and rerenders accordingly to keep all the parts of your UI in sync.

It can take up to 3 arguments (only the key is required) :

```javascript
const [value, setValue] = useStore(key, defaultValue, assertFunction)
```
- `key` : Any string unambiguously identifying a unique store slot.
- `defaultValue` : The value affected by default to the store slot and returned by `useStore` when no previous save was found. This could be any JSON value.
- `assertFunction` : Any deserialized JSON save passes through this function and has to return `true`. Otherwise, `defaultValue` will be used and overwrite the save. This can be very handy, for example to prevent hydration of ill-formed JSON. I would usually use [ajv](https://www.npmjs.com/package/ajv) in places like these.

#### Identity and hook optimization

Just like most hooks, `useStore` relies on **object identity** to optimize internal recomputations. If your `defaultValue` is **an object or an array**, please use [`useRef`](https://reactjs.org/docs/hooks-reference.html#useref) or [`useMemo`](https://reactjs.org/docs/hooks-reference.html#usememo) to keep the same reference as long as possible :

```javascript
const [value, setValue] = useStore('key', useRef({ x: 1, y: 2 }).current)
```

Similarly, use [`useCallback`](https://reactjs.org/docs/hooks-reference.html#usecallback) for the assert function :

```javascript
const assert = useCallback(state => ajv.validate(mySchema, state), [])
const [state, setState] = useStore('key', 42, assert)
```

#### The update function

Like `useState`, the update function can take a value, or a _function_ taking the old value as an argument and returning the new one.

```javascript
const [counter, setCounter] = useStore('counter', 0)
<button onClick={() => setCounter(counter + 1)}>
  Increment
</button>

// Equivalent to :

const setCounter = useStore('counter', 0)[1]
<button onClick={() => setCounter(counter => counter + 1)}>
  Increment
</button>
```

The **identity** of this update function is preserved as long as the identity of `useStore`'s three parameters are preserved.

### 2- The `addSchema` function

This configuration function allows you to set _default_- default values and _default_ assert functions to certain keys or key patterns outside of your React tree, typically in `index.js` before your `ReactDOM.render`. If you don't rely on props to set default values and assert functions, you shouldn't set them at component-level and `addSchema` should be **your primary way of configuring store slots**.

It takes the same arguments as `useStore` except the key can be a regexp. If it is, then all keys matching the regexp will use the given configuration.

```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import { addSchema } from 'react-stored'
import App from './App'

addSchema('counter', 0, counter => counter < 100)
// Any invocation of 'counter' will now use 0 as its default value, and ensure
// that any retrieved save is smaller than 100. If not, 0 will be used instead.

addSchema(/app-v\d+/, { currentUser: null })
// Any invocation of 'app-v1', 'app-v43', 'app-v9987', etc. will use the given
// object as its default value.

ReactDOM.render(<App/>, document.getElementById('root'))
```

### 3- The `config` function

With this function, you can configure your stores globally. It has to be called of the React structure, before any `useStore` call, so usually somewhere in your `index.js` before your `ReactDOM.render`.

```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import { config } from 'react-stored'
import App from './App'

// Below are the DEFAULT settings, it is pointless to set them explicitly to these values :

config({
  // IMPORTANT : A seemless prefix to ALL your keys, this has to be specific to your app :
  keyPrefix: '',
  
  // The persistent storage to be used (could be replaced with sessionStorage) :
  storage: window.localStorage,
  
  // A function that should transform JSON into a string :
  serialize: JSON.stringify,
  
  // A function that should transform a string into JSON :
  deserialize: JSON.parse
})

ReactDOM.render(<App/>, document.getElementById('root'))
```

#### `keyPrefix` is important

Unless you have some very specific use cases, the `keyPrefix` is really the only important part to configure. You set it _once_, and everything stored or retrieved from the storage will use that prefix in addition to the keys used in your components. All this happens of course _seemlessly_, you don't have to think about it.

Yes, `localStorage` is compartmentalized by domain but you could have several apps by domain. It's just a good habit to set a `keyPrefix` that is specific to your app.

Also, imagine several customers already tested your app and have _their local copy_ of the store. Now say you wanna change the JSON schema because of some new requirements. You could just set `keyPrefix` to the **current version of the app**, thus preventing any hydration of outdated JSON saves.

Here is typically what my `index.js` looks like :

```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import { config } from 'react-stored'
import App from './App'

config({
  keyPrefix: 'my-app-v2.4.1-',
})

ReactDOM.render(<App/>, document.getElementById('root'))
```

#### Schema definition with `config`

You can also replace all your [`addSchema`](#2--the-addschema-function) calls with a single array using the `config` function :

```javascript
config({
  keyPrefix: 'my-app',
  schemas: [
    {
      key: 'counter',
      init: 0,
      assert: counter => counter < 100
    },
    {
      key: /grid-v\d+/,
      init: { x: 1, y: 0 }
    }
  ]
})
```

### 3- The `readStore` function

This gives you the possibility to passively read the content of your store outside of any component. This was useful for me when I needed to pass a stored token to some server requests using [Apollo](https://www.apollographql.com/docs/react/) links. It takes only one argument, the key, and returns the corresponding JSON.
