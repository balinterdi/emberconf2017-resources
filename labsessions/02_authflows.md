# Authentication with OAuth 2.0

We start with an application that already provides authentication via
email/password. Also, the back-end is ready to serve all the authorization flows
we need during the workshop so that we can focus on the front-end.

## Back-end

The back-end is a JSON API compliant (Rails) app. The main endpoint you'll have
to use during the workshop serves to validate the access token (or authorization
code) and return you the user's email and the token you can use to authorize
your requests to the backend:

    POST /tokens/validate

    Request headers:
      'Accept': 'application/vnd.api+json'
      'Content-Type': 'application/vnd.api+json'
    Request payload:
      code: the authorization code (required for the authorization code flows)
      access_token: the token you received from Google (required for the implicit grant flow)
      provider: the name of the 3rd party provider, 'google', 'github' or 'facebook' (required)
    Response:
      user_email: the email of the authenticated user
      token: the JWT token used as a session identifier with the backend

The headers for all requests to the API need to include the following two:

    'Accept': 'application/vnd.api+json',
    'Content-Type': 'application/vnd.api+json',

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

We'll see an example on how to do that for the implicit grant (sign in via
Google) scenario. Adjusting this for the authorization code flow (Github &
Facebook) is left as an exercise for the reader.

--------

The main interface of ESA is the `session` service. To initiate the
authentication process, you call `authenticate` on it, passing in the name of
the `authenticator` and any parameters it needs. For example:

    this.get('session').authenticate('authenticator:github-auth-code', 'github-oauth2');

    this.get('session').authenticate('authenticator:google-implicit-grant', 'google-oauth2-bearer');

The referenced authenticator is called `github-auth-code` and needs to exist as
a file under `app/authenticators`. The easiest way to create an authenticator is
by using the generator provided by the add-on:

    $ ember g authenticator google-implicit-grant

You can also pass the `base-class` option that will make the generated
authenticator extend that base class. ESA provides the `torii` authenticator to
integrate with Torii so the command becomes:

    $ ember g authenticator google-implicit-grant  --base-class=torii

```js
// app/authenticators/google-implicit-grant.js
import Torii from 'ember-simple-auth/authenticators/torii';

export default Torii.extend({
  torii: Ember.inject.service('torii')
});
```

When you call `session.authenticate` on a Torii authenticator, the second
parameter should be the name of the Torii provider that you want to use
(`google-oauth2-bearer` in the above example). You can list the full list
[here][list-of-torii-providers].

What you return from the `authenticate` method of the `authenticator` will be
set into the ESA session and persisted in your browser to be restored later:

![](images/persisted-esa-session-information.png)

In the current application, this return value should be an object with the
following keys:

    userEmail: the `user_email` value received from the backend
    token: the `token` value received from the backend
    provider: the name of the Torii provider (e.g `google-oauth2-bearer`)

So you need to implement the `authenticate` method of this authenticator.

TODO: Talk about restoring the session with `restore`

### Hint

Calling `this._super` in your authenticator's `authenticate` method will do the
whole OAuth authorization flow with Google and return an object that has an
`authorizationToken` and `provider` parameter.

The `authorizationToken` is the one you have to exchange for user information
(email and token)  with the backend. See the Back-end section at the start of
this guide on how to do this.

```js
// app/authenticators/google-implicit-grant.js
import Torii from 'ember-simple-auth/authenticators/torii';

export default Torii.extend({
  torii: Ember.inject.service('torii'),

  authenticate() {
    return this._super(...arguments)
      .then((params) => {
        // the actual Google access token can be accessed as
        // `params.authorizationToken.access_token`
        // the Torii provider, like `google-oauth2-bearer`
        // is in `params.provider`
        //
        // TODO: Send access token to back-end to
        // get a session token and the user email back
      });
  },
});
```

TODO CONTENT (show how the backend links oauth info to an account)

## Two-legged and three-legged flows

TODO CONTENT (implement both types of flows)

[list-of-torii-providers]: https://github.com/Vestorly/torii/tree/master/addon/providers
