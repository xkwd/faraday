## Faraday 2.0

### Adapters have moved!

With this release, we've officially moved all adapters out of Faraday.
What that means, is that they won't be available out-of-the-box anymore,
and you'll instead need to add them to your Gemfile.

#### Why was this decision made?

We've taken this decision for the following technical reasons:

* We wanted the Faraday gem to focus on providing a clean, standardized, middleware-stack-based API.
* We wanted to free the core team from maintaining all the different adapters, relying more on the community to
  maintain them based on the broad interest. This will allow the core team to focus on implementing features
  focused on the Faraday API more quickly, without having to push it on all adapters immediately.
* With the community creating more and more adapters, we wanted to avoid having first and second-class adapters
  by having some of them included with the gem and others available externally.
* Moving adapters into separate gems allow to solve the dependency issues once and for all.
  Faraday will remain a dependency-free gem, while adapter gems will be able to automatically pull
  any necessary dependency, without having to rely on the developer to do so.

#### So what will this mean for me?

We did our best to make this transition as painless as possible for you, so here is what we did.

* All adapters have already been moved out and released as separate gems.
  They've then been re-added into Faraday's dependencies so that you wouldn't notice.
  This means that immediately after v2.0 will be released, all the adapters that were previously available will be
  already compatible with Faraday 2.0!
* We've setup an [Awesome Faraday](https://github.com/lostisland/awesome-faraday) repository, where you can find and discover adapters.
  We also highlighted their unique features and level of compliance with Faraday's features.

#### That's great! What should I change in my code immediately after upgrading?

* Add the corresponding adapter gem to your Gemfile (e.g. `faraday-net_http`). Ideally, this should replace
  `faraday` altogether as these gems usually have Faraday already in their dependencies.
* If you're relying on `Faraday.default_adapter` (e.g. if you use `Faraday.get` or other verb class methods, or not
  specifying an adapter in your connection initializer), then you'll now need to set it yourself. It previously
  defaulted to `:net_http`, but it now defaults to `:test`. You can do so simply by using the setter:
  
  ```ruby
  # For example, to use net_http (previous default value, will now require `gem 'faraday-net_http'` in your gemfile)
  Faraday.default_adapter = :net_http
  ```

### Faraday Middleware Deprecation

In case you never used it, [Faraday Middleware](https://github.com/lostisland/faraday_middleware) is a handy collection of middleware that have so far been maintained by the Faraday core team.
With Faraday 2.0 focusing on becoming an ecosystem, rather than an out-of-the-box solution, it only made sense to take the same approach for middleware as we did for adapters. For this reason, `faraday_middleware` will not be updated to support Faraday 2.0.
Instead, we'll support the transition from centralised-repo collection of middleware to individual middleware gems, effectively replicating what we did with adapters. Each middleware will have its own repository and gem, and it will allow developers to only add the ones they require to their Gemfile.

So don't fret yet! We're doing our best to support our `faraday_middleware` users out there, so here are the steps we've taken to make this work:
* We've promoted the highly popular JSON middleware (both request encoding and response parsing) to a core middleware and it will now be shipped together with Faraday. We expect many `faraday_middleware` users will be able to just stop using the extra dependency thanks to this.
* We've created a [faraday-middleware-template](https://github.com/lostisland/faraday-middleware-template) repository that will make creating a new middleware gem simple and straightforward.
* We've added a middleware section to the [awesome-faraday](https://github.com/lostisland/awesome-faraday) repo, so you can easily find available middleware when you need it.

It will obviously take time for some of the middleware in `faraday_middleware` to make its way into a separate gem, so we appreciate this might be an upgrade obstacle for some. However this is part of an effort to focus the core team resources tackling the most requested features.
We'll be listening to the community and prioritize middleware that are most used, and will be welcoming contributors who want to become owners of the middleware when these become separate from the `faraday_middleware` repo.

### Autoloading and dependencies

Faraday has until now provided and relied on a complex dynamic dependencies system.
This would allow to reference classes and require dependencies only when needed (e.g. when running the first request) based
on the middleware/adapters used.
As part of Faraday v2.0, we've removed all external dependencies, which means we don't really need this anymore.
This change should not affect you directly, but if you're registering middleware then be aware of the new syntax:

```ruby
# `name` here can be anything you want.
# `klass` is your custom middleware class.
# This method can also be called on `Faraday::Adapter`, `Faraday::Request` and `Faraday::Response`
Faraday::Middleware.register_middleware(name: klass)
```

### Authentication helper methods in Connection have been removed

You were previously able to call `authorization`, `basic_auth` and `token_auth` against the `Connection` object, but this helper methods have now been dropped.
Instead, you should be using the equivalent middleware, as explained in [this page](https://lostisland.github.io/faraday/middleware/authentication).

For more details, see https://github.com/lostisland/faraday/pull/1306

### Others

* Rename `Faraday::Request#method` to `#http_method`.
* Remove `Faraday::Response::Middleware`. You can now use the new `on_complete` callback provided by `Faraday::Middleware`.
* Drop `Faraday::UploadIO` in favour of `Faraday::FilePart`.

## Faraday 1.0

### Errors
* Removes sub-class constants definition from `Faraday::Error`. A sub-class (e.g. `ClientError`) was previously accessible
either through the `Faraday` module (e.g. `Faraday::ClientError`) or through the `Faraday::Error` class (e.g. `Faraday::Error::ClientError`).
The latter is no longer available and the former should be used instead, so check your `rescue`s.
* Introduces a new `Faraday::ServerError` (5xx status codes) alongside the existing `Faraday::ClientError` (4xx status codes).
Please note `Faraday::ClientError` was previously used for both.
* Introduces new Errors that describe the most common REST status codes:
  * Faraday::BadRequestError (400)
  * Faraday::UnauthorizedError (401)
  * Faraday::ForbiddenError (403)
  * Faraday::ProxyAuthError (407). Please note this raised a `Faraday::ConnectionFailed` before.
  * Faraday::ConflictError (409)
  * Faraday::UnprocessableEntityError (422)
* The following error classes have changed the hierarchy to better mirror their real-world usage and semantic meaning:
  * TimeoutError < ServerError (was < ClientError)
  * ConnectionFailed < Error (was < ClientError)
  * SSLError < Error (was < ClientError)
  * ParsingError < Error (was < ClientError)
  * RetriableResponse < Error (was < ClientError)

### Custom adapters
If you have written a custom adapter, please be aware that `env.body` is now an alias to the two new properties `request_body` and `response_body`.
This should work without you noticing if your adapter inherits from `Faraday::Adapter` and calls `save_response`, but if it doesn't, then please ensure you set the `status` BEFORE the `body` while processing the response.

### Others
* Dropped support for jruby and Rubinius.
* Officially supports Ruby 2.4+
* In order to specify the adapter you now MUST use the `#adapter` method on the connection builder. If you don't do so and your adapter inherits from `Faraday::Adapter` then Faraday will raise an exception. Otherwise, Faraday will automatically push the default adapter at the end of the stack causing your request to be executed twice.
```ruby
class OfficialAdapter < Faraday::Adapter
  ...
end

class MyAdapter
  ...
end

# This will raise an exception
conn = Faraday.new(...) do |f|
  f.use OfficialAdapter
end

# This will cause Faraday inserting the default adapter at the end of the stack
conn = Faraday.new(...) do |f|
  f.use MyAdapter
end

# You MUST use `adapter` method
conn = Faraday.new(...) do |f|
  f.adapter AnyAdapter
end
```
