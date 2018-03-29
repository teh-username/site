---
title: Integrating Redux with React Native
date: 2018-03-29 17:12:01
tags:
  - react-native
  - react-navigation
  - redux
  - pin-vault
---

TL;DR: Use `screens` as `containers`. See sample snippet below.

When I started drafting out [pin-vault](https://github.com/teh-username/pin-vault), I knew from the start that I was going to use redux for all that state management goodness. The only blocker then was how do I emulate the `container` and `presentational` setup that I'm used to with normal react + redux.

I was able to distill the situation with the following action items:

1. How to do routing in React Native?
2. Where do I inject the Redux stuff?

The first problem is easy, [react-navigation](https://reactnavigation.org/) seems to be the community's choice for react native routing. As for the second problem, that's where I kinda got stuck.

There is some [documentation about integrating react-navigation with redux](https://reactnavigation.org/docs/redux-integration.html) but it's for the use case when you want to manage the navigation state by yourself, something I'd like react-navigation to handle for me. What I was looking for was the 'container component' equivalent in react-native.

## Screens as `containers`

react-navigation introduces the notion of a 'screen' which is essentially the component that gets 'imported' and rendered when you navigate to a particular route.

Screens are basically the root for all your presentational components so it is only logical that this is _THE_ component that should be connected to your redux store.

Here's a sample snippet demonstrating the integration:

```javascript
// screens/SetPasscode.js

import React from 'react';
import { connect } from 'react-redux';

import PasscodeForm from '../components/PasscodeForm';
import { setPasscode, getCurrentPasscode } from '../redux/modules/settings';
import { hashString } from '../utils/crypto';

export class SetPasscode extends React.Component {
  static navigationOptions = {
    title: 'Set Passcode',
  };

  render() {
    return <PasscodeForm {...this.props} />;
  }
}

export const mapDispatchToProps = (dispatch, ownProps) => ({
  handlePasscodeSubmit(passcode) {
    dispatch(setPasscode(hashString(passcode)));
    ownProps.navigation.goBack();
  },
});

const mapStateToProps = state => ({
  oldPasscode: getCurrentPasscode(state),
});

export default connect(mapStateToProps, mapDispatchToProps)(SetPasscode);
```

It's as easy as that! No need to include the boilerplate outlined in the documentation linked above. Happy coding!
