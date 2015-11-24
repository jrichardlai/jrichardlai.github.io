---
title: Documenting an API by testing
description: Kill two birds with one stone
---

At [TaskRabbit](https://www.taskrabbit.com) we are providing an API to our Mobile and Web clients. As an efficient Agile Team, we are moving fast, new features can be shipped and/or updated daily.

An advantage of our environment is that we do not have external clients that consume our API and have a limited number of internal clients.

Another strength is that our team works closely together. This allows us to be more lenient than we would be in other environments when dealing with API changes.

When the product needs a change in the API, the followed guideline is:

* The following updates won’t require an API version bump:
  * Adding a simple property in the response (an attribute from a table, a translation, cached data).
  * Allowing an extra HTTP parameter from the client.
  * Removing a property that was never used by a client.
  * Removing a property from a client that is fully deprecated ( app will not allow you to start and ask to update ).
  * API backend refactoring/change that does not modify the response.

This methodology requires however rigorous testing and documentation, in this regard we created a gem [tests_doc](https://github.com/taskrabbit/tests_doc) allowing us to ensure our request’s tests create the right response and to share API responses quickly with the Mobile team. This also helps to keep track of unwanted changes.

To record an api interaction a minimal change in your test is required:

```ruby
recording_api_interaction do
  get users_path(popular: true)
  expect(response_body.count).to eq(2)
  # other expectations
  expect(response.status).to be(200)
end
```

Will generate a file `users.md`, then we simply share this file to the team, for more information about the options and configuration go to the tests_doc [github page](https://github.com/taskrabbit/tests_doc).

Example Output:

![](/assets/images/posts/documenting-api-by-testing/example.png){:width="500px"}
