---
title: Actionable Text in React Native
description: For all the hashtags lovers
---

With React Native, it is very easy to bind an action when a `Text` component is pressed by using the `onPress` props, however when you have a dynamic text and want to add press events to different parts corresponding to urls or emails it is a little bit trickier. In iOS it is possible to parse a text natively by using [dataDetectorTypes](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UITextView_Class/index.html#//apple_ref/occ/instp/UITextView/dataDetectorTypes) but unfortunately the `RCTTextView` does not provide this option and it would need to be implemented for Android too.

So I decided to write a React Native Component to automatically parse a text and make multiple interactive `Text` components from it.

Here comes [react-native-parsed-text](https://github.com/taskrabbit/react-native-parsed-text), to parse a text just use a `RegExp` to find the part you are looking for, you can also use one of the predefined type (`url`, `phone` or `email`).

## In Action:

![](https://cloud.githubusercontent.com/assets/159813/11152673/d5fe86f0-89e8-11e5-8b5e-f3c06bdc1b6b.gif)
