# No wasted rendering

In this article, we shall learn how a naive thought of optimizing the
`render`-ing of a ReactJS application led to a deep rabbit hole.

## Contents

- [How does `render` works in ReactJS](#how-does-render-works-in-reactjs)
- [State management in React](#state-management-in-react)
- ["Mocking" `Redux` using React Hooks](#mocking-redux-using-react-hooks)

---

## How does `render` works in ReactJS

Actually, my question was more of _"How does React know when to update a
component?"_.

### So how does React know when to update a component?

React utilizes the concept of a
_[Virtual DOM (VDOM)](https://reactjs.org/docs/faq-internals.html)_ and
_[Reconciliation](https://reactjs.org/docs/reconciliation.html)_ to enable its
declarative API.

> #### Aside: what is declarative?
>
> **Declarative**: "I would like a cup of tea, with ice, please"
>
> **Imperative**: "First, boil water; then, put some tea leaves in a bag; then,
> put that bag in a cup. Pour the hot water into the cup, and wait for 3 minutes.
> Next, prepare a cup full of ice; then, pour the tea into that cup. Done."
>
> #### What does this mean in React?
>
> You tell React what state you want the UI to be in, and it makes sure the DOM
> matches that state. This abstracts out the attribute manipulation, event
> handling, and manual DOM updating that you would otherwise have to use to build
> your app.

### And how does _Reconciliation_ works?

TLDR; React will perform a **referential** check on each component using the Virtual
DOM and decide if it needs to update the "real" DOM.

For the longer (and more official) version, please refer to this
[link](https://reactjs.org/docs/reconciliation.html).

> #### Aside: what is referential check?
>
> In Javascript, a referential check is done using `===`.
>
> ```javascript
> 1 === 1             // true
> "hello" === "hello" // true
> true === true       // true
>
> [] === []           // false
> {} === {}           // false
>
> const obj = {};     // assigned a memory address
> const obj2 = obj;   // created another pointer to the same address
> obj === obj2        // true, as both point to the same address
> ```
>
> For primitives, as long as the values are the same, the check will return
> true. For arrays and objects, return true if memory address is the same.
>
> #### And, what does that mean in React?
>
> When performing reconciliation, React does not have to perform deep comparison
> of a component's state and props to determine if there are changes. The
> assumptions made are:
>
> 1. Two elements of different types will produce different trees.
> 2. The developer can hint at which child elements may be stable across
>    different renders with a `key` prop.

### Now, how does `render()` come into play?

Each time there is a change to the state or props of a component, the `render()`
function is called on that component to perform reconciliation.

Each time `render()` is called on a component, `render()` is also called on its
children components. This happens recursively until there are no more components
to `render`.

If a component has a deep tree structure, `render`-ing it _may_ be expensive.

#### But sometimes, `render` is skipped!

Based on the concept of purity in functional programming paradigms, a React
component is considered pure if _it renders the same output given the same state
and props_.

For class components, we have
[`PureComponents`](https://reactjs.org/docs/react-api.html#reactpurecomponent).
For functional components, we have
[`React.memo`](https://reactjs.org/docs/react-api.html#reactmemo).

Both methods have differences in how they did it, but the idea is to avoid
calling `render()` if a shallow[\*]() comparison of the component's state and props
showed no changes

###### \* through referential checks done of each key in the state and props

### Summing it all up

TLDR; `render()` is called on a component each time the state or props of that
component changed, or when `render()` is called on the parent of that component;
Pure components can be used to avoid costly `render`-ing.

---

## State management in React

In ReactJS, a React component translate raw data into rich HTML for use in the
UI. These data are represented by two things - `state` and `props`.

The main difference between `props` and `state` is that `props` are passed in,
and `state` is maintained by the component itself.

### When do we use `state`?

- when the data should be created, and maintained by the component
- preferably as little as possible (because they add complexity)

### And when do we use `props`?

- when the data is considered read-only in that component i.e. the data is not
  meant to be changed by the receiving component

### Various ways of state management in React

Another way to think about it is "how is data used throughout an application?"

For example,

- how is the data created
- who use the data (or which part of the data)
- how does one update the data (or part of)
- how to pass data around

In short, there are three main cases of "passing" data in a React application:

1. From Parent to Child
2. From Child to Parent
3. Between Siblings (or cousins, or distant relatives)

### Passing data from parent to child

This one is the easiest. It is usually[\*]() done through `props` passing.

```js
const fruits = [ { name: 'apple' }, { name: 'banana' } ];

const Parent = () => (
  <div>
    fruits.map(fruit => <Child name={fruit.name}>);
  </div>
);

const Child = ({ name }) => (
  <div key={`child-${name}`}>{name}</div>
);
```

###### \* there are other methods to do so

### Passing data from child to parent

This is trickier, but usually[\*]() done through callback functions.

```js
const initialCounters = { apple: 42, banana: 42 };

const Parent = () => {
  const [counters, setCounters] = useState(initialCounters);
  const updateCount = (name, newCount) => {
    setCounters({ ...counters, name: newCount });
  };
  return (
    <div>
      counters.map(({(name, count)}) => (
        <Child name={name} count={count} updateCount={updateCount} />
      );
    </div>
  );
};

const Child = ({ name, count, updateCount }) => (
  <div key={`child-${name}`}>
    <span>{`${name}: ${count}`}</span>
    <button onClick={updateCount(name, count + 1)}>+1</button>
    <button onClick={updateCount(name, count - 1)}>-1</button>
  </div>;
);
```

###### \* there are other methods to do so

### Passing data between siblings (or cousins, or distant relatives)

Sometimes, data needs to be shared amongst siblings and can be updated by either
parent, or the children themselves.

There are several ways, and the two most common ways are:

1. Using a callback function in a **common** "ancestor"
2. Using React Context API

#### Using a callback function in a common parent

When these siblings (or cousins) share a common parent further up the ancestry
lineage, one common strategy is to have the common parent pass down a callback
function.

The common parent will maintain the `state`, and the children (or
grand-children) receive part of this `state` through `props`.

These children (or grand-children) then uses the callback function to update the
`state`, so that all dependents of that part of the `state` can update.

```js
const initialCategories = {
  a: [ { apple: 42 }, { avocado: 10 } ],
  b: [ { banana: 42 } ]
};

const GrandParent = () => {
  const [categories, setCategories] = useState(initialCategories);
  const updateCount = (category, name, newCount) => {
    const { category: fruits, ...others } = categories;
    const allExceptTheOneToUpdate = fruits.filter(fruit => fruit.name !== name);
    setCategories({
      ...categories
      category: [...allExpectTheOneToUpdate, { name: newCount }]
    });
  };
  return (
    <div>
      Object.keys(categories).map(category) => {
        const { category: fruits } = categories;
        return (
          <Parent
            category={category}
            fruits={fruits}
            updateCount={updateCount}
          />
        );
      });
    </div>
  );
};

const Parent = ({category, fruits}) => {
  const updateCountForThisCategory = (name, newCount) => {
    updateCount(category, name, newCount);
  }
  return (
    <div key={`parent-${nameStartingWith}`}>
      fruits.map(({ name, count }) => (
        <Child
          name={name}
          count={count}
          updateCount={updateCountForThisCategory}
        />
      );
    </div>
  )
}

const Child = ({ name, count, updateCount }) => (
  <div key={`child-${name}`}>
    <span>{`${name}: ${count}`}</span>
    <button onClick={updateCount(name, count + 1)}>+1</button>
    <button onClick={updateCount(name, count - 1)}>-1</button>
  </div>;
);
```

#### Using React Context API

Another way we can pass data between siblings (or cousins, or distant relatives)
is through the [React Context](https://reactjs.org/docs/context.html).

Context is designed to share data that can be considered "global" for a
**tree** of React components.

It can be used to prevent props-drilling.

An alternative to Context is to use
[Component Composition](https://reactjs.org/docs/composition-vs-inheritance.html)

#### Example of using Context

To create a context:

```js
// src/MyContext.js

const MyContext = React.createContext();

export default MyContext;
```

Creating a context provider:

```js
// src/ContextProvider.js

import MyContext from 'MyContext';

const ContextProvider = ({ children }) => (
  <MyContext.Provider value={
    {
      color: 'blue'
      callback: () => {}
    }
  }>
    {children}
  </MyContext.Provider>
);
```

Creating a context consumer using React hooks:

```js
// src/ContextConsumer.js

import MyContext from "MyContext";

const ContextConsumer = () => {
  const { color, callback } = useContext(MyContext);

  const handleClick = event => {
    callback();
  };

  return <button onClick={handleClick}>{color}</button>;
};
```

Creating a context consumer using React class components:

```js
import React from "react";
import MyContext from "MyContext";

class ContextConsumer extends React.Component {
  static contextType = MyContext;
  constructor(props) {
    super(props);
    const value = this.context;
    // --- or ---
    const { color, callback } = this.context;
  }
}

// Another way to write it is:
class ContextConsumer extends React.Component {
  /* ... */
}
ContextConsumer.contextType = MyContext;
```

---

## "Mocking" `Redux` using React Hooks

`Redux` is a powerful library built to help deal with state management.
Essentially, it is much like a global data store that helps maintain the state
of the application. Different components can subscribe to different parts of
the store to receive updates on changes to various parts of the state.

However, sometimes using `Redux` feels like using a sledgehammer to crack a nut.

### How React Hooks came to the rescue

In React Hooks, there are two hooks that we can leverage to create a "global"
data store (much like how `Redux` does it).

These two are `useReducer` and `useContext`.

> #### Aside: how to `useReducer`
>
> React calls it an alternative to `useState`. Preferable to `useState` when:
>
> - there are complex state logic that involves multiple sub-values
> - the next state depends on the previous one
>
> Optimizes performance for components that trigger deep updates by passing
> `dispatch` down instead of callbacks[\*]().
>
> - Can avoid passing callbacks through props-drilling
> - `dispatch` context never changes, so components that reads it don't need to
>   rerender
>
> ###### \* according to this [article](https://reactjs.org/docs/hooks-faq.html#how-to-avoid-passing-callbacks-down)
>
> ---
>
> #### React Hooks: `useReducer` - how to use it
>
> The `useReducer` API:
>
> ```js
> const [state, dispatch] = useReducer(
>   reducerFunction,
>   initialState,
>   lazyInitializationFunction
> );
> ```
>
> Example of a reducer function:
>
> ```js
> // src/countReducer.js
>
> const INCREMENT_COUNT = "INCREMENT_COUNT";
> export const actionTypes = {
>   INCREMENT_COUNT
> };
>
> export const reducer = (state, action) => {
>   switch (action.type) {
>     case "INCREMENT_COUNT":
>       return { ...state, count: state.count + 1 };
>     default:
>       return state;
>   }
> };
> ```
>
> Using `useReducer`:
>
> ```js
> // src/Component.js
>
> import { actionTypes, reducer } from "countReducer";
>
> const initialState = { count: 0 };
>
> const Component = () => {
>   const [state, dispatch] = useReducer(reducer, initialState);
>
>   const handleIncrement = () => {
>     dispatch({ type: actionTypes.INCREMENT_COUNT });
>   };
>   return (
>     <div>
>       <span>Count: {state.count}</span>
>       <button onClick={handleIncrement}>+</button>
>     </div>
>   );
> };
> ```

### Creating "`Redux`" using React Hooks

_Serves 1 React Component_

- State to be stored "globally"
- 1 Reducer function
- 1 Context Provider
- 1 Pure Component
- 1 Contextual Component
- 1 connectToContext utility function
- 1 selector custom hook

**Steps**:

1. Create a state that we want to store "globally".
1. Create a reducer function to maintain the state changes.
1. Use `useReducer` to generate the `[state, dispatch]` array.
1. Store the `state` and the `dispatch` function in the context.
1. Connect the component to the context.
1. Serve up the contextual component while hot.

#### 1. Create a state that we want to store "globally"

First, we need a state that we want to store "globally"[\*](). In this recipe, we
shall use the following list that contains ingredients that we have in stock:

```js
const initialInventory = [
  {
    identifier: "white-flour",
    amountInGrams: 500
  },
  {
    identifier: "baking-powder",
    amountInGrams: 2000
  }
];
```

###### \* Only global to a tree of React components

#### 2. Create a reducer function to maintain the state changes

Next, we need a reducer function to help us maintian the state changes.

```js
const inventoryReducer = (inventory, action) => {
  switch (action.type) {
    case "RESTOCK":
      const everythingElse = inventory.filter(
        ingredient => ingredient.identifier !== action.identifier
      );
      const ingredientToRestock = inventory.find(
        ingredient => ingredient.identifier === action.identifier
      );
      return [
        ...everythingElse,
        {
          identifier: ingredientToRestock,
          amountInGrams:
            ingredientToRestock.amountInGrams + action.amountToRestockInGrams
        }
      ];
    case "USE":
    // ...
    default:
      return inventory;
  }
};
```

#### 3. Use `useReducer` to generate the `[state, dispatch]` array

Now, we can use the `useReducer` hook to generate the `dispatch` function.

```js
const [inventory, dispatch] = useReducer(inventoryReducer, initialInventory);
```

#### 4. Store the state and the dispatch function in the context

We can then "hook" up the reducer to the context.

```js
<InventoryContext.Provider value={[inventory, dispatch]} >
```

And putting it all together to form a Higher-Order-Component (HOC):

```js
const InventoryProvider = ({ reducer, initialState, children }) => (
  <InventoryContext.Provider value={useReducer(reducer, initialState)}>
    {children}
  </InventoryContext.Provider>
);
```

You might think that we can start using the context in a consumer, like this:

```js
const Ingredient = ({ identifier }) => {
  const [inventory, dispatch] = useContext(MyContext);

  const thisIngredient = inventory.find(
    ingredient => ingredient.identifier === identifier
  );

  const handleRestock = () => {
    dispatch({
      type: "RESTOCK",
      amountToRestockInGrams: 100
    });
  };

  const handleUse = () => {
    /* ... */
  };

  const { name, amountInGrams } = thisIngredient;

  return (
    <div key={identifier}>
      <span>
        {name}: {amountInGrams} g
      </span>
      <button onClick={handleRestock}>+ 100 g</button>
      <button onClick={handleUse}>- 100 g</button>
    </div>
  );
  //...
};
```

This is not entirely wrong, but not entirely `render`-cycle-saving.

This is because each time `inventory` changes in the context, the `Ingredient`
component will have to re-`render`. Now, `inventory` is part of this component's
`state`. Remember that each time there is a change in the `state` or `props`,
`render` will be called.

#### 5. Connect the component to the context

To do it the `render`-saving way, we will need to break the component into two
parts - the contextual part, and the pure part.

How our pure component will look like:

```js
import React from 'react';

const PureIngredient = React.memo({ identifier, name, amountInGrams, dispatch }) => {
  return /* same thing here */;
};
```

Note the magic words `React.memo`.

Before we can create our contextual component, we will need two more things:

1. A utility function to connect the component to the context, and
2. A custom hook to perform "sieving" of `props` from the context

The utility function that we will create is as such:

```js
const connectToContext = (WrappedComponent, selectFrom) => {
  return props => {
    const selectedProps = selectFrom(props);
    return <WrappedComponent {...selectedProps} {...props} />;
  };
};
```

The `connectToContext` function is actually a Higher-Order-Component (HOC).
It basically just uses a custom hook to sieve out `props` that will be passed
into the component to be wrapped.

The custom hook to "sieve" `props` is as such:

```js
const useSelectFromInventoryContext = ({ identifier }) => {
  const [inventory, dispatch] = useInventoryContext();
  const thisIngredient = inventory.find(
    ingredient => ingredient.identifier === identifier
  );
  const amountInGrams = thisIngredient ? thisIngredient.amountInGrams : 0;
  return {
    identifier: identifier,
    amountInGrams: amountInGrams,
    dispatch: dispatch
  };
};
```

Following specifications from the docs, we will need to prepend all custom hooks
with `use`.

And now, our contextual component will look like:

```js
const ContextualIngredient = connectToContext(
  PureIngredients,
  useSelectFromInventoryContext
);
```

#### 6. Serve up the contextual component while hot.

Now, we are ready to serve up the contextual component!

```js
const InventoryWithFetching = () => {
  const [recipes, setRecipes] = setState([]);
  const [inventory, setInventory] = setState([]);

  /* some state initialization happens here */

  return (
    <InventoryProvider initialState={inventory} reducer={inventoryReducer}>
      recipes.map(recipe => {
        const { identifier, name, ingredients } = recipe;
        return (
          <Recipe
            identifier={identifier}
            name={name}
            ingredients={ingredients}
          />
        );
      });
    </InventoryProvider>
  );
};

const Recipe = ({ identifier, name, ingredients }) => {
  return (
    <div key={identifier}>
      <span>{name}</span>
      ingredients.map(ingredient => (
        <ContextualIngredient identifier={ingredient.identifier} />
      ));
    </>
  );
};
```

---

## References

1. React Virtual DOM - https://reactjs.org/docs/faq-internals.html
1. React Reconciliation - https://reactjs.org/docs/reconciliation.html
1. PureComponents - https://reactjs.org/docs/react-api.html#reactpurecomponent
1. React.memo - https://reactjs.org/docs/react-api.html#reactmemo
1. React Context - https://reactjs.org/docs/context.html
1. Component Composition -
   https://reactjs.org/docs/composition-vs-inheritance.html
1. Avoid passing callbacks down -
   https://reactjs.org/docs/hooks-faq.html#how-to-avoid-passing-callbacks-down
