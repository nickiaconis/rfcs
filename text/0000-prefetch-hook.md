- Start Date: 2015-10-01
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Ember's data fetching is sequential.
Although this has advantages (one of which being that it presents a simpler programming model), the resulting latency of each request is compounded and can result in a degraded experience.
The general solution to the "waterfall" problem of sequential requests is to execute independent requests in parallel.

Automaticly inferring which requests can be made in parallel is tricky, and likely impossible to achieve while maintaining a semver compatible public API.
Although it may not be automatic, developers have sufficient context with which they can decide what requests to make in parallel.

This RFC proposes an approach that allows developers to tune the balance between sequential and parallel requests.

# Motivation

Currently, the `model` hook is Ember's way for routes to request data.
However, a route's `model` hook is not entered until it's parent route has resolved.
This is suboptimal for nested application structures.
Child routes are forced to wait for their parents to receive data before they may make requests for their own data.
This occurs even when the child doesn't make use of its parents' data.

This RFC proposes adding a new `Route#prefetch` hook, which is used to allow routes to make requests in parallel.
Child routes that make use of `prefetch` will resolve faster since their data can resolve at the same time as (or before) their parents'.
The semantics and ordering of `Route`'s existing model hooks (`beforeModel`, `model`, `afterModel`) are preserved.

See this demo for a visualization: http://nickiaconis.github.io/ember-parallel-model-demo/

# Detailed design

A `prefetch` hook is added to `Route`.
It takes the same parameters as the `model` hook.
Like the `model` hook, it is not called if an object is passed to the transition.

```javascript
App.PostRoute = Ember.Route.extend({
  prefetch(params) {
    return Ember.$.get(`/api/posts/${params.id}`);
  }
});

App.PostCommentsRoute = Ember.Route.extend({
  prefetch(params, transition) {
    return Ember.$.get(`/api/posts/${transition.params.post.id}/comments`);
  }
});
```

The default functionality of the `model` hook is modified to return the prefetched data if it exists.
As such, a route that defines a `prefetch` hook is not required to define a `model` hook.

A `prefetched` method, which takes an optional `name` parameter and always returns a promise, is added to `Route`.
It is used to access data fetched by the named route's `prefetch` hook.
If `name` is omitted, the method will return a promise for its own route's prefetched data.
The default `model` hook utilizes it in this way.

```javascript
App.PostCommentsRoute = Ember.Route.extend({
  prefetch(params, transition) {
    return Ember.$.get(`/api/posts/${transition.params.post.id}/comments`);
  },

  async model() {
    return {
      OP: this.modelFor('post')).author,
      comments: await this.prefetched()
    };
  }
});
```

# Drawbacks

- Ember's API becomes larger.

# Alternatives

- Implement the functionality as an addon.
  - Would require Ember to provide something like `Router#willTransition` that is triggered on redirects. It could be triggered on either all transitions (`Router#willChangeURL`?) or only redirects (`Router#changeTransition`? to supplement `willTransition`).
    - Otherwise, the addon must change the functionality of `Router#willTransition` in order to function properly.
  - Would benefit from guarantees of stability around `Transition#handlerInfos` and `handlerInfo#runSharedModelHook`.
