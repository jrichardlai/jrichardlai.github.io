---
title: React Component Extension
description: A Higher Order Component implementation
---

I have been working for 4 months with React on the Native Mobile Tasker app at [TaskRabbit](https://www.taskrabbit.com) and I really enjoy it.

I love the way React allows you to build apps by thinking about all the visual elements that compose the screens. It is recommended to have small and simple elements, meaning that they should not do too many different things.

In React, those parts are called components. For example, you might have `Button`, `Text`, `ErrorBar`, `Slider`, and `Loader` components.

As the app grew, we needed to share behavior between components. For example, several screens had a spinning loader or a disabled form input.

ReactJS provides a way to do so by including [mixins](https://facebook.github.io/react/docs/reusable-components.html#mixins):

```javascript
const CurrentUser = {
  isLoggedIn() {
    return !!this.state.currentUser;
  },
}

const Loader = {
  start() {
    this.setState({load: true});
  },
  stop() {    
    this.setState({load: false});
  },
}


const TopBar = React.createClass({
  mixins: [CurrentUser, Loader],

  logIn() {
    console.log(this.state.load) // => false
    this.start();
    console.log(this.state.load) // => true
    $.getUser('api/user', (currentUser) => {
      this.stop();
      this.setState({currentUser});
    });
  },

  render() {
    if (this.isLoggedIn()) {
      return <div>Hello!</div>;
    } else {
      return <div><button onClick={this.logIn}>Please Log in<button></div>;
    }
  },
});
```

This solution was fine at first, but after rereading the code, one issue I faced is that the behavior was obstructed. I didn't know where `start` or `isLoggedIn` methods were coming from. Also, if a mixin defines the same method as another one that is included there would be a method collision.

I was trying to figure out a way to resolve this issue and read that actually [mixins were dead](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750#.jqp1e0of3).

So I gave it a try. It looked like this at first:

```javascript
const CurrentUser = (Component) => {
  return React.createClass({
    // ... previous code
    render() {
      return <Component isLoggedIn={this.isLoggedIn} />;
    },
  });
}

const Loader = (Component) => {
  return React.createClass({
    // ... previous code
    render() {
      return <Component start={this.start} stop={this.stop} load={} />;
    };
  });
}

var TopBar = React.createClass({
  logIn() {
    console.log(this.props.load) // => false
    this.props.start();
    console.log(this.props.load) // => true
    $.getUser('api/user', (currentUser) => {
      this.props.stop();
      this.setState({currentUser});
    });
  }

  render() {
    if (this.props.isLoggedIn()) {
      return <div>Hello!</div>;
    } else {
      return <div><button onClick={this.logIn}>Please Log in<button></div>;
    }
  }
});

TopBar = CurrentUser(TopBar);
TopBar = Loader(TopBar);
```

It worked well enough, but I felt that when reading the code, I still was not sure where the behavior came from. I wanted to pass options to the higher order component. So to make it explicit, I decided to add namespacing to the props that are passed down.

It looked like this:

```javascript
var TopBar = React.createClass({
  logIn() {
    console.log(this.props['Loader'].load) // => false
    this.props.start();
    console.log(this.props['Loader'].load) // => true
    $.getUser('api/user', (currentUser) => {
      this.props['Loader'].stop();
      this.setState({currentUser});
    });
  }

  render() {
    if (this.props['CurrentUser'].isLoggedIn()) {
      return <div>Hello!</div>;
    } else {
      return <div><button onClick={this.logIn}>Please Log in<button></div>;
    }
  }
});
```

In the goal of making a higher order component more explicit, we decided to make a library to define React Component Extensions. It allows you to define:

* Methods passed down
* Variables from the state of the Extension that can be used
* Params required/allowed to configure the Extension

You can check it out on [GitHub](https://github.com/taskrabbit/react-component-extension).
