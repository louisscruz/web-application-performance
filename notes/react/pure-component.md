# Approaching Pure Component

In increasing the performance of React applications, one source of optimization is reducing the number of required `render` calls that are made by the application's components.

## When Does a Component Re-render?

There are four situations in which a component will re-render.

### `this.state` Changes

Whenever a component's `state` changes, a re-render is triggered.
A component's state should only ever change via `setState()`.

### `this.props` Changes

Whenever a component's `props` change, a re-render is triggered.
To tell if `this.props` has changed, a deep comparison occurs with the `nextProps`.

### The Parent Component Re-renders

A component will re-render whenever its parent re-renders.

### `this.forceUpdate` Is Called

Don't do this.
This basically flies in the face of React's design.


## What Does a Wasteful Re-render Look Like?

Imagine the following scenario:

``` javascript
class WastefulComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      seen: 'something',
    };
  }

  componentDidMount() {
    setInterval(() => {
      this.setState({ notSeen: 'something' }, () => {
        console.log('just finished updating the state');
      });
    }, 1000);
  }

  render() {
    console.log('render called');
    const { seen } = this.state;
    return (
      <div>
        {seen}
      </div>
    );
  }
}
```

This is very wasteful.
We end up re-rendering the component roughly once a second just because an unused portion of state has changed.

We can tell our component that this is wasteful by using `shouldComponentUpdate`.

## Detecting Unnecessary Re-renders

Re-renders are costly.
We want to remove as many `render` invocations as possible.

However, it can sometimes be hard to spot these inefficiencies.
Fortunately, React gives us `Perf` to find these damn things.

```javascript
import Perf from 'react-addons-perf';

Perf.start();
// Render something
Perf.stop();
Perf.printWasted();
```

## `shouldComponentUpdate`

By default, `shouldComponentUpdate` always returns `true`.
By overriding this behavior, we can let our component re-render intelligently.

For instance, we could add `shouldComponentUpdate` to our above example like so:

``` javascript
class WastefulComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      seen: 'something',
    };
  }

  shouldComponentUpdate(nextProps, nextState) {
    return this.state.seen !== nextState.seen;
  }

  componentDidMount() {
    setInterval(() => {
      this.setState({ notSeen: 'something' }, () => {
        console.log('just finished updating the state');
      });
    }, 1000);
  }

  render() {
    console.log('render called');
    const { seen } = this.state;
    return (
      <div>
        {seen}
      </div>
    );
  }
}
```

## Introducing Pure Component

The above `shouldComponentUpdate` is pretty simple.
However, it could become pretty unwieldy if our component relies on a lot of props or has a large state.
This is where `React.PureComponent` comes in.

`PureComponent` makes `shouldComponentUpdate` do a shallow comparison of the props and state.
The result is that re-renders only occur when properties of the props or state are reassigned to new objects.
This practice forces us to treat our state and props immutably.

For instance, the following `Numbers` component would not properly re-render:

```javascript
class Parent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      numbers: [1],
    };
  }

  handleClick() {
    const { numbers } = this.state;

    numbers.push(numbers.length + 1);
    this.setState({numbers});
  }

  render() {
    return (
      <div>
        <button onClick={() => this.handleClick()} />
        <Numbers numbers={this.state.numbers}
      </div>
    )
  }
}

class Numbers extends React.PureComponent {
  constructor(props) {
    super(props);
  }

  render() {
    const { numbers } = this.props;
    const numberString = numbers.join(', ');

    return (
      <div>
        {numberString}
      </div>
    );
  }
}
```

This doesn't work because `PureComponent` does a shallow comparison between `this.props` and `nextProps`.
When `this.props.numbers` is compared with `nextProps.numbers`, they are both the same since the they both point to the same array.

To fix this issue, we could treat the array as a persistent data structure by setting the state of the parent with return value of a concatenation of the previous state:

```javascript
class Parent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      numbers: [1],
    };
  }

  handleClick() {
    this.setState(prevState => ({
      numbers: [...prevState.numbers, prevState.length + 1],
    }));
  }

  render() {
    return (
      <div>
        <button onClick={() => this.handleClick()} />
        <Numbers numbers={this.state.numbers}
      </div>
    )
  }
}

class Numbers extends React.PureComponent {
  constructor(props) {
    super(props);
  }

  render() {
    const { numbers } = this.props;
    const numberString = numbers.join(', ');

    return (
      <div>
        {numberString}
      </div>
    );
  }
}
```

In this case, the updated state is a new array with the old elements and a new value.
So, the `Number` component will properly re-render.

## Immutable.js

Immutable.js is an option for handling props and state immutably.
