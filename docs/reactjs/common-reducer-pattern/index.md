# Common Reducer Pattern for Javascript

In this article, we shall explore some of the common code pattern that can be
used for reducer functions in Javascript.

## Contents

- Working with objects
- Working with arrays
- Common pattern for reducer functions

## Working with objects

### Retrieving a key-value mapping from an object

```js
const { keyOfInterest: contentOfInterest } = someMapping;
// --- or ---
const { [keyOfInterest]: contentOfInterest } = someMapping;
```

### Adding a new mapping into an object

```js
const newMapping = {
  ...theRest,
  newId: newContent
};
```

### Removing a mapping from an object

```js
const { toRemove, ...theRest } = someMapping;
```

### Updating a mapping in an object

In essence, an `update` is like a `remove` then an `add` function.

```js
const { toUpdate: contentToUpdate, ...theRest } = someMapping;
const newMapping = {
  ...theRest,
  toUpdate: updatedContent
};
```

## Working with arrays

### Retrieving an object from an array

```js
const contentOfInterest = array.find(item => item.id === idOfInterest);
```

### Adding a new item to an array

```js
const newArray = [...theRest, newContent];
```

### Removing an object from an array

Note that we need some form of unique identifier in the objects in the array,
and all objects in that array has that identifier key.

```js
const theRest = array.filter(item => item.id !== idOfInterest);
```

### Updating an object in an array

Similarly, an `update` is like a `remove` then an `add` function.

```js
const theRest = array.filter(item => item.id !== idOfInterest);
const updatedArray = [...theRest, updatedContent];
```

## Common patterns for reducer functions

### Reducer function for an array

Having an array for a `state` is a little unusual for React. Nonetheless, here
is an example of a reducer function for an array as `state`.

```js
const reducer = (state = [], action) => {
  switch (action.type) {
    case "ADD_TO_LIST":
      return [...state, action.itemToAdd];
    case "REMOVE_FROM_LIST":
      return state.filter(item => item.id !== action.idToRemove);
    case "UPDATE_ITEM_IN_LIST": {
      const theRest = state.filter(item => item.id !== action.idToUpdate);
      return [
        ...theRest,
        {
          id: action.idToUpdate,
          content: action.contentToUpdate
        }
      ];
    }
    default:
      return state;
  }
};
```

### Reducer function for an object with keys pointing to arrays

It is more common to have an object as a `state`. Some times, we may have arrays
mapped to the keys in a `state` in React.

An example of a reducer for such an `state` object is as such:

```js
const reducer = (state = {}, action) => {
switch (action.type) {
  case 'ADD_TO_MAP':
    return {
      ...state,
      action.idToAdd: action.contentToAdd
    };
  case 'REMOVE_FROM_MAP': {
    const { [action.idToRemove]: contentToRemove, ...theRest } = state;
    return theRest;
  }
  case 'UPDATE_ITEM_IN_MAP': {
    const theRest = state.filter(item => item.id !== action.idToUpdate);
    return [
      ...theRest,
      {
        id: action.idToUpdate,
        content: action.contentToUpdate
      }
    ];
  }
  default:
    return state;
  }
};
```
