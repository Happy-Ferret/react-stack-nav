React stack nav
==========
Simple universal navigation for React Native _and_ React

- **Universal** Works in iOS, Android, and Web
- **Simple, familiar API** Has the same API as the web's [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API)
- **Composable and declarative** Uses React's component tree to compose and handle routes
- **Supports back/forward buttons** Automatic support for Android back button and back/forward buttons on web
- **Server-side rendering** Simple as pre-loading redux state with the requested url
- **Easy to understand** You can read the source! Only ~200 lines of code

## What it looks like

A navigation component (drawer, tabs, anything):
```js
import React from 'react'
import { Button, View } from 'react-native'
import { connect } from 'react-redux'
import { pushState } from 'react-stack-nav'

const Navigation = ({ pushState }) =>
  <View>
    <Button onPress={() => pushState(0, 0, '')}>Go to home</Button>
    <Button onPress={() => pushState(0, 0, '/myRoute')}>Go to My Route</Button>
  </View>

const mapDispatchToProps = { pushState }

export default connect(null, mapDispatchToProps)(Navigation)
```

A component receives and handles the first fragment of the url:
```js
import React from 'react'
import { Text, View } from 'react-native'
import { createOrchestrator } from 'react-stack-nav'

const Index = ({ routeFragment }) =>
  <View>
    {routeFragment === '' && <Text>Home</Text>}
    {routeFragment === 'myRoute' && <Text>My route</Text>}
  </View>

export default createOrchestrator(Index)
```

## Installation

```
npm -S i tuckerconnelly/react-stack-nav
```

Then match your redux setup to this:

```js
import { createStore, applyMiddleware, compose, combineReducers } from 'redux'
import thunk from 'redux-thunk'
import { navigation, attachHistoryModifiers } from 'react-stack-nav'
import { BackAndroid } from 'react-native'

import app from './modules/duck'

const rootReducer = combineReducers({ navigation })

export default (initialState = {}) =>
  createStore(
    rootReducer,
    initialState,
    compose(
      applyMiddleware(thunk),
      attachHistoryModifiers({ BackAndroid }),
    ),
  )
```

## Usage

react-stack-nav follows a few principles that make it simple and portable
- Treat the redux store as a single-source-of-truth for routing
- Use URLs and the History API, even in React Native
- Treat the URL as a stack, and "pop" off fragments of the URL to orchestrate transitions between URLs
- Leave you in control, favor patterns over frameworks

---

At the core of react-stack-nav is the `navigation` reducer, whose state looks like this:

```js
{
  index: 0, // The index of the current history object
  history: [{ statObj, title, url }], // An ordered list of history objects from pushState
}
```

If you want to navigate to a new url/page, use the `pushState` action creator in one of your components (it's the same as [history.pushState](https://developer.mozilla.org/en-US/docs/Web/API/History_API)!):

```js
import { pushState } from 'react-stack-nav'

class MyComponent {
  _handleClick = () => this.props.pushState({ yo: 'dawg' }, 'New Route', '/newRoute')
}
```

That'll push a new object onto `state.navigation.history`, so your new navigation reducer state might look like this:

```js
{
  index: 1,
  history: [
    { stateObj: {}, title: '', url: '' },
    { stateObj: {}, title: 'New Route', '/newRoute' }
  ]
}
```

Now say you clicked back in the browser (or hit the android back button), your state would automatically update to:

```js
{
  index: 0,
  history: [/* same as above */]
}
```

With `index: 0` to say, hey, we're on the first history entry.

---

If you want to use the history object, to ya know, render stuff, you can do so directly:

```js

const MyComponent = ({ url, title }) =>
  <View>
    <Text>{title}</Text>
    {url === '/users' && <UsersPage />}
    {url === '/pictures' && <PicturesPage />}
  </View>

const mapStateToProps = ({ navigation }) => ({
  url: navigation.history[navigation.index].url,
  title: navigation.history[navigation.index].url,
})

connect(mapStateToProps)(MyComponent)
```

You could handle all your routing like that, but...wait a minute...if you turn a url sideways, it kinda looks like a stack yeah?

```
/users/1/profile

 users
-------
   1
-------
profile
```

`react-stack-nav` runs with this idea and introduces the idea of an _orchestrator_. Orchestrators pops off an element in the stack and declaratively handles transitions when it changes:

```
 users     -------> Popped off and handled by first orchestrator in component tree
-------
   1       -------> Popped off and handled by second orchestrator in component tree
-------
profile    -------> Popped off and handled by third orchestrator in component tree
```

The first orchestrator found in the component tree would handle changes to the top of the stack:


```js
import { createOrchestrator } from 'react-stack-nav'

const Index = ({ routeFragment }) => {
  <View>
    {routeFragment === 'users' && <UsersIndex />}
    {routeFragment === 'pictures' && <PicturesIndex />}
  </View>
}

export default createOrchestrator()(Index)
```

The next orchestrators found in the tree would handle the next layer of the url stack, so `UsersIndex` might look like this:

```js
class UsersIndex extends Component {
  async componentWillReceiveProps(next) {
    const { routeFragment } = this.props
    if (routeFragment !== next.routeFragment) return

    this.setState({ user: await fetchUserData(routeFragment) })
  }

  render() {
    <View>
     {/* Use this.state.user info */}
    </View>
  }
}

export default createOrchestrator('users')(UsersIndex)
```

`createOrchestrator()` accepts a string or a regular expression for that particular layer of that stack. If it doesn't match the current layer of the url, `props.routeFragment` will be undefined.

You could create orchestrators _ad inifitum_ all the way down the component tree to handle as much of the URL stack as you want:

```js
const UserIndex = ({ routeFragment }) => {
  <View>
    {routeFragment === 'profile' && <ProfileIndex />}
    {routeFragment === 'settings' && <SettingsIndex />}
  </View>
}

export default createOrchestrator(/\d+/)(UserIndex)
```

```js
const ProfileIndex = ({ routeFragment }) => {
  <View>
    {/* Would handle /users/${userId}/profile/posts */}
    {routeFragment === 'posts' && <UserPosts />}
    {/* Would handle /users/${userId}/profile/pictures */}
    {routeFragment === 'pictures' && <UserPictures />}
  </View>
}

export default createOrchestrator('profile')(ProfileIndex)

```

## API

- `attachHistoryModifiers` - Store enhancer for redux, handles android back-button presses, and browser back/forward buttons
- `createOrchestrator(fragment: string | regexp)` -- Higher-order-component that creates an orchestrator
- `navigation` -- `navigation` reducer for setting up your store

#### Action creators

These are the same as the [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API) :

- `pushState(stateObj: object, title: string, url: string)`: Pushes a new state onto the history array
- `replaceState(stateObj: object, title: string, url: string)`: Replaces the current state on the history array
- `back(reduxOnly: bool)`: Moves the `navigation.index` back one. If `reduxOnly` is true, it won't change the browser `history` state (this is mostly for internal use)
- `forward(reduxOnly: bool)`: Moves the `navigation.index` forward one, reduxOnly is the same as for `back()`
- `go(numberOfEntries: int)`: Moves the `navigation.index` forward/backward by the passed number (`go(1)` is the same as `forward()`, for example).

## License
MIT
