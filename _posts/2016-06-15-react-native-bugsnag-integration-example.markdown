---
title: "Bugsnag integration example with React Native"
description: Android & iOS
---

**Note:** This might not be relevant soon since Bugsnag seems to be working [on a React Native integration](https://twitter.com/bugsnag/status/732343569042677760?ref_src=twsrc%5Etfw).

Handling errors and being notified about customers crashes is really important when your customers grows. This is even more important for React Native as this is a new and evolving fast technology, but also as it is used by different platforms.

As noted in the previous post at [TaskRabbit](https://www.taskrabbit.com) we first tried [Crashlytics](http://try.crashlytics.com/) but were not pleased with the delay in notifications and grouping of errors. I looked around and the only solution I found was [Sentry](https://docs.getsentry.com/hosted/clients/javascript/integrations/react-native/) but the service didn't quite fit our needs, as we were using [Bugsnag](https://bugsnag.com/) for our Ruby On Rails web app, I decided to take a stab at it and see what would be required to make it work with React Native.

The goal with the error reporting was to group errors per platform, so I made a Bugsnag project for Android and another for iOS, I wanted to be able to see which phone (model, version) was affected by crashes (another way to do it is would be to make 3 Bugsnag projects: Javascript, iOS and Android).

### Setting up Bugsnag in your React Native App

Here are steps that we took to set up Bugsnag in our app:

- Create a project for the Android app and the iOS App on Bugsnag.
- Integrate Bugsnag in the native code ([iOS](http://docs.bugsnag.com/platforms/ios/) \|  [Android](http://docs.bugsnag.com/platforms/android/))
- Create a `ErrorManager` NativeModule class in iOS and Android that exports 2 methods:
  - `reportException(message: String, stack: Array<stackFrame>, exceptionID: String, extraData: Object, callback: function)`: Format the data and send it to Bugsnag.
  - `setIdentifier(attributes: Object)`: Set the user's id, email and name.
- Hook up errors to call our `ErrorManager.reportException`: The default behavior for React Native with exception is to just crash, the behavior is handled by `ExceptionsManager`. One way to override it is by using `ErrorUtils.setGlobalHandler`, I didn't wanted to modify the default behavior so I decided to just wrap the default handler which is stored in `ErrorUtils._globalHandler`:

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

- In our Javascript code after logging in our user we call `ErrorManager.setIdentifier` to set the user informations.
- Another thing we did was to also add a global `notifyError` method that we call when there is an error but without crashing the app.

```
global.notifyError = (error, errorData) => {
  console.log('notifyError', error, errorData);
  let currentExceptionID = ++exceptionID;
  const stack = parseErrorStack(error);

  ErrorManager.reportException(
    error.message,
    stack,
    currentExceptionID,
    // Make all values string
    _.mapObject(errorData || {}, (val, key) => (val || 'NULL').toString()),
    () => {}
  );
}
```

Here is a [gist](https://gist.github.com/jrichardlai/072d2f13098f38df504a7fe031a803bb) of the implementation.

## Extra Step: Sourcemap through the bridge

**Note** The following solution is naive and might not fit everyone.

However as in production the javascript bundle is minified, the error doesn't show the full stacktrace, this makes it a little tricky to debug.

We decided that it was okay to put the sourcemap in the project. The following solution is not optimal, the performance is not great as the sourcemap content is loaded through the bridge and then parsed in Javascript.

You can either load it on app startup or do it when an error occurs, which means that either the startup time will be slower or when an error occurs it might take more time.

After some testings we decided that it was okay to do it when an error occurs.

Here are the steps we took to use the sourcemap in our Bugsnag notification:

- Add the sourcemap to the build. You can do this by hand after building the app or you can make it part of your build with some changes in your build phases:
  - For Android you can generally edit (or make a copy of) `react.gradle` and add to the bundle command the `"--sourcemap-output", "$jsBundleDir/sourcemap.js"` options
  - For iOS you can make a copy of `react-native-xcode.sh` append `--sourcemap-output "$DEST/sourcemap.js"` to the command and make your app run the new script.
- Add `getSourceMaps(callback: function)` to `ErrorManager`, which callback will be called with the source map data.
- Modify the error handler to load the sourcemap then trigger the error.
