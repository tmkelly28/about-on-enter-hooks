# about-on-enter-hooks
Everything you wanted to know about on enter hooks but were afraid to ask

# What are onEnter hooks
The `onEnter` hook is a lifecycle hook specific to `react-router` - you can only attach `onEnter` hooks
to `Route` components, like so:

```
<Route path="/" component={MyComponent} onEnter={myOnEnterHook} />
```

What you attach to the `onEnter` prop is a function.
That function can expect to receive the following arguments:

```
const myOnEnterHook = function (nextRouterState, replace, done) {
  // your slick code here
}
```

Let's break each of these down:

# Arguments of an onEnter hook

## `nextRouterState` 
`nextRouterState` is an object that contains the upcoming internal state of the `Router` component.
It usually contains key-value pairs like 'location', 'params', and 'routes' - the main things that `react-router` handles for you. If you need to access the routeParams of a Route (ex. `/stories/:storyId`), this is the place to access that! (In this example, you would be able to get the storyId by saying `nextRouterState.params.storyId`).

## `replace` 
`replace` is a function that allows you to dynamically change the url. It is similar to `hashHistory.push`. It can either take a pathname, or an object with more configuration. You might use this in an onEnter hook to dynamically re-direct a user from one Route to another. For example, say you have a `Route` like this:

```
<Route path="/adminsOnly" component={AdminStuff} />
```

If would be a shame if commoners were able to see that component. We might implement some access control like so:

```

const adminsOnlyHook = function (nextRouterState, replace) {
  const currentUser = store.getState().currentUser;
  if (!currentUser.isAdmin) {
    replace('/normalFolks')
  }
}

<Route path="/adminsOnly" component={AdminStuff} onEnter={adminsOnlyHook} />
```

`replace` is **different** from `hashHistory.push` in that it does not immediately change the URL. When you use `hashHistory.push`, that immediately causes the url transition. The transition that you create with `replace` must wait until the hook finishes running!

## `done`
`done` is the trickiest argument. It's a special function, which may be used to signal to `react-router` when to actually transition to rendering the component.

The first thing to note is that it **matters** whether or not `done` is included in your list of arguments. That means there is a difference between this:

```
const adminsOnlyHook = function (nextRouterState, replace) {
  const currentUser = store.getState().currentUser;
  if (!currentUser.isAdmin) {
    replace('/normalFolks');
  }
}
```

and this:

```
// note that we've included done as an argument
const adminsOnlyHook = function (nextRouterState, replace, done) {
  const currentUser = store.getState().currentUser;
  if (!currentUser.isAdmin) {
    replace('/normalFolks');
  }
}
```

For the first example (without including 'done'), the `onEnter` hook will run to completion and will transition to rendering its component as soon as it completes.

This is **not** the case for the function that contains 'done' as an argument. Because the second function uses 'done', `react-router` will **block** rendering **until** done is invoked. That means that right now, our 'adminsOnlyHook' with done will NEVER actually transition to showing the component! That's bad!

To fix this, we need to invoke 'done' to let `react-router` know it's okay to transition.

```
const adminsOnlyHook = function (nextRouterState, replace, done) {
  const currentUser = store.getState().currentUser;
  if (!currentUser.isAdmin) {
    // remember - replace does NOT stop the thread - we keep going after this
    replace('/normalFolks');
  }
  // this will be invoked regardless of whether the user is an admin or not
  done();
}
```

Using the `done` callback is useful if you want to totally prevent transition to a certain state until you have done something (possible asynchronous). For example, if we needed to fetch the user asynchronously in our 'adminsOnlyHook', we may want to use the 'done' callback to make sure that non-admins don't get a 'peek' of the admin content.

# How do I access state in an onEnter hook

`onEnter` hooks are not passed any props from your own state (be that state that's held locally by React compoents, or your Redux store).
Because of this, it's perfectly okay to `import store from './yourStore` and access state directly by saying `store.getState`, and
dispatching action using `store.dispatch`.

# Difference between onEnter and componentDidMount

`onEnter` and `componentDidMount` are often used for the same purpose - fetching data from the server when the component is loading. They can often be used interchangeably. There is one big important difference though.

Remember in `Juke` that when we created our list of playlists, switching between playlists didn't cause the view to update because the `Playlist` component had already mounted, so the `componentDidMount` hook did not run again. We needed to use `componentWillReceiveProps` to solve this.

`onEnter` hooks do not have this problem - they run every time the `Route` is entered, regardless of whether the component it renders is mounted or not.

For this reason, you should generally favor `onEnter` hooks over `componentDidMount` for doing things like fetching data from the server that will be displayed in the component.

# Where do I define my onEnter hooks
This is up to you. If you have many onEnter hooks, you may write them in a `hooks.js` file and import them into the file(s) containing your Routes. If you don't have many, you may write them in the same file as your `Routes`.

