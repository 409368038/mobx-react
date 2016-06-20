# mobx-react

[![Build Status](https://travis-ci.org/mobxjs/mobx-react.svg?branch=master)](https://travis-ci.org/mobxjs/mobx-react)
[![Join the chat at https://gitter.im/mobxjs/mobx](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/mobxjs/mobx?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![#mobservable channel on reactiflux discord](https://img.shields.io/badge/discord-%23mobx%20%40reactiflux-blue.svg)](https://discord.gg/0ZcbPKXt5bYAa2J1)


Package with react component wrapper for combining React with mobx.
Exports the `observer` decorator and some development utilities.
For documentation, see the [mobx](https://mobxjs.github.io/mobx) project.
This package supports both React and React-Native.

## Installation

`npm install mobx-react --save`

```javascript
import {observer} from 'mobx-react';
// - or -
import {observer} from 'mobx-react/native';
// - or, for custom renderers without DOM: -
import {observer} from 'mobx-react/custom';
```

This package provides the bindings for MobX and React.
See the [official documentation](http://mobxjs.github.io/mobx/intro/overview.html) for how to get started.

## Boilerplate projects that use mobx-react

* Minimal MobX, React, ES6, JSX, Hot reloading: [MobX-React-Boilerplate](https://github.com/mobxjs/mobx-react-boilerplate)
* TodoMVC MobX, React, ES6, JSX, Hot reloading: [MobX-React-TodoMVC](https://github.com/mobxjs/mobx-react-todomvc)
* Minimal MobX, React, Typescript, TSX: [MobX-React-Typescript-Boilerplate](https://github.com/mobxjs/mobx-react-typescript-boilerplate)
* Minimal MobX, React, ES6(babel), JSPM with hot reloading modules:
[jspm-react](https://github.com/capaj/jspm-react)
* React-Native Counter: [Mobx-React-Native-Counter](https://github.com/bartonhammond/mobx-react-native-counter)


## API documentation

### observer(componentClass)

Function (and decorator) that converts a React component definition, React component class or stand-alone render function into a reactive component.
See the [mobx](https://mobxjs.github.io/mobx/refguide/observer-component.html) documentation for more details.

```javascript
import {observer} from "mobx-react";

// ---- ES5 syntax ----

const TodoView = observer(React.createClass({
    displayName: "TodoView",
    render() {
        return <div>{this.props.todo.title}</div>   
    }
}));

// ---- ES6 syntax ----

@observer class TodoView extends React.Component {
    render() {
        return <div>{this.props.todo.title}</div>   
    }   
}

// ---- or just use a stateless component function: ----

const TodoView = observer(({todo}) => <div>{todo.title}</div>)
```

### `componentWillReact` (lifecycle hook)

React components usually render on a fresh stack, so that makes it often hard to figure out what _caused_ a component to re-render.
When using `mobx-react` you can define a new life cycle hook, `componentWillReact` (pun intended) that will be triggered when a component will be scheduled to re-render because
data it observes has changed. This makes it easy to trace renders back to the action that caused the rendering.

```javascript
import {observer} from "mobx-react";

@observer class TodoView extends React.Component {
    componentWillReact() {
        console.log("I will re-render, since the todo has changed!");    
    }

    render() {
        return <div>{this.props.todo.title}</div>   
    }   
}
```

* `componentWillReact` doesn't take arguments
* `componentWillReact` won't fire before the initial render (use `componentWillMount` instead)
* `componentWillReact` won't fire when receiving new props or after `setState` calls (use `componentWillUpdate` instead)

### `Provider` (Experimental)

_This feature is marked as experimental as the exact api might change in a next minor, pending any community feedback_.

`Provider` is a component that can stores (or other stuff) on React's context.
This is useful if you have things that you don't want to pass through multiple layers of components explicitly.
By passing a string array as first argument to `observer`, observer will pick up the named stores from the context and make them available as props of de decorated component:

```javascript
@observer(["color"])
class Button extends React.Component {
  render() {
    return (
      <button style={{background: this.props.color}}>
        {this.props.children}
      </button>
    );
  }
}

class Message extends React.Component {
  render() {
    return (
      <div>
        {this.props.text} <Button>Delete</Button>
      </div>
    );
  }
}

class MessageList extends React.Component {
  getChildContext() {
    return {color: "purple"};
  }

  render() {
    const children = this.props.messages.map((message) =>
      <Message text={message.text} />
    );
    return <Provider color="red">
        <div>
            {children}
        </div>
    </Provider>;
  }
}
```

Some note about passing stores around:
* If a component ask a store and receives a store via a property with the same name, the property takes precedence. Use this to your advantage when testing!
* Values provided through `Provider` should be final, to avoid issues like mentioned in [React #2517](https://github.com/facebook/react/issues/2517) and [React #3973](https://github.com/facebook/react/pull/3973), where optimizations might stop the propagation of new context. Instead, make sure that if you put things in `context` that might change over time, that they are `@observable` or provide some other means to listen to changes, like callbacks.

## FAQ

**Should I use `observer` for each component?**

You should use `observer` on every component that displays observable data. 
Even the small ones. `observer` allows components to render independently from their parent and in general this means that
the more you use `observer`, the better the performance become.
The overhead of `observer` itself is neglectable.
See also [Do child components need `@observer`?](https://github.com/mobxjs/mobx/issues/101)


**I see React warnings about `forceUpdate` / `setState` from React**

The following warning will appear if you trigger a re-rendering between instantiating and rendering a component: 

`Warning: forceUpdate(...): Cannot update during an existing state transition (such as within `render`). Render methods should be a pure function of props and state.`

Usually this means that (another) component is trying to modify observables used by this components in their `constructor` or `getInitialState` methods.
This violates the React Lifecycle, `componentWillMount` should be used instead if state needs to be modified before mounting.

## Internal DevTools Api

### trackComponents()

Enables the tracking from components. Each rendered reactive component will be added to the `componentByNodeRegistery` and its renderings will be reported through the `renderReporter` event emitter.

### renderReporter

Event emitter that reports render timings and component destructions. Only available after invoking `trackComponents()`.
New listeners can be added through `renderReporter.on(function(data) { /* */ })`.

Data will have one of the following formats:

```javascript
{
    event: 'render',
    renderTime: /* time spend in the .render function of a component, in ms. */,
    totalTime: /* time between starting a .render and flushing the changes to the DOM, in ms. */,
    component: /* component instance */,
    node: /* DOM node */
}
```

```javascript
{
    event: 'destroy',
    component: /* component instance */,
    node: /* DOM Node */
}
```

### componentByNodeRegistery

WeakMap. It's `get` function returns the associated reactive component of the given node. The node needs to be precisely the root node of the component.
This map is only available after invoking `trackComponents`.

