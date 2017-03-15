# Choose your flows wisely

TODO INTRO (2 legged, 3 legged is better)

The *access token* plays a crucial role in every OAuth 2.0 flow. This token is known as a *bearer token*, which essentially means that posession of the token is sufficient to gain access to a protected resource. 

In this lab session, we dissect the *Implicit Grant* flow, and the *Authorization Code* flow. Additionally, we will investigate the security properties of *access tokens*.


## One access token to rule them all

The first scenario illustrates the power of the *accesstoken*. We will steal the token from the *Rock & Roll* application, and use it to access the protected resources directly.

### Stealing the token

Stealing the token can be done in various ways. Think about eavesdropping on the network, an XSS attack, a vulnerable or malicious browser extension, ...

To avoid complicating this workshop even further, we're simply going to grab the *access token* from the network requests in the developer tools. Follow the steps below in *Chrome* to get hold of an *access token*:

1. Make sure you're logged out of the *Rock & Roll* application
2. Open *Chrome*'s developer tools, and select the *Network* tab
2. Login to the *Rock & Roll* application using an *Implicit Grant* flow (either with *Facebook* or *Google*)
3. Inspect the network requests that are being logged. You should see a request to the `/tokens/validate` endpoint
4. Grab the *access token* from the request and store it in a file for later use

The screenshot below shows which requests holds the *access token*, and where you can find it.

TODO SCREENSHOT


### Using the stolen access token

The access token we stole was issued explicitly for the *Rock & Roll* application. Unfortunately, there is no way to restrict its use in an *Implicit Grant* flow. We have created a simple *Token Inspector* application, that allows you to simulate the abuse of an *access token*.

The application is hosted on GitHub pages, at the following url: [https://philippederyck.github.io/emberconf2017-tokeninspector](https://philippederyck.github.io/emberconf2017-tokeninspector). If everything works correctly, you should see something like the screenshot below.

TODO SCREENSHOT

You can give the application an *access token* for one of the supported providers. The application will contact the APIs and retrieve some information with this *access token*. If you're worried about security, you'll be pleased to hear the everything runs on the client-side, and tokens are only sent nowhere else but to their corresponding providers.

So, let's grab the token you saved to a file earlier, and see what the *Token Inspector* can find. 

TODO ADD EXPLICIT INSTRUCTIONS WHEN THE APP IS BUILT

TODO THINGS WE SHOULD COVER HERE

* The app for which the token was issued
* User info
* ...