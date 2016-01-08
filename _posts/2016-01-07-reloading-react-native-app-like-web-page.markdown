---
title: Reloading your React Native App like a Web page
description: Magic with Live Reload
---

At [TaskRabbit](https://www.taskrabbit.com), the Tasker App is using a similar infrastructure as the [React Native Sample App](https://github.com/taskrabbit/ReactNativeSampleApp).

One of the core foundation of the App is the Navigation pattern based on URLs. As noted [before](http://tech.taskrabbit.com/blog/2015/09/21/react-native-example-app/), URLs mapping to Component stacks require that all screens can be rendered by themselves from just the URL. This dramatically simplifies deep linking, but it also has the ability to speed up the development process.

As a web developer, I am quite used to thinking about URLs and really love the RESTfulness they imply. When you send someone a URL, you know which page they will see (*assuming correct permissions to the resource*). Also in development, one is able to refresh a page after updating the code. This is a common cycle, of course.

So there I was facing a major pain point with React Native, as currently there is no easy Hot Reloading solution (*but seems to be soon yay!*).  When updating the code to see the changes you have to go back to the exact same spot to see the change, which can be painful when it takes multiple taps to get to that location in the app.

But that was easy to fix.

By using `AsyncStorage`, I stored the current path in a `DebugStore`. Then after reloading the javascript, I can hydrate back the current path from the store and voilà:

**A Native App acting like a Web page.**

I just updated the [React Native Sample App](https://github.com/taskrabbit/ReactNativeSampleApp) with the implementation. I've been using the `DebugStore` in our app since the beginning and it does make a huge difference.

Here is an example of manual reload and live reload with Android and iOS (*Note: the live reload for Android at the end was acting a bit funny*):

![](https://cloud.githubusercontent.com/assets/159813/12137052/09019338-b402-11e5-8457-b0028b7af3de.gif)
