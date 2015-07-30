Authentication for the Retail SDK
=================================

[![Build Status](https://travis-ci.org/djMax/paypal-retail-node.svg)](https://travis-ci.org/djMax/paypal-retail-node)

[TL;DR](http://www.urbandictionary.com/define.php?term=tl%3Bdr): You need to use your
[client ID and secret from developer.paypal.com](https://developer.paypal.com/developer/applications)
to get a token to call InitializeMerchant within the SDK. You can use our modules in your own server
(node.js express/Kraken, ruby, Java, .Net, ...), or use one of our prebuilt Docker containers or
Heroku builds. You can do this for free on Heroku and Amazon EC2, including SSL if you use something
like [CloudFlare](https://blog.cloudflare.com/introducing-universal-ssl/).

To get started, deploy to Heroku and then configure the environment variables in the Heroku dashboard.

[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy?template=https://github.com/djMax/paypal-retail-node.git)

<table>
<tr><th colspan="2">Required Environment Variables</th></tr>
<tr><td>APP_SECURE_IDENTIFIER</td><td>A value [you make up](https://identitysafe.norton.com/password-generator/)
 and keep secret to protect your PayPal refresh tokens.</td></tr>
<tr><th colspan="2">Other Environment Variables</th></tr>
<tr><td>SETUP_ENABLED</td><td>If set to true, you can go to /setup/live or /setup/sandbox to generate a token which you
can copy into your code. You should turn this off in a 'live' server.</td></tr>
<tr><td>ROOT_URL</td><td>The root URL of your new server, e.g. http://something.herokuapp.com. If not set, we will pull
it from the first request we see.</td></tr>
<tr><td>PAYPAL_LIVE_CLIENTID</td><td>The PayPal client id for your application in the live environment.</td></tr>
<tr><td>PAYPAL_LIVE_SECRET</td><td>The PayPal secret for your application in the live environment.</td></tr>
<tr><td>PAYPAL_SANDBOX_CLIENTID</td><td>The PayPal client id for your application in the sandbox environment</td></tr>
<tr><td>PAYPAL_SANDBOX_SECRET</td><td>The PayPal secret for your application in the sandbox environment</td></tr>
<tr><td>APP_REDIRECT_URL</td><td>For third party use, *after* PayPal authentication is complete the server will redirect to
this URL with the token for InitializeMerchant in the Retail SDK. This is usually some sort of custom URL protocol like 'myappname://paypaltoken' or similar. The URL needs to launch your app and this server will pass it the token.</td></tr>
</table>

Usually you're in one of these two situations:

* [First Party](#first-party) - The account I made the PayPal.com developer application with is the only one I ever want to transact with. You will use this server to get a token and then you will put that token in your app.
* [Third Party](#third-party) - I want other merchants to use my application with their own accounts and give me permission to transact for them. You will call this server from your app to get a URL for PayPal Login, then when login completes this server will call APP_REDIRECT_URL as described above.

Why can't I just put the id and secret in the app?
==================================================
PayPal uses [OAuth](http://en.wikipedia.org/wiki/OAuth) to authenticate an application's ability to transact on behalf of a merchant account.
OAuth has three main "sensitive quantities" - the access token, the refresh token and the application secret. The access token is short lived (typically 15 minutes-8 hours)
and useful only for a single account. A refresh token is long lived (by spec 10 years, in theory forever, in practice until the owner revokes it) and useful only
for a single account. The application secret is permanent for a given application and useful for potentially all accounts.

Generally, it's ok to store the access token on the end-user (e.g. mobile) device. You should still store it securely, not record it in logs, etc, but it's value is short lived.
The refresh token should not generally live on the end-user device because it takes away another choke point for you to disable the use of your application in case the device is
lost or stolen (because you could just "invalidate" the refresh token for that account and then that device, the next time it came to your server to refresh the token would not get a
new access token).

The application secret should never live on the end-user device because anyone who has that can infinitely refresh access tokens given refresh
tokens, and can get an access token for YOUR account any time they want. More importantly, anyone holding the application secret can pretend to be your
app and have an unsuspecting user give the rogue app an access and refresh token (by logging into PayPal and believing they are granting permission to your app).

So our recommended approach is to use our server modules or containers to create a server that stores your application secret and provides access tokens to your end user devices.
Our modules also employ an additional server-only password to return a "modified refresh token" to the end user device which means you don't need a database. When your application
needs a new access token, it calls the server module and provides the modified refresh token. The server module combines that with the application secret and calls the PayPal
servers to obtain a new access token which it delivers back to the client.

Note that if you needed to make calls OUTSIDE of your application (such as from your backend services), you would need to get the access token/refresh token flow working on your
backend services and would then need a database to store relevant user data. Doing so securely is outside of the scope of this document, but each server module readme describes
the API, and there's no reason you can't use the same services for your other backend services.

First Party
===========
You can go through the Login With PayPal flow and grant consent to your application just once. The token returned will include the
"modified refresh token" that can be baked into your application and shipped. The server modules and containers we provide expose an
endpoint for first party token creation (/setup) that can be disabled after you generate the original value. The point of the additional
identifier (the APP_SECURE_IDENTIFIER, which you can just make up) is in the case where you have an application compromise or significant bug, you can change
this identifier and applications with the old value will stop being able to transact on your behalf. Essentially this identifier is
a remote kill switch that you control.

Third Party
===========
Within this scheme there is another big question - do you have your own account system or do you want to use PayPal for
all authentication? If you have your own account system, it's very likely you want to use our server modules instead of
the standalone containers. In that case you would first authenticate the API caller against your own system, and verify
the link between that account and the PayPal account before obtaining access tokens on their behalf. This provides an
additional layer of security and control for end user applications making calls for your merchants via your app.

Note: You MUST set the redirect URL of your app on developer.paypal.com to match the server to which you deploy the paypal-retail-node code with the path /returnFromPayPal appended. For example, if you've deployed to http://pph-retail-node.herokuapp.com, the return URL MUST be set to http://pph-retail-node.herokuapp.com/returnFromPayPal
