# State management in React

In ReactJS, a React component translate raw data into rich HTML for use in the
UI. These data are represented by two things - `state` and `props`.

The main difference between `props` and `state` is that `props` are passed in,
and `state` is maintained by the component itself.

## When do we use `state`?

- when the data should be created, and maintained by the component
- preferably as little as possible (because they add complexity)

## And when do we use `props`?

- when the data is considered read-only in that component i.e. the data is not
  meant to be changed by the receiving component

## Various ways of state management in React

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

## Passing data from parent to child

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

## Passing data from child to parent

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

## Passing data between siblings (or cousins, or distant relatives)

Sometimes, data needs to be shared amongst siblings and can be updated by either
parent, or the children themselves.

There are several ways, and the two most common ways are:

1. Using a callback function in a **common** "ancestor"
2. Using React Context API

### Using a callback function in a common parent

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

### Using React Context API

Another way we can pass data between siblings (or cousins, or distant relatives)
is through the [React Context](https://reactjs.org/docs/context.html).

Context is designed to share data that can be considered "global" for a
**tree** of React components.

It can be used to prevent props-drilling.

An alternative to Context is to use
[Component Composition](https://reactjs.org/docs/composition-vs-inheritance.html)

### Example of using Context

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
