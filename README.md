# React Tips and Tricks

When working with React, it can be hard to find the **right** way to do things.
Here I've tried to compile some things I want to remember when working with a
React application.

## Table of Contents

1. [Working with Refs](#working-with-refs)
2. [Working With Children](#working-with-children)
3. [Higher Order Components](#higher-order-components)

## Working with Refs

For the most part, rendering components should be done explicitly through the
use of props. When a child needs to rerender itself its props should change
giving it the new information it needs. However, there are cases where you may
want access to direct DOM nodes or components. For this, `refs` are available
in React.

When using refs, there are two types that we can handle. The first is a simple
string which identifies a DOM node. The second is using a callback function.
While each is appropriate, one might be better than the other in different
cases.

In React, it's best to keep components simple and contained. In building a
sample form we may want a certain text input to be focused when the DOM loads.
We can do this through a ref.

```
class App extends React.Component {
  componentDidMount() {
    ReactDOM.findDOMNode(this.refs.textInput).focus();
  }

  render() {
    return (
      <div>
        <input
          type="text"
          ref="textInput" />
      </div>
    );
  }
}
```

In this basic example, we can see two things: the first is that we add a `ref`
prop to our text input. This tags this component and gives us a reference to
use in other places within this component. We can see this concept in the
`componentDidMount` method which calls `this.refs.textInput`. A React component
stores all of its refs here and allows us to access them in other methods.
In our case of wanting the input to be focused, we need access to the actual
DOM node and not just the component. To do this, we use `findDOMNode` from the
`ReactDOM` library. This function will look at the param and if it's a DOM node
it will just return that node. Otherwise it will get the DOM node from the
component for us.

With that, we can focus the element as expected.

Remember, in most cases we want to use props and state (where appropriate) to
handle interactions but in a case like this then it makes sense to work with
the DOM directly.

I mentioned above that we can also use a callback function to handle this. We
could replace the above code with a callback alternative but it may not be
needed and leads to more code ultimately. When a callback is especially
important though is when you want access to the refs in a child component or a
stateless functional component. For example, let's see how we would access a
DOM node of a child from a parent component:

```
const RefInput = (props) => <input type="text" ref={props.tellRef} />

class App extends React.Component {
  constructor() {
    super();

    this.focus = this.focus.bind(this);
  }

  focus() {
    this.textInput.focus();
  }

  render() {
    return (
      <div>
        <RefInput tellRef={(input) => { this.textInput = input; }} />

        <input
          type="button"
          value="Click to focus the first text input"
          onClick={this.focus} />
      </div>
    );
  }
}
```

Our child component in this example is a stateless functional component the
takes a prop called `tellRef`. This is not an official prop but is a good
identifier for this case. Our main component then passes the tellRef as a
callback function which assigns `textInput` to the parent class. This callback
takes the component as a param meaning we have a pointer to the text input.
These callbacks are called when the DOM renders meaning our `RefInput`
component will call the callback assigning its text input to the parent class.
Now we have access to `this.textInput` in the parent and can call focus on it.

This concept may not be used all that often but imagine a case where you are
rendering a row inside a table component and within that row you have button
components. When you click the row it has an onClick action that fires but you
don't want it to fire when one of the buttons is clicked. There are a few ways
to handle this but one way is to assign a ref callback from the row to the
button components. Then in the row's onClick method you can check if the event
target is the same DOM node as the button ref. If it is, you do not complete
the row's onClick event.

## Working With Children

Sometimes when working with child components we want to enhance them in a
specific way dynamically. In these cases, we will almost always work with
React's `React.Children` API. You can find the documentation [here](https://facebook.github.io/react/docs/react-api.html#react.children)
but two of the main methods you will run across are `React.Children.map()` and
`React.Children.only`.

As an example, let's say we have a bunch of buttons where we want the same
behavior when clicked. While this is a silly case, it illustrates the fact that
we want to dynamically add a click handler to each of those buttons. Of course,
we can modify the button HTML ourselves and add a click handler to each but
what if we don't want to modify those components?

Similiar to a decorator, we can wrap these buttons in a parent component which
acts on its children. This would look something like this:

```
class App extends React.Component {
  render() {
    return (
      <BtnsWithClickHandler>
        <button>One</button>
        <button>Two</button>
        <button>Three</button>
      </BtnsWithClickHandler>
    );
  }
}
```

Given that React is a functional library, this would be roughly equivilent to
`BtnsWithClickHandler([Button('One'), Button('Two'), Button('Three')])` or just
a function that takes an array. In that vein, we could simply loop over this
array, modify the component, and return it in a mapping function but React helps
us out with the opaque data structure of child components. This is where
`React.Children.map` comes into play. Our parent wrapping component would look
something like this:

```
const BtnsWithClickHandler = (props) => (
  <div className="clickable-btns">
    {
      React.Children.map(props.children, (child, i) => React.cloneElement(child, {
        ...child.props,
        onClick: (e) => alert(e.target.textContent)
      }))
    }
  </div>
);
```

The first thing to note about this component is that it is a functional
stateless component meaning it's simply a function with no state. In these
situations where you just want to enhance components, this is a good approach
since it keeps your code simple and doesn't need to do anything complex. Our
component will wrap the children in a new parent div and then loop over each
child with map function. Like a normal array, map applies a function to each
child element. In order to preserve the element's structure we use another
handy React method called `cloneElement`. This will copy the child element and
return a new one based on the params passed in.

In this case, we apply the existing props the child has and add a new one for
`onClick`. In this example, we are showing a javascript alert with the
textContent of the DOM node when the user clicks the button.

Now when you load a browser you will see each button rendered and clicking on
them will show an alert.

Sometimes it doesn't make sense enhance a group of children though. In fact, a
lot of times we only want to enhance a single child element. For example, let's
say that we want to have a parent disable the child. If we are sure that we
want only a single child element, this is where we can use `React.Children.only`
which will ensure that we are in fact using a single child. An example of this
component would be:

```
const BtnDisabled = (props) => {
  const child = React.Children.only(props.children);

  return (
    <div>
      {
        React.cloneElement(child, {
          ...child.props,
          disabled: true
        })
      }
    </div>
  );
};
```

We first get the child element which will throw an error if we have more than a
single child. Then we use `cloneElement` on this single child and add the
disabled attribute to the child. We can use this component with the above code:

```
class App extends React.Component {
  render() {
    return (
      <BtnsWithClickHandler>
        <BtnDisabled>
          <button>One</button>
        </BtnDisabled>
        <button>Two</button>
        <button>Three</button>
      </BtnsWithClickHandler>
    );
  }
}
```

Now when we render this component in the browser all children will have a click
handler but the first child in the list will be disabled.

When working in React, keeping our components as simple as possible is one of
the real keys to success. Using parent components where their only job is to
enhance their children goes along with this principle. It allows us to write
generic components and add any funcionality we may need.

## Higher Order Components

Higher order components are one of the more advanced React ideas which can be
summed up as functions which act as component factories. In React we strive to
create reusable components. Overall this is a great idea because we keep our
code DRY but in cases where our generic components are not enough we need a way
to enhance them. Higher order components give us this ability. For example, say
we have multiple modals in our application. We define a generic modal component
which renders the header and body based on props as well as any button
behaviors. In some cases though we want the close button to simply close the
modal but in other cases we want to alert the user if there are unsaved changes
to a form. We could define the onClick handler for the close button on each
instance but we would be repeating ourselves.

Instead, we can use a higher order component to assign the two types of
behaviors we want encapsulating this behavior in a single place. We could do
something like this:

```
class Modal extends React.Component {
  render() {
    return (
      <div className="my-modal">
        <div onClick={this.props.onCloseModal} className="close-btn">
          X
        </div>
        <div className="header">
          {this.props.header}
        </div>
        <div className="body">
          {this.props.body}
        </div>
      </div>
    );
  }
}

withDefaultCloseBehavior = (WrappedComponent) => {
  return class Component extends React.Component {
    render() {
      const newProps = {
        onClick: () => // ..close the modal
      };

      return <WrappedComponent {...this.props} {...newProps} />
    } 
  }
}

withConfirmCloseBehavior = (WrappedComponent) => {
  return class Component extends React.Component {
    render() {
      const newProps = {
        onClick: () => // ..show confirm dialog before close
      };

      return <WrappedComponent {...this.props} {...newProps} />
    } 
  }
}
```

With this example we can now create an enhanced component with
`withDefaultCloseBehavior(Modal)`. This enhanced component can then be used in
our application and can take the normal props that a Modal component can but it
will by default have an `onCloseModal` prop passed to it for us.

In this example, we are using a concept called **Props Proxy**. This is one of
the two main ways we work with higher order components. A props proxy
manipulates a component's props and gives us a new component to use.

In the above example we add props to a component. We can also abstract state
away from components. The following demonstrates abstracting state of a
controlled input away from the input itself.

```
function controlledInput(WrappedComponent) {
  return Component extends React.Component {
    constructor() {
      super(props);
      this.state = {
        name: ''
      };

      this.onNameChange = this.onNameChange.bind(this);
    }

    onNameChange(event) {
      this.setState({
        name: event.target.value
      });
    }

    render() {
      const newProps = {
        name: {
          value: this.state.name,
          onChange: this.onNameChange
        }
      };

      return <WrappedComponent {...this.props} {...newProps} />
    }
  }
}

class Input extends React.Component {
  render() {
    return <Input name="name" {this.props.name} />
  }
}

const MyInput = controlledInput(Input);
```

Here we see that we can automagically created a controller input by applying
our higher order component to our dumb component. Now we can choose to control
the `Input` component on our own or get it for "free" by wrapping it.

The other way to create higher order components is through **Inheritance
Inversion**. The basic structure is:

```
function iiHOC(WrappedComponent) {
  return class Enhancer extends WrappedComponent {
    render() {
      return super.render()
    }
  }
}
```

This seemed a little weird to me at first but is actually pretty cool. This
gives us complete access to the WrappedComponent instance meaning the props,
state, lifecycle, and render method. One thing this allows is something called
**render highjacking**. The best example of this is conditional rendering based
on a certain prop. For example, showing content to logged in users and other
content to anonumous users.

```
function iiHOC(WrappedComponent) {
  return class Enhancer extends WrappedComponent {
    render() {
      if (this.props.loggedIn) {
        return super.render()
      } else {
        return null
      }
    }
  }
}
```

Since we have access to the render method of the wrapped instance we can call
it to output the component to the DOM. Otherwise we can choose to render
nothing.

In general, higher order components allow us to enhance components in a sane
way.
