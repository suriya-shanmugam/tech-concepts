# React, Browser Engine, DOM, HTML, CSS, JavaScript, and Why React Exists

## 1. Big picture

A browser application is built using three core technologies:

```txt
HTML        = structure
CSS         = style
JavaScript  = behavior
```

The browser takes these files, understands them, and turns them into pixels on the screen.

React is built **on top of** HTML, CSS, JavaScript, and the browser. React does not replace them. Instead, React gives developers a better way to build and manage complex, dynamic user interfaces.

---

# 2. Browser engine

A **browser engine** is the part of the browser that reads HTML, CSS, and JavaScript, then renders the webpage.

Examples:

| Browser       | Engine |
| ------------- | ------ |
| Chrome / Edge | Blink  |
| Safari        | WebKit |
| Firefox       | Gecko  |

The browser engine does this:

```txt
HTML file
  ↓
Parse HTML
  ↓
Create DOM tree

CSS file
  ↓
Parse CSS
  ↓
Create CSSOM tree

JavaScript
  ↓
Read or change DOM

DOM + CSSOM
  ↓
Render tree
  ↓
Layout
  ↓
Paint
  ↓
Pixels on screen
```

So when a page appears in the browser, it is not just “showing HTML.” The browser has parsed the code, built internal trees, calculated styles and layout, and painted the final result.

---

# 3. HTML

**HTML** defines the structure and meaning of the page.

Example:

```html
<h1>Todo App</h1>

<ul>
  <li>Learn HTML</li>
  <li>Learn CSS</li>
  <li>Learn JavaScript</li>
</ul>

<button>Add Todo</button>
```

HTML tells the browser:

```txt
There is a heading.
There is a list.
There are list items.
There is a button.
```

HTML is mainly about **content structure**, not dynamic behavior.

---

# 4. DOM

The **DOM**, or Document Object Model, is the browser’s in-memory representation of the HTML document.

HTML is text:

```html
<h1>Hello</h1>
<button>Click me</button>
```

The browser converts it into a tree:

```txt
document
  └── html
      └── body
          ├── h1
          │   └── "Hello"
          └── button
              └── "Click me"
```

JavaScript can access and change this DOM:

```js
const heading = document.querySelector("h1");
heading.textContent = "Hello React";
```

So the DOM is the bridge between the HTML document and JavaScript behavior.

Important point:

```txt
JavaScript usually does not change the original HTML file.
JavaScript changes the DOM currently loaded in the browser.
```

---

# 5. CSS

**CSS** controls how HTML elements look.

Example:

```css
h1 {
  color: blue;
  font-size: 32px;
}

button {
  background: black;
  color: white;
}
```

CSS controls things like:

```txt
Color
Size
Spacing
Layout
Position
Visibility
Animation
Responsive behavior
```

The browser combines:

```txt
DOM   = what elements exist
CSS   = how those elements should look
```

Then the browser calculates layout and paints the page.

---

# 6. JavaScript

**JavaScript** adds behavior and interactivity.

Example:

```html
<h1 id="count">0</h1>
<button id="btn">Increment</button>

<script>
  let count = 0;

  const countElement = document.getElementById("count");
  const button = document.getElementById("btn");

  button.addEventListener("click", function () {
    count = count + 1;
    countElement.textContent = count;
  });
</script>
```

JavaScript does four main things here:

```txt
Stores data
Finds DOM elements
Listens for user events
Updates the DOM manually
```

This works well for small pages. But as applications grow, manual DOM management becomes difficult.

---

# 7. Plain HTML/CSS/JavaScript design

In plain web development, the typical design is:

```txt
HTML creates structure.
CSS styles the structure.
JavaScript finds elements and changes them manually.
```

Example:

```html
<p id="count">0</p>
<button id="increment">Increment</button>
```

```js
let count = 0;

const countText = document.getElementById("count");
const button = document.getElementById("increment");

button.addEventListener("click", () => {
  count++;
  countText.textContent = count;
});
```

This is called an **imperative** style.

You tell the browser step by step:

```txt
Find this element.
Listen for this event.
Change this text.
Add this class.
Remove that element.
Create this node.
Append it there.
```

This gives direct control, but it becomes hard to manage in large applications.

---

# 8. Problems in plain design

## Problem 1: UI and data can go out of sync

Example:

```js
let isLoggedIn = false;
```

But the DOM may still show:

```html
<p>Welcome, John</p>
<button>Logout</button>
```

Now the data and UI disagree.

In plain JavaScript, you must manually remember to update every affected part of the UI:

```js
if (isLoggedIn) {
  showDashboard();
  hideLoginForm();
  updateNavbar();
  loadUserMenu();
} else {
  hideDashboard();
  showLoginForm();
  updateNavbar();
  clearUserMenu();
}
```

As the app grows, this becomes hard to maintain.

---

## Problem 2: Too much manual DOM manipulation

Plain JavaScript often requires code like this:

```js
const li = document.createElement("li");
li.textContent = todo.title;

const deleteButton = document.createElement("button");
deleteButton.textContent = "Delete";

li.appendChild(deleteButton);
todoList.appendChild(li);
```

This code explains **how to build the UI step by step**, but the final UI structure is not easy to see.

---

## Problem 3: Code becomes scattered

One feature may be split across many places:

```txt
HTML file
CSS file
JavaScript selectors
JavaScript event listeners
JavaScript DOM update functions
Validation logic
State logic
```

For example:

```js
document.querySelector("#save-button").addEventListener("click", saveUser);
document.querySelector("#name-input").value = user.name;
document.querySelector("#error-message").textContent = error;
document.querySelector("#modal").classList.add("open");
```

The logic is scattered, and the relationship between data and UI is not obvious.

---

## Problem 4: Selectors are fragile

Plain JavaScript often depends on selectors:

```js
document.querySelector(".user-card .title");
```

If someone changes the HTML class names:

```html
<div class="profile-card">
  <h2 class="heading">John</h2>
</div>
```

Then the JavaScript selector may break.

So structure, style, and behavior become tightly connected in a fragile way.

---

## Problem 5: Reusing UI is difficult

Suppose you need a user card in many places.

In plain JavaScript, you may write:

```js
function createUserCard(user) {
  const div = document.createElement("div");
  div.className = "user-card";

  const name = document.createElement("h2");
  name.textContent = user.name;

  div.appendChild(name);

  return div;
}
```

This works, but for complex UI it becomes difficult to read, reuse, and maintain.

---

## Problem 6: Updating lists is hard

For a todo list, you may need to handle:

```txt
Add item
Delete item
Edit item
Mark complete
Filter list
Sort list
Preserve input focus
Preserve item state
```

In plain JavaScript, you must manually decide what to create, update, move, or remove.

---

## Problem 7: Performance mistakes are easy

DOM operations can be expensive if done carelessly.

Example:

```js
for (let i = 0; i < 1000; i++) {
  const item = document.createElement("li");
  item.textContent = i;
  list.appendChild(item);
}
```

This can cause unnecessary DOM work.

Also, repeatedly reading and writing layout values can force the browser to recalculate layout many times.

---

# 9. Why React was designed

React was designed to solve this main problem:

```txt
As UI becomes more dynamic, manually keeping data, DOM, events, and visual state in sync becomes difficult.
```

React changes the question from:

```txt
How do I manually change the page?
```

to:

```txt
What should the page look like for the current state?
```

This is the most important React design idea.

---

# 10. React’s core mental model

React follows this idea:

```txt
UI = function of state
```

Or:

```txt
State / Props / Context
        ↓
Component render
        ↓
UI description
        ↓
React updates DOM
```

Instead of manually editing DOM elements, you describe the UI for a given state.

---

# 11. Components

A **component** is a reusable piece of UI.

Example:

```jsx
function UserCard({ user }) {
  return (
    <div className="user-card">
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

You can reuse it:

```jsx
<UserCard user={john} />
<UserCard user={mary} />
<UserCard user={alex} />
```

A component can contain:

```txt
UI structure
State
Event handling
Rendering logic
Reusable behavior
```

Components solve the problem of scattered UI logic by grouping related logic together.

---

# 12. JSX

React uses **JSX**, which looks like HTML inside JavaScript.

Example:

```jsx
function Product({ product }) {
  return (
    <article>
      <h2>{product.name}</h2>
      <p>${product.price}</p>
      <button>Add to cart</button>
    </article>
  );
}
```

JSX makes the UI structure easier to read.

Without JSX, you might write:

```js
const article = document.createElement("article");

const title = document.createElement("h2");
title.textContent = product.name;

article.appendChild(title);
```

JSX lets you describe the final UI more clearly.

Important:

```txt
JSX is not HTML.
JSX is JavaScript syntax that describes UI.
React eventually turns it into real DOM elements.
```

---

# 13. State

State means data that can change over time.

Example:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </>
  );
}
```

Here:

```txt
count       = current data
setCount    = function to update data
```

When state changes:

```txt
State update happens
        ↓
React calls the component again
        ↓
Component returns new UI description
        ↓
React updates the DOM
```

You do not manually write:

```js
countElement.textContent = count;
```

React handles the DOM update.

---

# 14. Props

**Props** are inputs passed from a parent component to a child component.

Example:

```jsx
function App() {
  const user = { name: "John" };

  return <Profile user={user} />;
}

function Profile({ user }) {
  return <h1>{user.name}</h1>;
}
```

Data usually flows downward:

```txt
Parent component
  ↓ props
Child component
  ↓ props
Grandchild component
```

This is called **one-way data flow**.

It makes applications easier to understand because data has a clear direction.

---

# 15. Declarative rendering

Plain JavaScript is usually imperative:

```js
if (isLoggedIn) {
  showDashboard();
  hideLoginForm();
} else {
  hideDashboard();
  showLoginForm();
}
```

React is declarative:

```jsx
function App() {
  const [isLoggedIn, setIsLoggedIn] = useState(false);

  return (
    <div>
      {isLoggedIn ? <Dashboard /> : <LoginForm />}
    </div>
  );
}
```

You describe the result:

```txt
If logged in, show Dashboard.
If not logged in, show LoginForm.
```

React decides how to update the DOM.

---

# 16. Render and commit

React works in two major phases:

```txt
Render phase
Commit phase
```

## Render phase

React calls your components and calculates what the UI should look like.

```txt
Component function runs
        ↓
Returns JSX / React elements
        ↓
React builds next UI description
```

Rendering should be pure.

That means:

```txt
Same input should produce same output.
Rendering should not directly change the DOM.
Rendering should not start timers or subscriptions.
Rendering should not perform random side effects.
```

## Commit phase

After React calculates what changed, it updates the real DOM.

```txt
Old UI description
        ↓ compare
New UI description
        ↓
React finds changes
        ↓
React commits changes to DOM
        ↓
Browser paints the screen
```

---

# 17. Reconciliation

**Reconciliation** is React’s process of comparing the old UI tree with the new UI tree.

Example:

```jsx
// Before
<div>
  <h1>Hello</h1>
  <p>Old message</p>
</div>

// After
<div>
  <h1>Hello</h1>
  <p>New message</p>
</div>
```

React sees:

```txt
div is same
h1 is same
p is same, but text changed
```

So React only updates the changed text.

This is better than recreating the entire page manually.

---

# 18. Keys

Keys help React understand identity in lists.

Example:

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  );
}
```

Keys tell React:

```txt
This item is the same item as before.
This item is new.
This item was removed.
This item moved.
```

Good key:

```jsx
<li key={todo.id}>{todo.title}</li>
```

Risky key:

```jsx
<li key={index}>{todo.title}</li>
```

Using array index as a key can cause bugs when items are inserted, deleted, or reordered.

---

# 19. Fiber

React’s modern internal architecture is called **Fiber**.

A simplified explanation:

```txt
React element tree = description of UI
Fiber tree         = React's internal work structure
```

Fiber helps React split rendering work into smaller units.

This allows React to:

```txt
Pause work
Resume work
Prioritize urgent updates
Discard unfinished work
Reuse previous work
```

For example:

```txt
Typing in an input     = urgent
Rendering a big list   = less urgent
```

React can prioritize the urgent update first.

---

# 20. Scheduling

React does not only ask:

```txt
What changed?
```

It also asks:

```txt
When should this work happen?
How urgent is this update?
Can this work be interrupted?
```

This is important for smooth user experience.

For example:

```txt
User typing in input should feel instant.
Rendering a large filtered list can be delayed slightly.
```

React’s scheduler and Fiber architecture help manage this kind of work.

---

# 21. Hooks

Hooks let function components use React features such as state, effects, refs, memoization, and transitions.

Example:

```jsx
function SearchBox() {
  const [query, setQuery] = useState("");

  return (
    <input
      value={query}
      onChange={(event) => setQuery(event.target.value)}
    />
  );
}
```

Here, React keeps the input value and component state connected.

For side effects, React uses `useEffect`:

```jsx
useEffect(() => {
  document.title = `Search: ${query}`;
}, [query]);
```

This separates:

```txt
Rendering UI
from
Interacting with the outside world
```

Examples of side effects:

```txt
Changing document title
Fetching data
Subscribing to events
Using timers
Working with browser APIs
```

---

# 22. Why Hooks must follow rules

Hooks must be called in the same order on every render.

Bad:

```jsx
if (isLoggedIn) {
  const [name, setName] = useState("");
}
```

Good:

```jsx
const [name, setName] = useState("");

if (!isLoggedIn) {
  return null;
}
```

React tracks Hook state based on the component’s render order. If the order changes, React may connect the wrong state to the wrong Hook.

---

# 23. State preservation

React preserves component state based on position in the UI tree.

Example:

```jsx
{showA ? <Counter /> : <Counter />}
```

Even though there are two `<Counter />` expressions in the code, React may treat them as the same component position.

Keys can change identity:

```jsx
<Profile key={user.id} user={user} />
```

When `user.id` changes, React treats it as a different component and resets its state.

So:

```txt
Same position + same key      = preserve state
Different position/key        = reset state
```

---

# 24. Batching updates

React can group multiple state updates together.

Example:

```jsx
setFirstName("John");
setLastName("Smith");
setAge(30);
```

Instead of updating the DOM after every single state change, React can batch them and update once.

This helps performance and avoids unnecessary DOM work.

---

# 25. Plain JavaScript counter vs React counter

## Plain JavaScript

```html
<p id="count">0</p>
<button id="increment">Increment</button>

<script>
  let count = 0;

  const countElement = document.getElementById("count");
  const button = document.getElementById("increment");

  button.addEventListener("click", () => {
    count = count + 1;
    countElement.textContent = count;
  });
</script>
```

You manage:

```txt
State
DOM selection
Event listener
DOM update
```

## React

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </>
  );
}
```

You manage:

```txt
State meaning
UI description
User interaction logic
```

React manages:

```txt
DOM update
Re-rendering
Reconciliation
Efficient committing
```

---

# 26. Plain design vs React design

| Topic             | Plain HTML/CSS/JS                   | React                                    |
| ----------------- | ----------------------------------- | ---------------------------------------- |
| UI update style   | Imperative                          | Declarative                              |
| DOM updates       | Manual                              | React-managed                            |
| Code organization | Often scattered                     | Component-based                          |
| State handling    | Manual                              | State-driven rendering                   |
| Reuse             | Manual functions/templates          | Components                               |
| List updates      | Manual create/update/remove         | Keys + reconciliation                    |
| Performance       | Developer must manage more directly | React batches and optimizes many updates |
| Data flow         | Can become tangled                  | Usually one-way                          |
| Side effects      | Often mixed with UI logic           | Separated with effects                   |

---

# 27. Important: React does not replace browser knowledge

React still depends on the browser.

A React element like this:

```jsx
<button>Save</button>
```

eventually becomes a real DOM element:

```html
<button>Save</button>
```

The browser still handles:

```txt
HTML parsing
CSS styling
Layout
Painting
Events
Accessibility behavior
Performance constraints
```

So to become good at React, you still need to understand:

```txt
HTML
CSS
JavaScript
DOM
Browser rendering
Accessibility
Performance
```

React makes UI development easier, but it does not remove the need to understand the web platform.

---

# 28. Final mental model

## Browser model

```txt
HTML
  ↓
DOM

CSS
  ↓
CSSOM

DOM + CSSOM
  ↓
Render tree
  ↓
Layout
  ↓
Paint
  ↓
Screen
```

## Plain JavaScript model

```txt
Data changes
  ↓
Developer manually finds DOM elements
  ↓
Developer manually updates DOM
  ↓
Browser recalculates and paints
```

## React model

```txt
State changes
  ↓
React re-renders components
  ↓
React creates new UI description
  ↓
React compares old UI and new UI
  ↓
React updates only needed DOM parts
  ↓
Browser recalculates and paints
```

---

# 29. Core takeaway

Plain JavaScript asks:

```txt
How do I change this element on the page?
```

React asks:

```txt
What should the UI look like for the current state?
```

That is the main design shift.

React’s purpose is not just to make HTML shorter. Its purpose is to make large, dynamic UIs easier to reason about by using:

```txt
Components
State
Props
JSX
Declarative rendering
One-way data flow
Reconciliation
Keys
Hooks
Fiber
Scheduling
Batching
Render/commit separation
```

The browser still renders the final page, but React helps decide **what should change**, **when it should change**, and **how to keep the UI synchronized with application state**.

### Getting Started
- [Thinking in React](https://react.dev/learn/thinking-in-react)
- [Tutorial: Tic-Tac-Toe](https://react.dev/learn/tutorial-tic-tac-toe)