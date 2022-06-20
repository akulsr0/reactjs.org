---
title: 'You Might Not Need an Effect'
---

<Intro>

Effects are an escape hatch from the React paradigm. They let you "step outside" of React and synchronize your components with some external system like a non-React widget, network, or the browser DOM. If there is no external system involved (for example, if you want to update a component's state when some props or state change), you shouldn't need an Effect. Removing unnecessary Effects will make your code easier to follow, faster to run, and less error-prone.

</Intro>

<YouWillLearn>

* How to remove unnecessary Effects from your components

</YouWillLearn>

## How to remove unnecessary Effects {/*how-to-remove-unnecessary-effects*/}

There are two common cases in which you don't need an Effect:

* **You don't need Effects for rendering logic.** For example, say you want to filter a list of data before displaying it. You might feel tempted to write an Effect that updates a state variable in response to the data changing. However, this is inefficient. When you update your component's state, React will first call your component functions to calculate what should be on the screen. Then React will ["commit"](/learn/render-and-commit) these changes to the DOM, updating the screen. Then React will run your Effects. If your Effect *also* immediately updates the state, this restarts the whole process from scratch! To avoid the unnecessary render passes, keep all the rendering logic at the top level of your components. It will automatically re-run whenever your props or state change.
* **You don't need Effects for event-specific logic.** For example, say you want to send an `/api/buy` POST request and show a toast when the user clicks a Buy button on a product. In the Buy button event handlder, you know exactly what happened. By the time an Effect runs, you don't know *what* the user did (for example, which button was clicked). This is why you'll usually handle specific interactions in the event handlers.

You *do* need Effects to [synchronize](/learn/synchronizing-with-effects#what-are-effects-and-how-are-they-different-from-events) with external systems. For example, you can write an Effect that keeps a jQuery widget synchronized with the React state. You can also fetch data with Effects: for example, you can synchronize the search results with the current search query. Keep in mind that modern [frameworks](/learn/start-a-new-react-project#building-with-a-full-featured-framework) provide more efficient built-in data fetching mechanisms than writing Effects directly in your components.

To help you gain the right intuition, let's look at some common concrete examples!

### Updating the state based on other state {/*updating-the-state-based-on-other-state*/}

Suppose you have a component with two state variables: `firstName` and `lastName`. You want to calculate a `fullName` from them by concatenating them. Moreover, you'd like `fullName` to update whenever `firstName` or `lastName` change. Your first instinct might be to add a `fullName` state variable and update it in an effect:

```js {5-9}
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');

  // 🔴 Bad: redundant state and unnecessary Effect
  const [fullName, setFullName] = useState('');
  useEffect(() => {
    setFullName(firstName + ' ' + lastName);
  }, [firstName, lastName]);
```

This is more complicated than necessary. It is inefficient too: it does an entire render pass with a stale value for `fullName`, then immediately re-renders with the updated value. Remove both the state variable and the Effect:

```js {5-6}
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');

  // ✅ Good: calculated during rendering
  const fullName = firstName + ' ' + lastName;
```

**When something can be calculated from the existing props or state, [don't put it in state.](/learn/choosing-the-state-structure#avoid-redundant-state) Instead, calculate it during rendering.** This makes your code faster (you avoid the extra "cascading" updates), simpler (you remove some code), and less error-prone (you avoid bugs caused by different state variables getting out of sync with each other). If this approach feels new to you, [Thinking in React](/learn/thinking-in-react#step-3-find-the-minimal-but-complete-representation-of-ui-state) has some guidance on what should go into state.

### Caching expensive calculations {/*caching-expensive-calculations*/}

This component computes `visibleTodos` by taking the `todos` it receives by props and filtering them according to the `filter` prop. You might feel tempted to store the result in a state variable and update it in an Effect:

```js {4-8}
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // 🔴 Bad: redundant state and unnecessary Effect
  const [visibleTodos, setVisibleTodos] = useState([]);
  useEffect(() => {
    setVisibleTodos(getFilteredTodos(todos, filter));
  }, [todos, filter]);
```

Like in the earlier example, this is both unnecessary and inefficient. First, remove the state and the Effect:

```js {4-5}
function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // ✅ This is fine if getFilteredTodos() is not slow.
  const visibleTodos = getFilteredTodos(todos, filter);
```

In many cases, this code is fine! But maybe `getFilteredTodos()` is slow or you have a lot of `todos`. In that case you don't want to recalculate `getFilteredTodos()` if some unrelated state variable like `newTodo` has changed.

You can cache (or ["memoize"](https://en.wikipedia.org/wiki/Memoization)) an expensive calculation by wrapping it [`useMemo`](/apis/usememo) Hook:

```js {6-9}
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  const visibleTodos = useMemo(() => {
    // ✅ Does not re-run unless todos or filter change
    return getFilteredTodos(todos, filter);
  }, [todos, filter]);
```

Or, written as a single line:

```js {6-7}
import { useMemo, useState } from 'react';

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState('');

  // ✅ Does not re-run getFilteredTodos() unless todos or filter change
  const visibleTodos = useMemo(() => getFilteredTodos(todos, filter), [todos, filter]);
```

**This tells React that you don't want the inner function to re-run unless either `todos` or `filter` have changed.** React will remember the return value of `getFilteredTodos()` during the initial render. During the next renders, it will check if `todos` or `filter` are different. If they're the same as last time, `useMemo` will return the last result it has stored. But if they are different, React will call the wrapped function again (and store _that_ result instead).

The function you wrap in [`useMemo`](/apis/usememo) runs during rendering, so this only works for [pure calculations](/learn/keeping-components-pure).

<DeepDive title="How to tell if a calculation is expensive?">

In general, unless you're creating or looping over thousands of objects, it's probably not expensive. If you want to get more confidence, you can add a console log to measure the time spent in a piece of code:

```js {1,3}
console.time('filter array');
const visibleTodos = getFilteredTodos(todos, filter);
console.timeEnd('filter array');
```

Perform the interaction you're measuring (for example, typing into the input). You will then see logs like `filter array: 0.15ms` in your console. If the overall logged time adds up to a significant amount (say, `5ms` or more), it might make sense to memoize that calculation. As an experiment, you can then wrap the calculation in `useMemo` to verify whether the total logged time has significantly decreased:

```js
console.time('filter array');
const visibleTodos = useMemo(() => {
  return getFilteredTodos(todos, filter); // Skipped if todos and filter haven't changed
}, [todos, filter]);
console.timeEnd('filter array');
```

Keep in mind that your machine is probably faster than your users' so it's a good idea to test the performance with an artificial slowdown. For example, Chrome offers a [CPU Throttling](https://developer.chrome.com/blog/new-in-devtools-61/#throttling) option for this.

Also note that measuring performance in development will not give you the most accurate results. (For example, when [Strict Mode](/apis/strictmode) is on, you will see each component render twice rather than once.) To get the most accurate timings, build your app for production and test it on a device like your users have.

</DeepDive>

### Handling user interactions {/*handling-user-interactions*/}

Consider a product page with two buttons that both add that product to the shopping cart:

```js
function ProductPage({ product, addToCart }) {
  function handleBuyClick() {
    addToCart(product);
  }

  function handleCheckoutClick() {
    addToCart(product);
    navigateTo('/checkout');
  }
```

Let's say you want to show a toast whenever the user puts the product in the cart. Adding the `showToast()` call to both event handlers might feel a bit repetitive so you might be tempted to place this logic in an Effect:

```js {2-7}
function ProductPage({ product, addToCart }) {
  // 🔴 Bad: Event-specific logic inside an Effect
  useEffect(() => {
    if (product.isInCart) {
      showToast(`Added ${product.name} to the shopping cart!`);
    }
  }, [product]);

  function handleBuyClick() {
    addToCart(product);
  }

  function handleCheckoutClick() {
    addToCart(product);
    navigateTo('/checkout');
  }
```

This Effect is unnecessary. It will also most likely cause bugs. For example, let's say that your app "remembers" the shopping cart between the page reloads. If you add a product to the cart once and refresh the page, the notification toast will appear again. It will keep appearing every time you refresh that product's page. This is because `product.isInCart` will already be `true` on the page load, so the Effect above will call `showToast()`.

Delete the Effect and put the shared logic into a function that you call from both event handlers:

```js {2-6}
function ProductPage({ product, addToCart }) {
  // ✅ Good: Event-specific logic is called from event handlers
  function buyProduct() {
    addToCart(product);
    showToast(`Added ${product.name} to the shopping cart!`);    
  }

  function handleBuyClick() {
    buyProduct();
  }

  function handleCheckoutClick() {
    buyProduct();
    navigateTo('/checkout');
  }
```

This both removes the unnecessary Effect and fixes the bug.

**When you're not sure whether some code should be in an Effect or in an event handler, ask yourself *why* this code needs to run.** Effects are for code that runs *because* the component was displayed to the user. However, in this example, the toast should not appear *because* the product page was displayed! It should appear because the user _pressed the button_. This is why `showToast()` should be called from the event handler--not from an Effect.

### Resetting the state on a prop change {/*resetting-the-state-on-a-prop-change*/}

TODO

### Moar sections {/*moar-sections*/}

TODO


## Principles for writing Effects {/*principles-for-writing-effects*/}

