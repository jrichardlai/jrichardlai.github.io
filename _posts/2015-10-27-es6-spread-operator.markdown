---
title: ES6 Spread Operator
description: Cleanup code
---

One of my favorite feature of ES6 is the `spread` operator that is represented by `...`. It is comparable in Ruby to the `splat` operator `*`.

I use this feature all the time in my code, it really simplify the code readability by removing repetition.

You can use it to merge array:

```javascript
// javascript
const previousArray = [3, 4];
const array = [1, 2, ...previousArray]; // => [1, 2, 3, 4];
```
```ruby
# ruby
array = [1, 2]
newArray = [*array, 3, 4] # => [1, 2, 3, 4];
```

Call a function that can get an unknown number of arguments:

```javascript
// javascript
function foo(...args) {
  console.log(args.length) // => 3
}

foo('a', 'b', 'c');

```
```ruby
# ruby
def foo(*args)
  puts args.size # => 3
end

foo('a', 'b', 'c');
```

Here is an example of refactoring by using the spread operator for the [Timeshift](https://github.com/plaa/TimeShift-js) library:

![](/assets/images/posts/es6-spread-operator/example-1.png)
![](/assets/images/posts/es6-spread-operator/example-2.png)

Here is a link to the [commit](https://github.com/jrichardlai/TimeShift-js/commit/93add531d5cd7fe25d42c6b56df546227c8eb344).
