---
title: "Reflecting on React Native Android"
description: 87% shared code
---

<style>
  .entry-content table th, .entry-content table td {
    border: 1px solid #ccc;
    background-color: #f5f5f5;
    padding: 3px;
  }
</style>

Before starting this article, I want to give a big thanks to [Jeremy](http://jeremyeaton.co/) for his hard work on implementing the features for Android and to [BL](https://twitter.com/bleonard) for being such a lenient PM 😊 and for proofreading the blogpost.

Last week we launched [at TaskRabbit](http://tech.taskrabbit.com/blog/2016/03/24/react-native-android-launch/) the Android Tasker app written in React Native. The app had been in a Beta for about 2 weeks and few bugs came up (mostly related to push notifications). I'm going to write here about the experience we had throughout the development of the app and provide some insights about the challenges we faced in comparison to iOS.

**Disclaimer:** I am not an Android developer, my only (and brief) experience with Android was in 2009 with Android 🍩 and I remember at this time I had a lot of issues setting up Android on Windows.

Consider the following notes to be from someone new and with no background in Android. Please let me know if you see anything incorrect.

### Compiling a React Native Android app

Settings up the environment for Android requires more effort than for iOS (as XCode mostly works out of the box). Thankfully React Native comes with good documentation on [how to set up your environment](https://facebook.github.io/react-native/docs/android-setup.html). Be sure to select all the required plugins for Android.

#### Gradle

Gradle is the official build system used for Android. It is highly customizable and takes cares of installing the dependencies of the project. I find building the app through the command line more reliable than using an IDE like InteliJ.
When using InteliJ, gradle will sometimes fail when building the project. I would get a `Gradle project sync failed` error and trying to clear the cache would not fix the issue.

#### Dependencies version conflict

Another issue I found with gradle is that different libraries could use different versions of dependencies as the application. At some point one of the library was using `com.google.android.gms:play-services-base:8.3.1` but our app was on `com.google.android.gms:play-services-base:8.3.0` and this made the app crash on launch. The problem was not obvious as there was no clear message about what was causing the crash. Upgrading to the same version of play-services the issue was resolved. Another issue happened with React Native 0.21 ( which is now fixed in 0.22) with `support-v4` and we had to force the version by adding it into `build.gradle`:

```
subprojects {
    configurations.all {
        resolutionStrategy {
            // https://github.com/facebook/react-native/issues/6152
            force 'com.android.support:support-v4:23.0.1'
        }
    }
}
```

#### Build environments

To provide flexibility in development, we wanted to set up multiple build environments. This was done for our iOS app using Schemes and wanted to set it up as well for Android by using build variants.

Here is the list of variants:

- `localtest`: Local test environment
- `remotetest`: Remote test environment on [Travis](https://travis-ci.com) and [Saucelabs](https://saucelabs.com/)
- `debug`: API requests to a local server
- `stagingdebug`: API requests to a staging server with debug
- `staging`: Staging release version
- `production`: Production release version

When React Native Android first shipped, there were only the `debug` and `release` default environments. I made a copy of `react.gradle` and tweaked it to allow configuring multiple build variants. With the new versions of React Native, you don't have to do this any more because React Native allows you to configure variants and pass it to `react.gradle`. It seems though that sometimes the bundled JavaScript file was not regenerated and clearing the cache was required,so I added a dependency to our release build:

```
task clearReactCache(type: Exec) {
    commandLine "rm", "-rf", '$TMPDIR/react-*'
}
```

### Testing

#### Integration testing with Appium

As noted [previously on the TaskRabbit blog](http://tech.taskrabbit.com/blog/2015/11/08/react-native-integration-tests/), the Tasker App is tested with unit and integration tests using [Appium](http://appium.io/). React Native is a really fast-moving framework and a strong test suite is required to make sure that we are not breaking the app when upgrading the version of React Native.

As our iOS and Android app shares pretty much the same screens and interactions, we can use the same test suite for both iOS and Android for most of it. When there is different behaviors (for example when selecting the `DatePicker` and `Picker`) we can use `isIOS` and `isAndroid` variables to set the proper test.

#### Travis

Our Android and iOS test suite can be ran locally, but as the whole suite takes a long time (~30 minutes 😞). We decided to set up a continuous integration using [Travis](https://travis-ci.com); however the test suite then ran in 90 minutes, so we set up parallelization for the suite by using the Appium driver on [Saucelabs](https://saucelabs.com/) which we also used to see a video of the running test suite.

#### Saucelabs

Having the test suite set up to work remotely on Sauce took me more time that I wanted. There was a lot of back and forth to set up properly the tests on Sauce. Also, running the test suite locally to Sauce does not be have the same way when its running on Travis to Sauce because the requests and commands can take longer so some tests would only fail with Travis.

To reduce issues, we are bundling the JavaScript into the uploaded app on Sauce. Right now only the iOS app is running on Travis. The reason is that for Android I didn't find a way to bundle the JavaScript while still being in dev environment, It seems that you need a packager running on port `8081` even if the bundle is present in the assets.

Also, when working on the tests locally it seemed that Appium was failing more intermittently for Android than iOS. One way we fixed it was to wrap the commands with a retry and sleep the command:

```
driver.retryWithSleep = o_O(function *(action, time) {
  try {
    return yield action.call();
  } catch (e) {
    console.log(`Retry with sleep ${time}`);
    yield driver.sleep(time);
    return yield action.call();
  }
});

driver.retryWithSleep(function *() {
  yield driver.elementById('Yes');
}, 5000);
```

#### Testing on real phones

I recommend testing on as much devices as you can because different versions of Android could behave differently. For example, the back button, scrolling and tapping can feel different. You sometimes have to make little tweaks to accommodate different phones.

Also some libraries can have issues with different devices. When testing the app, a few phones some were running properly and others were crashing. This was due to a [Realm](https://realm.io/) issue. Thanks to the team there, the issue was fixed fast.

### Screen Sizes and layout

One of the issues that we didn't faced much on iOS were size and different aspect ratios. It is challenging to write code that will fit all screen sizes.
Thankfully React Native allows using [flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/) layouts which simplifies organizing elements inside containers.

I think one of the best practices is that you should *never use a hardcoded value* to set the width or height of a Component. You should try to use margin and padding to position the element properly or in the case you need a numerical value you should calculate it by using the height or width of the screen.

You can get the values by using `Dimensions`:

```
const { width, height } = Dimensions.get('window');
```

One thing I noticed is that on Android the height returned can sometimes include the virtual bar that some phones have. To fix it you can use [react-native-extra-dimensions-android](https://github.com/jaysoo/react-native-extra-dimensions-android) (not sure if this is still an issue).

In the same way you want to make sure that the text you put in a component fits properly. We used a [`AutoScaleText` component](https://gist.github.com/jrichardlai/de17535abc02a7472d7f4c17911087fd) that sets the proper font size to fit an element. This was really useful for our navigation bar where the text can get long.

### Mind the back button

One core feature of Android that is not present with iOS is the **Back Button**, it allows to go back in the navigation but also to dismiss keyboards, dialogs and other UI elements.

Android will take care of the keyboard and native UI elements, however it is the responsibility of the developer to handle React Native JavaScript Components behavior as well as the resulting navigation.

React Native provides a way to handle the back button by adding a listener to `hardwareBackPress` with `BackAndroid.addEventListener`.

I am not sure what is the best way to handle it. Unfortunately, I did not find a lot of documentation about it.

At first we tried to add listeners inside Components, subscribing to `hardwareBackPress` in `componentWillMount` and unsubscribing in `componentWillMount` but all of them would be called when the back button is pressed, which felt a little strange, because when pressing the back button you would expect that it would act only on the last opened element.

I think that it would be more correct if it was acting like a stack:

- Initial State => []
- Open Modal => [modal]
- Open Accordion => [modal, accordion]
- Back button pressed, close Accordion => [modal]
- Back button pressed, close Modal => []

In the end we made one file that contains one `hardwareBackPress` listener that manages the back press. We only had 2 components that needed to respond to the back button, `Modal` and `Drawer`, as those are singletons in the app, we choose to use class variables to manage their visibility, the simplified code looks like this:

```
let drawer = null;

class Drawer extends React.Component {
  componentDidMount() {
    drawer = this;
  }

  close() {
    setState({visible: false});
  }

  // more code
}

Drawer.close = () => {
  drawer.close();
}

// Subscription

BackAndroid.addEventListener('hardwareBackPress', () => {
  if (Modal.close())      { return true; }
  if (Drawer.close())     { return true; }
  if (Navigator.goBack()) { return true; }

  return false;
});
```

## Push notifications

I found implementing push on Android much harder than for iOS. In Android, it is the responsibility of the developer to manage how notifications are displayed, if there should be buttons displayed, and which sound should play, but also if there should be a notification at all.

Here is the documentation about how to set up [Google GCM](https://developers.google.com/cloud-messaging/android/client#get-config). Google GCM is registered into the app by using [Services](http://developer.android.com/guide/components/services.html) which are operations running in the background.

I looked at [react-native-push-notification](https://github.com/zo0r/react-native-push-notification) to understand how to pass down the GCM push token and notification content to the JavaScript runtime.

Push notifications are really important for Taskers, so we want them to be able to respond quickly to Clients when being notified when new tasks are available. In the beginning, we were facing an issue where GCM push tokens were old and were taking a long time to be updated, so we decided to start our service that extends `IntentService` after a `AppState`'s `activated` event is triggered. This seems to fix the issue.

To make sure all Client's messages are received, on every message, we will push to the phone. If we were to simply show all notifications, it could be really annoying. Imagine that you were on a chat and seeing push notifications popping up from the discussion you are seeing.

For this you would not create a notification but would rather emit an event through the bridge.

The class that extends `GcmListenerService` would be like this:

```
public class RNPushNotificationListenerService extends GcmListenerService {
    public void onMessageReceived(String from, Bundle bundle) {
        if (isApplicationRunning()) {
            Intent intent = new Intent("RNPushNotificationReceiveNotification");
            bundle.putBoolean("foreground", true);
            intent.putExtra("notification", bundle);
            sendBroadcast(intent);
        } else {
          new RNPushNotificationHelper(getApplicationContext()).sendNotification(bundle);
        }
    }
}
```

However, this was a naïve approach. In Android, sometimes the app can be running in background, or the screen can be locked, or even stranger case: the screen was not locked but just black. The check ended up being `isApplicationInForeground() && isScreenUnLocked() && isScreenOn()`.

### Error Handling

Before releasing the app, we wanted to make sure that we could monitor errors. We first used [Crashlytics](http://try.crashlytics.com/), but it seemed that issues were taking a long time to show up. We have now switched to [Bugsnag](https://bugsnag.com/).

We choose to set up Bugsnag natively as we wanted to get device informations (OS version, model, battery life, etc.). After the integration, we noticed that JavaScript errors were all grouped together because it uses the Java and Objective-C stack trace. To resolve the issue, we decided to manage the errors ourselves by wrapping `ErrorUtils._globalHandler` and writing a `ErrorManager` native module that triggers a notification to Bugsnag.

```
if (ErrorManager && ErrorUtils._globalHandler) {
  const previousGlobalHandler = ErrorUtils._globalHandler;
  const wrapGlobalHandler = (error, isFatal) => {
    let currentExceptionID = ++exceptionID;
    const stack = parseErrorStack(error);

    const timeoutPromise = new Promise((resolve) => {
      global.setTimeout(() => {
        resolve();
      }, 1000);
    });

    const reportExceptionPromise = new Promise((resolve) => {
      ErrorManager.reportException(error.message, stack, currentExceptionID, {}, resolve);
    });

    return Promise.race([reportExceptionPromise, timeoutPromise]).then(() => {
      previousGlobalHandler(error, isFatal);
    });
  };
  ErrorUtils.setGlobalHandler(wrapGlobalHandler);
}
```

## Android current progress UPDATED:

I provided a table detailing our progress [3 months ago](http://jrichardlai.com/2016/01/11/react-native-android/). Here is the updated version (I removed the rows that became part of React Native).

- ✅ The platform implementation is done
- 🔶 Have a Workaround
- ⛔ Not implemented / Work in progress

<br/>


|   Status   |   Class    |    Purpose    | iOS    | Android| iOS and Android |
| ✅| Actionsheet | Display a list of actions | ActionSheetIOS | Decided to use native dialogs from [react-native-dialogs](https://github.com/aakashns/react-native-dialogs) |
| ✅| ActivityIndicator | Loading indicator | ActivityIndicatorIOS | ProgressBarAndroid | |
| ✅| Alert | Display an alert to the user | AlertIOS | Uses Alert and [react-native-dialogs](https://github.com/aakashns/react-native-dialogs) for prompt |
| ✅| DatePicker | Picking a date and time | DatePickerIOS | Uses DatePickerAndroid and TimePickerAndroid |
| ✅| ExpandingTextInput | Text that auto expand on input |  |  | nativeEvent.contentSize |
| ✅| FacebookLogin | Log in with Facebook | | | [react-native-facebook-login](https://github.com/magus/react-native-facebook-login), Facebook just released an alpha that supports android now for [react-native-fbsdk](https://github.com/facebook/react-native-fbsdk)
| ✅| ImagePickerManager | Picking an image from the library or taking a photo | | | Using [react-native-fs](https://github.com/johanneslumpe/react-native-fs) and [react-native-image-picker](https://github.com/marcshilling/react-native-image-picker)
| ✅| Linking | Open url or app | LinkingIOS | IntentAndroid |
| 🔶| Modal | Open a Modal | ModalIOS | Using a [gist](https://gist.github.com/mkonicek/1a8bd7253e3257687228) from [Martin Konicek](https://twitter.com/martinkonicek) using Portal (be sure to wrap your Component with a Context Wrapper if needed)
| ✅| MapView | Displaying a map with circles | | | [react-native-maps](https://github.com/lelandrichardson/react-native-maps)
| ⛔| ProgressCircle | Display a progress circle | | Android strokeDash is [not available yet](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/views/art/ARTShapeShadowNode.java#L186).
| ✅| Tooltip | Show a Tooltip on long press | Uses [react-native-tooltip](https://github.com/taskrabbit/react-native-tooltip) | Decided to use native dialogs [react-native-dialogs](https://github.com/aakashns/react-native-dialogs)
| ✅| ZendeskChat | Allow ZendeskChat in App | | | [react-native-zendesk-chat](https://github.com/taskrabbit/react-native-zendesk-chat) integrating [Zopim Mobile SDK](https://github.com/zendesk/zendesk_sdk_chat_ios)

<br/>

Time to build new things! We're really looking forward to that.
