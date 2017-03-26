# Authentication with OAuth 2.0 in Ember

TODO INTRO (backend already implemented, focus on frontend)

## Getting acquianted

The current application is also set up to use work with the authenticated
backend, so let's see how that works.

If you take a look in `app/routes/login.js`, you'll find the following code:

```js
import Ember from 'ember';
import UnauthenticatedRouteMixin from 'ember-simple-auth/mixins/unauthenticated-route-mixin';

export default Ember.Route.extend(UnauthenticatedRouteMixin, {
  session: Ember.inject.service(),

  actions: {
    signIn() {
      let controller = this.get('controller');
      let { email, password } = controller.getProperties('email', 'password');
      return this.get('session').authenticate('authenticator:credentials', email, password)
        .then(() => {
          controller.set('errorMessage', '');
        })
        .catch((error) => {
          controller.set('errorMessage', error.error || error);
        });
    }
  }
});
```

When you click the "Let me in" button on the login page, this action is executed.

It looks up the `credentials` authenticator, and sends it the inputted email
and password to initiate the authentication process. Let's look at the authenticator next:

```js
// app/authenticators/credentials.js
import Ember from 'ember';
import Base from 'ember-simple-auth/authenticators/base';

const { RSVP } = Ember;

export default Base.extend({
  ajax:  Ember.inject.service(),
  store: Ember.inject.service(),

  authenticate(username, password) {
    return this.get('ajax').post('/token', {
      headers: {
        'Accept': 'application/vnd.api+json',
        'Content-Type': 'application/vnd.api+json',
      },
      data: JSON.stringify({
        username,
        password
      })
    }).then(({ user_email: userEmail, token }) => {
      return {
        userEmail,
        token
      };
    });
  },

  (...)
});
```

The `authenticate` method is called by ESA (Ember Simple Auth) with the
parameters we passed in to the session's authenticate call previously.

We then send the email and password to the API endpoint that authenticates the
user. The API responds with a `user_email` and `token`. We camelize these keys
and return the object for ESA to set it as authenticated data (in
`session.data.authenticated`).

The token is what we'll need to include in all subsequent requests to authorize
them. We do this in the so-called authorizer, found under `app/authorizers/jwt.js`:

```js
// app/authorizers/jwt.js
import Ember from 'ember';
import Base from 'ember-simple-auth/authorizers/base';

const { isEmpty } = Ember;

export default Base.extend({
  authorize(data, block) {
    let token = data['token'];

    if (!isEmpty(token)) {
      block('Authorization', `Bearer ${token}`);
    }
  }
});
```

This authorizer sets the token as the Authorization header in all outgoing requests.

But how do we tell ESA to use this authorizer? The answer lies in defining
that, which is done in the application adapter.

```js
// app/adapters/application.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';
import DataAdapterMixin from 'ember-simple-auth/mixins/data-adapter-mixin';

export default JSONAPIAdapter.extend(DataAdapterMixin, {
  authorizer: 'authorizer:jwt',
});
```

The `DataAdapterMixin` provided by ESA is the one that connects the above
defined authorizer with the Ember Data adapter. The `authorizer` property tells
ESA which authorizer to use.

## Running flows with Torii

Torii provides a clean set of abstractions (providers, adapters and the
session) to implement the authorization flows in your application. It handles
the whole redirection flow process, opens and closes the popup, and makes sure
security best practices (like using and verifying a `state` parameter) are
followed.

On top of that, it also implements numerous 3rd party OAuth2 providers, like
Google, Facebook, Twitter, Github, etc., so working with these providers
becomes a breeze. Its flexible architecture makes it easy to implement
providers that are not bundled into the add-on.

Integrating a provider can be as easy as just configuring the keys and redirect
URLs:

```js
// config/environment.js

module.exports = function(environment) {
  var ENV = {
    torii: {
      providers: {
        'github-oauth2': {
          clientId: 'e309dba06c2f82915da8',
          redirectUri: 'http://localhost:4200/oauth2callback'
        },
      }
    },
    (...)
  };
  (...)
};
```

## ember-simple-auth (ESA) with Torii

Session management is opt in with Torii, which makes it integrate very nicely
with ESA. In this scenario, ESA manages the session and delegates 3rd-party
authorization to Torii.

The main interface of ESA is the `session` service. To initiate the
authentication process, you call `authenticate` on it, passing in the name of
the `authenticator` and any parameters it needs. For example:

    this.get('session').authenticate('authenticator:github-auth-code', 'github-oauth2');

The referenced authenticator is called `github-auth-code` and needs to exist as
a file under `app/authenticators`. The easiest way to create an authenticator is
by using the generator provided by the add-on:

    $ ember g authenticator github-auth-code

You can also pass the `base-class` option that will make the generated
authenticator extend that base class. ESA provides the `torii` authenticator to
integrate with Torii so the command becomes:

    $ ember g authenticator github-auth-code --base-class=torii

```js
// app/authenticators/github-auth-code.js
import Torii from 'ember-simple-auth/authenticators/torii';

export default Torii.extend({
  torii: Ember.inject.service('torii')
});
```

When you call `session.authenticate` on a Torii authenticator, the second
parameter is the Torii provider that you want to use (`github-oauth2` in the
above example). You can list the full list [here][list-of-torii-providers].

So you need to implement the `authenticate` method of this authenticator.

Calling `this._super` will do the whole OAuth authorization flow with Github and
return an object that has an `authorizationCode` and `provider` parameter.

The `authorizationCode` is the one you have to exchange for an access token with
the backend. The backend has the following endpoint for this:

    POST /tokens/validate

    Request headers:
      'Accept': 'application/vnd.api+json'
      'Content-Type': 'application/vnd.api+json'
    Request payload:
      code: the authorization code (required)
      provider: the name of the 3rd party provider, 'google', 'github' or
      'facebook' (required)
    Response:
      user_email: the email of the authenticated user
      token: the JWT token used as a session identifier with the backend


What you return from the `authenticate` method of the `authenticator` will be
set into the ESA session and persisted in your browser to be restored later:

![](images/persisted-esa-session-information.png)

In the current application, this return value should be an object with the
following keys:

    userEmail: the `user_email` value received from the backend
    token: the `token` value received from the backend
    provider: the value received from `this._super` (e.g github-oauth2)

```js
// app/authenticators/github-auth-code.js
import Torii from 'ember-simple-auth/authenticators/torii';

export default Torii.extend({
  torii: Ember.inject.service('torii'),

  authenticate() {
    return this._super(...arguments)
      .then((params) => {
        // params have authorizationCode and provider
        //TODO: Exchange code with backend and return
        // what needs to be set into the session
      });
  },
});
```

TODO: Talk about restoring the session with `restore`

TODO CONTENT (how to integrate these two together)

TODO CONTENT (show how the backend links oauth info to an account)

## Two-legged and three-legged flows

TODO CONTENT (implement both types of flows)

[list-of-torii-providers]: https://github.com/Vestorly/torii/tree/master/addon/providers
