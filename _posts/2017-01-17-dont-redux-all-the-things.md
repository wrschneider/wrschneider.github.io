---
layout: post
title: Don't Redux All The Things
---

Redux is a library and a pattern for managing state in front-end applications.  It is typically associated with React but it can be used with other frameworks as well.
instead of directly modifying state, components dispatch actions, which are then handled by reducers.  Reducers take current (immutable) state plus the action to produce a new state.  The new state is then wired into React component properties, which triggers re-rendering.  This interaction is shown in the diagram below:

![diagram of Redux flow](https://cdn-images-1.medium.com/max/800/1*MKL1Im4ElgB38_MsubiMjw.png)

The value in Redux is there is a clear, consistent way to manage state shared between components.  Redux is a good solution for reacting to global state changes, or for reacting to state shared between components that are not connected by parent-child relationships.  On the downside, the extra layers of indirection -- triggering an action, handling that action in a reducer, then updating state – make it more complex than just setting component state directly.  

Take this simple example of displaying a counter with a button to increment the counter:

```jsx
class RegularCounter extends React.Component {
  state = {counter: 0}

  increment = () => {
    this.setState({counter: this.state.counter + 1})
  }

  render() {
    return (
      <div>
      <h2>Regular counter</h2>
      <div>current value: {this.state.counter}</div>
      <div><button onClick={this.increment}>increment</button></div>  
      </div>
      )
  }
}
```

You have a component that encapsulates some mutable state, plus some action to change the state, and presentation.  It's clear what the `increment` function does when you click the 
button.  Now consider a Redux version:

```jsx
class _ReduxCounter extends React.Component {
  render() {
    return (
      <div>
      <h2>Redux counter</h2>
      <div>current value: {this.props.counter}</div>
      <div><button onClick={this.props.increment}>increment</button></div>  
      </div>
      )
  } 
}

const ReduxCounter = connect(
  (state) => {return {counter: state}}, /* mapStateToProps */ 
  (dispatch) => {return {increment: () => dispatch({type: 'INCREMENT'})}} /*mapDispatchToProps*/
)(_ReduxCounter)

const counterReducer = (counter = 0, action) => {
  switch (action.type) {
    case 'INCREMENT': return counter + 1
    default: return counter
  }
}
```

Now, when I look at the component it's not immediately obvious what `increment` does because it's passed in as a prop.  When I go look at the injected prop, 
that still doesn't tell me what happens when you click the button--`increment` dispatches an action, which is like publishing a message.  You have to go look at the reducer separately
to see what happens to the state when the action is received.  Why should I consider the Redux version an improvement?

My point is not that Redux is bad, only that it doesn't necessarily add value in every use case, especially if you're using React for reusable view components without
building a true full-blown SPA.  Also, the Redux docs don't have a good explanation of why a particular use case would be harder or more complex without Redux, or how to tell the difference between when it will add value and when it won't.

So, when to use Redux?  Let's hear what one of the original React creators has to say:

> Flux should only be added once many components have already been built. ... You’ll know when you need Flux. If you aren’t sure if you need it, you don’t need it.
> (source: [https://github.com/petehunt/react-howto](https://github.com/petehunt/react-howto))

It's debatable whether Redux is a Flux implementation, but it fills the same niche of state management.  

Another good rule of thumb is "do whatever is less awkward."  Don't rush to Redux All The Things.  Instead, introduce Redux gradually, when it solves a specific 
problem--you can even mix local state and Redux on the same component.
