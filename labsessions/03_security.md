TODO ENABLE BINARY IN FILTER TO SEE GOOGLE RESPONSE

# Security properties of OAuth 2.0 flows

Many OAuth 2.0 tutorials advise the use of the *Implicit Grant* flow, also known as the *two-legged* flow. This flow is straightforward, and seems a lot easier to implement than the *Authorization Code* flow, also known as the *three legged* flow. 

In this lab session, we dissect both *Implicit Grant* flow and the *Authorization Code* flow, and show you which security properties each of these flows has to offer.

Additionally, we will discover how it all comes down to the *access token*, which plays a crucial role in every OAuth 2.0 flow. This token is known as a *bearer token*, which essentially means that posession of the token is sufficient to gain access to a protected resource. 


## One access token to rule them all

The first scenario illustrates the power of the *access token*. We will steal the token from the *Implicit Grant* flow in the *Rock & Roll* application, and use it to access the protected resources directly.

### Stealing the token

Stealing the token can be done in various ways. Think about eavesdropping on the network, an XSS attack, a vulnerable or malicious browser extension, ...

To avoid complicating this workshop even further, we're simply going to grab the *access token* from the network requests we already logged in *Burp*. Follow the steps below to get hold of an *access token*:

2. In *Burp*'s main window, select the *Proxy* tab
2. Here, you will see a list of requests that *Burp* has intercepted
3. Look for a request to the `/tokens/validate` endpoint
4. Grab the *access token* from the request body and store it in a file for later use

The screenshot below shows which requests holds the *access token*, and where you can find it.

TODO SCREENSHOT


### Using the stolen access token

The access token we stole was issued explicitly for the *Rock & Roll* application. Unfortunately, there is no way to restrict its use in an *Implicit Grant* flow. We have created a simple *Token Inspector* application, that allows you to simulate the abuse of an *access token*.

The application is hosted on GitHub pages, at the following url: [https://philippederyck.github.io/emberconf2017-tokeninspector](https://philippederyck.github.io/emberconf2017-tokeninspector). If everything works correctly, you should see something like the screenshot below.

TODO SCREENSHOT

You can give the application an *access token* for one of the supported providers. The application will contact the APIs and retrieve some information with this *access token*. If you're worried about the security of your account information, you'll be pleased to hear the everything runs on the client-side, and tokens are sent nowhere else but to their corresponding providers.

So, let's grab the *access token* you saved to a file earlier, and see what the *Token Inspector* can find. 

TODO ADD EXPLICIT INSTRUCTIONS WHEN THE APP IS BUILT

In the first section, you can see a few examples of the kind of information that can be retrieved using the access token. As you can see, our *Rock & Roll* application requested access to a lot of resources (which it does not need), which becomes a real problem if the token ever gets stolen. TODO MAKE SURE WE OVERREACH IN RARWE

TODO SCREENSHOT

In the second section, you can find some metadata about the token. Note that this information is not provided by default. An explicit call to the issuer is needed to retreive this information.

One of the most important pieces of information in the metadata is the identity of the client to which the token belongs. As you can see, the *client ID* corresponds to that of the *Rock & Roll* application, yet our *Token Inspector* application has no problems using the same token to retrieve information from the protected resource.

TODO SCREENSHOT

### Explicitly checking the client ID

TODO CONTENT (show how the backend checks this, and explain why this is so important)

### Restricting the access token

From this scenario, you can clearly conclude that the *access token* is a bearer token, and that anyone holding it can use it to access the protected resources.

That is exactly one of the reasons why the scope of a requested *access token* should be as limited as possible. Modify your implementation of the *Ipmlicit Grant* flow to minimize the scope to what we need (i.e. *email*).

Now that you have modified your implementation, repeat the scenario from before. Take the *access token* and give it to the *Token Inspector*. If everything went well, you should see a lot less personal information in the first section, and the modified scope in the second section.

TODO SCREENSHOT

## Access tokens vs authorization codes

In the *Authorization Code* flow, the client receives an *authorization code* instead of an *access token*. The backend application can exchange the *authorization code* for an *access token*, which it in turn can use to access the protected resources.

Let's investigate the security properties of such an *authorization code*. Just like before, we're going to grab an *authorization code* from the logged requests in *Burp*, following the steps below:

2. In *Burp*'s main window, select the *Proxy* tab
2. Here, you will see a list of requests that *Burp* has intercepted
3. Look for a request to the `/tokens/validate` endpoint (TODO CHECK)
4. Grab the *authorization code* from the request body 

Now, head back to the *Token Inspector* application, and see if we can get an *access token* with this *authorization code*. Fill out the *authorization code* field, but leave the other fields blank.

TODO SCREENSHOT

Here, you already see one major difference compared to the *access token*. To effectively use an *authorization code* you not only need to specify a *client ID* (which is public information), but also the *client secret*, which only the backend application knows.

Use the information from the *Rock & Roll* application to complete both fields, and re-inspect the results.

TODO SCREENSHWOT

Even though we have all information, including the secret values that we normally would not have access to, we still do not get an *access token*. The reason for this error is that an *authorization code* is invalidated after first use.

### Exchanging an authorization code for an access token

Let's see what happens if we manage to steal a fresh *authorization code*. Would we be able to get an *access token* with it? 

For this, we will use *Burp* to intercept requests, and stop the flow once we have obtained an *authorization code*. Follow the steps below to capture a fresh *authorization code*:

1. Open the *Rock & Roll* application and make sure you're logged out
1. In *Burp*'s *Proxy* tab, choose for *TODO* and make sure interception is turned on (see screenshot below)
2. Start the *Authorization Code* flow with on of the supported providers
3. *Burp* will have intercepted a request, so open *Burp* and click the *Forward* button.
4. Keep going through the flow and forwarding requests, until you see the request containing the *authorization code*
5. **Do not forward this request**, but simply copy the *authorization code*. Leave the window untouched, we will come back to this later.

TODO SCREENSHOT INTERCEPT

With the freshly obtained *authorization code*, go back to the *Token Inspector*, and enter it into the inspection field. Leave the other fields blank for now. 

As you can see, exchanging the *authorization code* for an *access token* is definitely not possible without providing the *client ID* and *client secret*. Fill out these values, and try again.

TODO SCREENSHOT

Now, we have actually obtained a valid *access token*, but we needed confidential information to get this far. Use the *access token* in the *Token Inspector* application to actually retrieve some information about the user.

Now, go back to the intercepted request in *Burp*, and hit the *Forward* button. The *authorization code* will not be sent to the backend, which will try to exchange this for an *access code*, which will result in an error, since have already used this *authorization code* in the previous steps.

TODO SCREENSHOT

