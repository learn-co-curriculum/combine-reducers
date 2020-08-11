# Redux Combine Reducers

## Objectives

1. Implement multiple reducers in an application
2. Use the `combineReducers()` function to delegate different pieces of state to each reducer

## Introduction

So far we have been using a single reducer to return a new state when an action
is dispatched. This works great for a small application where we only need our
reducer to manage the state of one resource. However, as you will see, when
working with multiple resources, placing all of this logic in one reducer
function can quickly become unwieldy.

Enter `combineReducers()` to save the day! In this lab, we'll see how
**Redux**'s `combineReducers()` function lets us delegate different pieces of
state to separate reducer functions.

We'll do this in the context of a book application that we'll use to keep track
of programming books that we've read.

We want our app to do two things:

1. Keep track of all the books we've read: title, author, description.
2. Keep track of the authors who wrote these books.

#### Determine Application State Structure

Our app will need a state object that stores two types of information:

1. All our books, in an array
2. Our authors, also in an array

Each of these types of information--all our books, and the authors--should be
represented on our store's state object. If we think of our store's state
structure as a database, we can represent this as a belongs to/has many
relationship, in that a book belongs to an author and an author has many books.
So this means each author would have their own id, and each book would have an
authorId as a foreign key.

With that, we can set the application state as:

```
{
  authors: //array of authors
  books: // array of books,
}
```

So our state object will have two top-level keys, each pointing to an array. For
now, let's write a single reducer to manage both of these resources.

```javascript
// src/reducers/manageAuthorsAndBooks.js

export default function bookApp(
  state = {
    authors: [],
    books: []
  },
  action
) {
  let idx;
  switch (action.type) {
    case "ADD_BOOK":
      return {
        ...state,
        authors: [...state.authors],
        books: [...state.books, action.book]
      };

    case "REMOVE_BOOK":
      idx = state.books.findIndex(book => book.id === action.id);
      return {
        ...state,
        authors: [...state.authors],
        books: [...state.books.slice(0, idx), ...state.books.slice(idx + 1)]
      };

    case "ADD_AUTHOR":
      return {
        ...state,
        books: [...state.books],
        authors: [...state.authors, action.author]
      };

    case "REMOVE_AUTHOR":
      idx = state.authors.findIndex(author => author.id === action.id);
      return {
        ...state,
        books: [...state.books],
        authors: [...state.authors.slice(0, idx), ...state.authors.slice(idx + 1)]
      };

    default:
      return state;
  }
}
```

This is the current setup in `src/reducers/manageAuthorsAndBooks.js`, and if you boot up the application you'll see that it works. You can see, however, by working with just two resources, that the amount of code in our reducer is already getting pretty substantial. Moreover, placing the resources in the same reducer violates our goal of maintaining separation of concerns. By creating separate reducers for each resource in an application, we are better able to keep our code organized as our applications get more complicated.

> **NOTE:** You may have noticed something in the reducer example: when we
update one part of `state`, we're also using the spread operator _on other
parts_. For example, in the `"ADD_AUTHOR"` case, we add `action.author` to the
`authors` array, but **we also use the spread operator to create a new `book`
array**. This is because both `Object.assign()` and the spread operator only
create shallow copies of objects. If we leave out `books: [...state.books]`,
and just write the following:

```js
return {
        ...state,
        authors: [...state.authors, action.author]
};
```

a new reference to the old `state.books` array will be used, _not a new copy of
the array_. This is subtle, and can easily be overlooked, but by referencing the
old state, we are no longer maintaining an immutable state. [The official redux
documentation][] goes into further detail on the benefits of immutability, [discusses this exact issue][], and [provides further examples][] of how to properly use the spread operator to deeply copy nested data.

[The official redux documentation]: https://redux.js.org/faq/immutable-data#what-are-the-benefits-of-immutability
[discusses this exact issue]: https://redux.js.org/faq/immutable-data#accidental-object-mutation
[provides further examples]: https://redux.js.org/recipes/structuring-reducers/immutable-update-patterns

## Refactor by using combineReducers

**Redux**'s `combineReducers()` function will allow us to write separate
reducers for each of the resources in our application. Each reducer is passed to the `combineReducers()` function, which will produce a single reducer such as the one we wrote above. We then pass that combined reducer to the store in
`src/index.js`. Let's write some code, and then we'll walk through it below.

```javascript
// src/reducers/manageAuthorsAndBooks.js

import { combineReducers } from "redux";

const rootReducer = combineReducers({
  authors: authorsReducer,
  books: booksReducer
});

export default rootReducer;

function booksReducer(state = [], action) {
  let idx;
  switch (action.type) {
    case "ADD_BOOK":
      return [...state, action.book];

    case "REMOVE_BOOK":
      idx = state.findIndex(book => book.id  === action.id)
      return [...state.slice(0, idx), ...state.slice(idx + 1)];

    default:
      return state;
  }
}

function authorsReducer(state = [], action) {
  let idx;
  switch (action.type) {
    case "ADD_AUTHOR":
      return [...state, action.author];

    case "REMOVE_AUTHOR":
      idx = state.findIndex(author => author.id  === action.id)
      return [...state.slice(0, idx), ...state.slice(idx + 1)];

    default:
      return state;
  }
}
```

There's a lot of code there, so let's unpack it a bit. At the very top you see
the following lines:

```javascript
import { combineReducers } from "redux";

const rootReducer = combineReducers({
  authors: authorsReducer,
  books: booksReducer
});

export default rootReducer;
```

Through `combineReducer`, we're telling **Redux** to produce a reducer which
will return a state that has both a key of books with a value equal to the
return value of the `booksReducer()` _and_ a key of **authors** with a value
equal to the return value of the `authorsReducer()`. Now if you look at the
`booksReducer()` and the `authorsReducer()` you will see that each returns a
default state of an empty array. This will produce the same initial state that we originally specified when we built the combined reducer ourselves:

```js
  state = {
    authors: [],
    books: []
  }
```

Note that because the `rootReducer` is the default export of `manageAuthorsAndBooks.js`, the app will continue to work with the code that's currently in the `index.js` file. However, if we like, we can update the file to reflect the new name:

```js
// 
import { createStore } from "redux";
import rootReducer from "./reducers/manageAuthorsAndBooks";

const store = createStore(
  rootReducer,
  window.__REDUX_DEVTOOLS_EXTENSION__ && window.__REDUX_DEVTOOLS_EXTENSION__()
);
```

As mentioned above, the initial state of the rootReducer is the same as the state we created when we had a single reducer for both resources. Here we are passing our rootReducer to the createStore method, so from the application's perspective nothing has changed.

#### Examining Our New Reducers

Now if we examine the `authorsReducer()`, notice that this reducer only concerns itself with its own slice of the state. This makes sense. Remember that ultimately the array that the `authorsReducer()` returns will be the value associated with the key of authors in our application state object. Consequently, the `authorsReducer()` should only receive as its state argument the value of `state.authors`, in other words, the authors array. This means that we no longer need to retrieve the
list of authors with a call to `state.authors`, but instead can access it simply by calling `state`.

```javascript
function authorsReducer(state = [], action) {
  let idx;
  switch (action.type) {
    case "ADD_AUTHOR":
      return [...state, action.author];

    case "REMOVE_AUTHOR":
      idx = state.findIndex(author => author.id === action.id);
      return [...state.slice(0, idx), ...state.slice(idx + 1)];

    default:
      return state;
  }
}
```

#### Dispatching Actions

The `combineReducers()` function returns to us one large reducer that looks like
the following:

```javascript
function reducer(state = {
  authors: [],
  books: []
}, action) {
  let idx
  switch (action.type) {

    case "ADD_AUTHOR":
      return [...state, action.author]

    case 'REMOVE_AUTHOR':
      ...
  }
}
```

Because of this, we can dispatch actions the same way we always did. 
`store.dispatch({ type: 'ADD_BOOK', { title: 'Snow Crash', author: 'Neal Stephenson' } });` will hit our switch statement in the reducer and add a new author. One thing to note, is that if you want to have more than one reducer respond to the same action, you can.

For example, in our application, when a user inputs information about a book,
the user _also_ inputs the author's name. It would be handy if, when a user
submits a book with an author, that author is also added to our author array.

The action dispatched doesn't change: `store.dispatch({ type: 'ADD_BOOK', { title: 'Snow Crash', author: 'Neal Stephenson' } });`. Our `booksReducer` can stay the same for now:

```javascript
function booksReducer(state = [], action) {
  let idx;
  switch (action.type) {
    case "ADD_BOOK":
      return [...state, action.book];

    case "REMOVE_BOOK":
      idx = state.findIndex(book => book.id === action.id);
      return [...state.slice(0, idx), ...state.slice(idx + 1)];

    default:
      return state;
  }
}
```

However, in `authorsReducer`, we can _also_ include a switch case for
"ADD_BOOK":

```js
import uuid from "uuid";

function authorsReducer(state = [], action) {
  let idx;
  switch (action.type) {
    case "ADD_AUTHOR":
      return [...state, action.author];

    case "REMOVE_AUTHOR":
      idx = state.findIndex(book => book.id === action.id);
      return [...state.slice(0, idx), ...state.slice(idx + 1)];

    case "ADD_BOOK":
      let existingAuthor = state.filter(
        author => author.authorName === action.book.authorName
      );
      if (existingAuthor.length > 0) {
        return state;
      } else {
        return [...state, { authorName: action.book.authorName, id: uuid() }];
      }

    default:
      return state;
  }
}
```

In the new "ADD_BOOK" case, we're checking to see if any of the `authorName`s currently stored in state matches with the name dispatched from the BookInput component. If the name already exists, state is returned unchanged. If the name is not present, it is added to the author array. Use the example above to modify the `manageAuthorsAndBooks` reducer and you can see the effect. We have two separate forms, one for adding just authors, and one that adds books _and_ authors.

**Note:** We're using a useful package, `uuid`, to handle unique ID generation.
With this refactor, since we are creating an author ID from within the reducer
instead of in `AuthorInput.js`, we need to import it here as well.

## A Note about Organization

For this codealong we have coded our two reducers in the same file, but it is common to separate each reducer into its own file. You could then import each reducer into a _new_ file, something like `reducers/rootReducer.js`, where `combineReducer` is called. Or, alternatively, you could include `combineReducer` in your `src/index.js` file. For example:

```js
import authorsReducer from './reducers/authorsReducer';
import booksReducer from './reducers/booksReducer';

const rootReducer = combineReducers({
  books: booksReducer,
  authors: authorsReducer
})

const store = createStore(rootReducer, window.__REDUX_DEVTOOLS_EXTENSION__ && window.__REDUX_DEVTOOLS_EXTENSION__())

...
```

In React/Redux apps where we're using and storing many resources in our store,
keeping reducers separated helps us organize code and separate concerns. Because `combineReducers` creates a single `rootReducer` for us, when an action is dispatched it can cause multiple reducers to modify their state, while still allowing us to keep all modifications to a _particular_ resource within its own separate file.

#### Resources

- [Implementing Combine Reducers from Scratch](https://egghead.io/lessons/javascript-redux-implementing-combinereducers-from-scratch)

