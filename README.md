# loopback-passport

The module provides integration between [LoopBack](http://loopback.io) and 
[Passport](http://passportjs.org) to support third party login and account 
linking for LoopBack applications.

# Use cases

## Third party login

Social login becomes popular these days as our users don’t want to deal with so 
many identities. It would be nice to allow the use of a third party provider 
such as Facebook, Google, Twitter, or Github to log into LoopBack. The login 
profiles will be tracked and associated with corresponding LoopBack users. 

## Linked accounts

In LoopBack, most APIs will be built using models that are backed by data 
sources, which in turn uses connectors to interact with other systems or cloud 
services. Some of the backend systems require user-specific credentials to 
access the protected resources. For example, an e-commerce engine requires the 
user credential to see the order history. It’s also true to get pictures from 
one or more facebook accounts. One solution to this requirement is to link or 
pre-authorize a LoopBack user to other accounts.

# Key components

![Key Components](ids_and_credentials.png)

## UserIdentity model

UserIdentity model keeps track of 3rd party login profiles. Each user identity
is uniquely identified by provider and externalId. UserIdentity model comes with
a 'belongsTo' relation to the User model.

Properties

- {String} provider: The auth provider name, such as facebook, google, twitter, linkedin
- {String} authScheme: The auth scheme, such as oAuth, oAuth 2.0, OpenID, OpenID Connect
- {String} externalId: The provider specific user id
- {Object} profile: The user profile, see http://passportjs.org/guide/profile
- {Object} credentials
  - oAuth: token, tokenSecret
  - oAuth 2.0: accessToken, refreshToken
  - OpenID: openId
  - OpenID Connect: accessToken, refreshToken, profile
- {*} userId: The LoopBack user id
- {Date} created: The created date
- {Date} modified: The last modified date

## UserCredential model

UserCredential has the same set of properties as UserIdentity. It's used to 
store the credentials from a third party authentication/authorization provider
to represent the permissions/authorizations from a user from the third party 
system. 

## ApplicationCredential model

Interacting with third party systems often require some client application level
credentials. For example, you will need oAuth 2.0 client id and client secret to 
call facebook APIs. Such credentials can be supplied from a configuration file 
to your server globally. But if your server accepts API requests from multiple
client applications, each client application should have its own credentials. To
support the multi tenancy, this module provides the ApplicationCredential model
to store credentials associated with a client application.

Properties

- {String} provider: The auth provider name, such as facebook, google, twitter, linkedin
- {String} authScheme: The auth scheme, such as oAuth, oAuth 2.0, OpenID, OpenID Connect
- {Object} credentials: The provider specific credentials
  - openId: {returnURL: String, realm: String}
  - oAuth2: {clientID: String, clientSecret: String, callbackURL: String}
  - oAuth: {consumerKey: String, consumerSecret: String, callbackURL: String}
- {Date} created: The created date
- {Date} modified: The last modified date

ApplicationCredential model comes with a 'belongsTo' relation to the Application 
model.

## PassportConfigurator

PassportConfigurator is the bridge between LoopBack and Passport. 

- set up models with LoopBack
- initialize passport
- load the provider configuration and set them as Passport strategies with 
corresponding routes for auth and callback. 

# Flows

## Third party login flow

1. A visitor uses Facebook (or other providers) login
2. LoopBack receives the Facebook profile
3. LoopBack searches the UserIdentity model by (provider, externalId) to see 
there is an existing LoopBack user for the given Facebook id
4. If yes, set the LoopBack user to the current context
5. If not, create a LoopBack user from the profile and create a corresponding 
record in UserIdentity to track the 3rd party login. Set the newly created user 
to the current context.

## Third party account linking flow

1. Log into LoopBack first
2. Access a backend on behalf of the logged in LoopBack user
3. Link accounts for a given backend to the current user

# Use the module with a LoopBack application

A demo application is built with this module to showcase how to use the APIs 
with a LoopBack application. The code is available at:

[https://github.com/strongloop-community/loopback-example-passport](https://github.com/strongloop-community/loopback-example-passport)

## Configure third party providers

The following example shows two providers: facebook-login for login with 
facebook and google-link for linking your google accounts with the current 
LoopBack user.

```json
{
  "facebook-login": {
    "provider": "facebook",
    "module": "passport-facebook",
    "clientID": "{facebook-client-id-1}",
    "clientSecret": "{facebook-client-secret-1}",
    "callbackURL": "http://localhost:3000/auth/facebook/callback",
    "authPath": "/auth/facebook",
    "callbackPath": "/auth/facebook/callback",
    "successRedirect": "/auth/account",
    "scope": ["email"]
  },
  ...
  "google-link": {
    "provider": "google",
    "module": "passport-google-oauth",
    "strategy": "OAuth2Strategy",
    "clientID": "{google-client-id-2}",
    "clientSecret": "{google-client-secret-2}",
    "callbackURL": "http://localhost:3000/link/google/callback",
    "authPath": "/link/google",
    "callbackPath": "/link/google/callback",
    "successRedirect": "/link/account",
    "scope": ["email", "profile"],
    "link": true
  }
}
```

**NOTE**

You'll need to register with facebook and google to get your own client id and 
client secret.

- Facebook: https://developers.facebook.com/apps
- Google: https://console.developers.google.com/project

## Add code snippets to app.js

```
var loopback = require('loopback');
var path = require('path');
var app = module.exports = loopback();

// Create an instance of PassportConfigurator with the app instance
var PassportConfigurator = require('loopback-passport').PassportConfigurator;
var passportConfigurator = new PassportConfigurator(app);

app.boot(__dirname);

...

// Enable http session
app.use(loopback.session({ secret: 'keyboard cat' }));

// Load the provider configurations
var config = {};
try {
  config = require('./providers.json');
} catch(err) {
  console.error('Please configure your passport strategy in `providers.json`.');
  console.error('Copy `providers.json.template` to `providers.json` and replace the clientID/clientSecret values with your own.');
  process.exit(1);
}

// Initialize passport
passportConfigurator.init();

// Set up related models
passportConfigurator.setupModels({
  userModel: app.models.user,
  userIdentityModel: app.models.userIdentity,
  userCredentialModel: app.models.userCredential
});

// Configure passport strategies for third party auth providers
for(var s in config) {
  var c = config[s];
  c.session = c.session !== false;
  passportConfigurator.configureProvider(s, c);
}
```