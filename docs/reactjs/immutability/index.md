# Immutability and React

In this article, we shall explore the topic of immutability and why it is
important to React.

## Contents

- [What is immutability](#what-is-immutability)
- [How React uses immutability to improve performance](#how-React-uses-immutability-to-improve-performance)
- [So how does immutability comes into play?](#so-how-does-immutability-comes-into-play)

## What is immutability

An immutable object is an object whose state (its content) cannot be modified.

This simple concept has a profound impact on performance in React.

## How React uses immutability to improve performance

Before we can go into how immutability is used to improve performance, we need
to first understand the definition of performance in React.

In React, the main purpose of a component is to translate raw data given to it
into rich HTML, which represents the UI that users will see. For the ease of
this discussion, we shall define that a perfomant UI is one that is
highly-responsive.

So, what might cause slow response in a React application? Each user interaction
brings about some change in the application state; Each time the state changes,
React will perform a re`render`-ing of the UI; and `render()` is an expensive
operation in React.

So each time there is a change to `state` or `props`, React will `render` the
affected component. Each time a component is `render`-ed, React will `render`
its children components too.

Fret not. React has a way to deal with this recursive nightmare.

Each React component has lifecycles for the mounting, updating and unmounting
stage. A good guide can be found
[here](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/).

Before we reach `render`, there is a
[`shouldComponentUpdate`](https://reactjs.org/docs/react-component.html#shouldcomponentupdate)
method. Do note that these lifecycle methods are only available in class
components, as functional components are meant to be **stateless**.

### How `shouldComponentUpdate` works

The `shouldComponentUpdate` method has the following form:

```ts
shouldComponentUpdate(nextProps: {} , nextState: {}): boolean
```

The `nextProps` parameter is the updated props that has been passed into the component, and the `nextState` parameter is the state that has been updated.
As the name suggests, `shouldComponentUpdate` returns `true` if the component
should update, and `false` otherwise. By default, `true` is returned.

From the official React docs, we should not be using the `shouldComponentUpdate`
method to prevent `render`-ing. The `shouldComponentUpdate` method should only
be used to optimize performance.

React has a built-in
[`PureComponent`](https://reactjs.org/docs/react-api.html#reactpurecomponent)
that performs shallow comparison of the `props` and `state` to decide if
`render` should be performed. For functional components, we have
[`React.memo`](https://reactjs.org/docs/react-api.html#reactmemo).

The `React.memo` method has the following form:

```js
React.memo(functionalComponent, areEqualFunction): functionalComponent
```

The `functionalComponent` parameter is the React functional component that needs
to be memoized. The `areEqualFunction` parameter is a function that returns
`true` if the component need not be re`render`-ed, and `false` otherwise. By
default, React will perform a shallow comparison on the `props` object. The
`React.memo` function returns the Higher-Order-Component (HOC) that wraps the
passed in `functionalComponent` and perform the `areEqualFunction` check to
determine if the component should re`render`.

## So how does immutability comes into play?

When React performs a shallow comparison of the `props` object, what React
actually do is to perform a referential check on the keys of the `props` object.

In Javascript, a referential check is done using `===`.

```js
1 === 1             // true
"hello" === "hello" // true
true === true       // true

[] === []           // false
{} === {}           // false

const obj = {};     // assigned a memory address
const obj2 = obj;   // created another pointer to the same address
obj === obj2        // true, as both point to the same address
```

For primitives, as long as the values are the same, the check will return
true.

For arrays and objects, the check will return true if memory address is the
same.

And Javascript has more to this than meets the eye.

### Javascript's spread operator (`...`)

Consider the following:

```js
const someObject = {
  str: "str",
  num: 2,
  simpleList: [42],
  complexList: [{ id: 1, name: "one" }],
  map: { a: 65 }
};

const anotherObject = {
  ...someObject
};

someObject === anotherObject; // false, as expected

someObject.str === anotherObject.str; // true!
someObject.num === anotherObject.num; // true!
someObject.simpleList === anotherObject.simpleList; // true!
someObject.complexList === anotherObject.complexList; // true!
someObject.map === anotherObject.map; // true!
```

It might seem strange, but what happened was that when we use the spread
operator (`...`) on an object, each key and the value it points to are "copied"
to the receiving object. Each value in this new object actually uses a new
pointer to the original memory address.

Now, consider that we overwrite the values manually:

```js
// continuing from previous example

const yetAnotherObject = {
  ...someObject,
  // here we are overwriting the value of map, as it exists in someObject
  map: {
    ...someObject.map,
    a: "a"
  }
};

someObject.str === yetAnotherObject.str; // true
someObject.num === yetAnotherObject.num; // true
someObject.simpleList === yetAnotherObject.simpleList; // true
someObject.complexList === yetAnotherObject.complexList; // true
someObject.map === yetAnotherObject.map; // false!
```

Notice that the `map` value has changed in this example, and that causes the
referential check return `false`.

What we seen from these two examples will be crucial in how React uses
immutability to optimize performance.

### Examples

Let's say we have the following component.

```js
class ClassComponent extends React.PureComponent {
  constructor(props) {
    super(props);
  }

  render() {
    return (
      <div>
        {props.someArray.map(item => (
          <div key={item.id}>{item.name}</div>
        ))}
      </div>
    );
  }
}
```

When we mount the component, there is already some `props` passed in. Let's say
it is the following:

```js
const props = {
  someArray: [{ id: 1, name: "one" }, { id: 2, name: "two" }]
};
```

As a recap, each time `props` changes, React will perform a shallow comparison
of the `props` to decide if a re`render`-ing is needed.

How the shallow comparison works is as such:

```js
const shallowComparePropsTo = nextProps => {
  for (var key in nextProps) {
    if (nextProps[key] !== props[key]) return false;
  }
  return true;
};
```

Remember that part about how the
[spread operator](#javascript_s-spread-operator) works? The
`shallowComparePropsTo` function will return `false` if any of the keys have
been reassigned when:

- the value itself has changed
- a value in the list has changed
- a value in the object has changed
- a combination of any of the above

And that will trigger the `render` function of that component.

For functional components, we have:

```js
const areEqual = (prevProps, nextProps) => {
  /* a custom comparison function that returns true/false */
};
const APureComponent = React.memo(props => {
  return (
    <div>
      {props.someArray.map(item => (
        <div key={item.id}>{item.name}</div>
      ))}
    </div>
  );
}, areEqual);
```

## In summary

Immutability plays a big part to ensure optimal performance in React.

---

## References

1. React lifecycle method - [http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)
1. React's `shouldComponentUpdate` - [https://reactjs.org/docs/react-component.html#shouldcomponentupdate)](https://reactjs.org/docs/react-component.html#shouldcomponentupdate)
1. `React.PureComponent` - [https://reactjs.org/docs/react-api.html#reactpurecomponent)](https://reactjs.org/docs/react-api.html#reactpurecomponent)
1. `React.memo` - [https://reactjs.org/docs/react-api.html#reactmemo](https://reactjs.org/docs/react-api.html#reactmemo)
