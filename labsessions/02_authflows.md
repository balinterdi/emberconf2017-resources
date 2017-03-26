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

TODO CONTENT (how to set up Torii, without tying it into ESA)

## Torii and ember-simple-auth

TODO CONTENT (how to integrate these two together)

TODO CONTENT (show how the backend links oauth info to an account)

## Two-legged and three-legged flows

TODO CONTENT (implement both types of flows)

