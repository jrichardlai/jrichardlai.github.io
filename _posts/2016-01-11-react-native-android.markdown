---
title: React Native Android
description: It works!
---

It's been 4 weeks since the launch of the iOS Tasker App and everything seems to be good so far, no apparent bug came through and our Tasker base didn't see that much difference, except for the better font scaling (I admit that some people got confused here 🙃) and more responsive app (#win).

You can read more about the launch with [Brian Leonard](https://twitter.com/bleonard) on the [TaskRabbit Tech Blog](http://tech.taskrabbit.com/blog/2015/12/17/react-native-launch/).

Now is time for the Android implementation.

## Android here we come!

### Preface

I haven't touched an Android phone for years (*little exageration*) until the past 2 weeks, so I am not fully aware of the best practices / UI paradigm of the Android World. Therefore we decided as a first step, acknowledging that some Android users might be displeased of native App feeling like an iOS App, to go for feature parity with the iOS App. We would then as 2nd step implement the parts that are breaking Android conventions.

### Running the App

The React Native team did an awesome job with the implementation of Android. It was pretty easy to launch the App in the simulator after few tweaks. The structure we had for the project were already thought for the upcoming Android support. All Components and Classes that required a library specific to iOS or iOS Platform React Native Component/Apis (with `IOS` suffix) were put inside a `Platform` folder.

For example there was a `Platform/Alert.js` with content:

```javascript
import React from 'react-native';

const {
  AlertIOS
} = React;

const Alert = {
  alert(title, message, buttons, type) {
    AlertIOS.alert(title, message, buttons, type);
  },
  prompt(title, value, buttons, callback) {
    AlertIOS.prompt(title, value, buttons, callback);
  }
}

export default Alert;
```

You can also just do `export default AlertIOS` without defining `Alert`, but having `alert` and `prompt` declared will show which API is used by the App and what is the method signature.

So I just needed to add `.ios` to all the files in the `Platform` folder then copy/paste them with `.android`.

Here is `Alert.android.js`:

```javascript
const Alert = {
  alert(title, message, buttons, type) {
    console.warn('Alert.alert not implemented');
  },
  prompt(title, value, buttons, callback) {
    console.warn('Alert.prompt not Implemented');
  }
}

export default Alert;
```

I repeated the same stubbing for the other files.

For AlertIOS, the newer version of React Native 0.17.0 (0.18.0 in RC) got released with the awaited implementation of an Alert compatible for both platforms. In the case where you are not using `prompt` (`prompt` is not available for Android).
It would be then easy as finding and replacing `import Alert from '../Platform/Alert'` by `const { Alert } = React`, no code change inside the Components will be required.

## Some gotchas found so far with Android

- When inheriting from a React Native Component, it throws an [error](https://github.com/taskrabbit/react-native-parsed-text/issues/4), bummer I liked writing `class Input extends React.Input`.
- Only `keyboardDidShow` and `keyboardDidHide` are emitted.
- The Keyboard does not act the same way than iOS, the trick to change the Layout using a Component that changes height depending of the height of the Keyboard does not seems to work.
- There seems to be disparities with the usage of `textAlign` for `TextInput`, iOS requires `auto`, `center`, `justify`, `left`, `right` and Android supports `start`, `center`, `end`.
- Our Available Tasks sliding does not currently work in Android.
- It does not seems to be a clear way how to implement push notifications.
- Be aware that some React Native Component features are not yet available for Android (MapView, Modal), `Alert.prompt`, `setApplicationIconBadgeNumber`.

## Android current progress:

Here is the list of our platform files implementation as today:

- ✅ The platform implementation is done
- 🔶 Have a Workaround
- ⛔ Not implemented / Work in progress

| Status | Class | Purpose |  Notes |
|----------|:----------|:-------------:|------:|
| ⛔| Actionsheet |  Display a list of actions | |
| ✅| ActivityIndicator | Loading indicator | Uses `ProgressBarAndroid` |
| ⛔| Actionsheet | Display a list of actions | |
| 🔶| Alert | Display an alert to the user | Uses `Alert` but missing `prompt` |
| ⛔| AppState | Triggers event when the app becomes visible / hidden |
| ⛔| DatePicker | Picking a date and time | |
| ✅| EnvironmentManager | Provides to the App the phone uuid, environment name (test, dev, staging, production), buildCode, version, i18n locale | |
| ⛔| ExpandingTextInput | Text that auto expand on input | Using implementation for iOS based on this [PR](https://github.com/facebook/react-native/pull/1229), Brian made a [gist](https://gist.github.com/bleonard/6770fbfe0394a34c864b) for it
| ⛔| FacebookLogin | Log in with Facebook | |
| ⛔| ImagePickerManager | Picking an image from the library or taking a photo | iOS is using [react-native-fs](https://github.com/johanneslumpe/react-native-fs) and [react-native-image-picker](https://github.com/marcshilling/react-native-image-picker)
| ✅| Linking | Open url or app | Uses `LikingIOS` and `IntentAndroid` (`openURL` and `canOpenURL`)
| 🔶| Modal | Open a Modal | Uses `ModalIOS` and for Android we are using a [gist](https://gist.github.com/mkonicek/1a8bd7253e3257687228) from [Martin Konicek](https://twitter.com/martinkonicek) using Portal ( be sure to wrap your Component with a Context Wrapper if needed )
| ⛔| MultiPicker | Multi picker
| ⛔| Picker | Picker | Uses for iOS PickerIOS
| ⛔| ProgressCircle | Display a progress circle | Will be ✅ with React Native 0.18 with ART support
| ⛔| Tooltip | Show a Tooltip on long press | Uses our fork of [react-native-tooltip](https://github.com/taskrabbit/react-native-tooltip)
| ⛔| ZendeskChat | Allow ZendeskChat in App | Uses a Native Module providing API to [Zopim Mobile SDK](https://github.com/zendesk/zendesk_sdk_chat_ios)

Thanks for reading, feel free to comment and provide solutions if you have some!
