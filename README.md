# azure-auth-client-alpha

A lightweight abstraction around adal.js. Use this to get identity and access tokens and information about the user using your application.

Please note that this is alpha quality software at the moment. I'm probably mostly done changing the API, though. The name of the node module will change to `azure-auth-client` rather than `azure-auth-client-alpha` when I feel comfortable doing that.

## Example

```js
import AuthClient from "azure-auth-client";

const config = {clientType: "ADAL",
                clientId: "ae33c32e-d2f2-4992-a4b2-51d03e7c8677",
                tenantId: "c834c34e-bbd3-4ea1-c2c2-51daeff91aa32",
                domain: "bar.com"};

const authClient = AuthClient.build(config)

const {ok, token, name, roles} = authClient.getIdentityToken();

const {ok, token, name, roles} = authClient.getAccessToken("foo");
```

## Installing

Using npm:

```bash
$ npm install azure-auth-client-alpha
```

Using yarn:

```bash
$ yarn add azure-auth-client-alpha
```

## Why?

It abstracts you away from adal.js, which you might not want to deal with directly because it's kind of gross. Note that right now, this library intentionally doesn't expose all of the functionality of adal.js - just the functionality for the stuff I care about at the moment. This library also decodes the tokens for you and pulls some potentially relevant identity information out of the token's claims.

Also, if you have a single-page application that interacts with Azure AD in production, you might not want it to do that during development or testing. This abstraction lets you use a dummy implementation that uses a pre-configured identity, identity token, and access tokens, so you don't have to integrate when you don't want to. You may or may not find this helpful.

## Fine, how?

### Making a config

You'll need to create an `Azure auth config` object.

Here's what a "real" `Azure auth config` object might look like:

```js
{clientType: "ADAL",
 clientId: "ae33c32e-d2f2-4992-a4b2-51d03e7c8677",
 tenantId: "c834c34e-bbd3-4ea1-c2c2-51daeff91aa32",
 domain: "bar.com"}
 ```

Setting the **clientType** to 'ADAL' means you'll use the real implementation that talks to Azure AD.

* **clientId** - The client ID of your application (either a single-page application or a traditional web application, depending on your situation) from Azure
* **tenantId** - The ID of your Azure tenant
* **domain** - The AD domain

You can also pass `tenantName` rather than `tenantId` if you feel like it, as well as override the default values of adal.js configuration properties that I chose.

If you don't feel like talking to Azure AD and want to pretend to be an arbitrary user for testing, here's what your `Azure auth config` would look like:

```js
{clientType: "DUMMY",
 identity: {name: "JONES, FRED",
            givenName: "FRED",
            familyName: "JONES",
            roles: ["Ninja", "Lawyer"]},
 identityDelay: 0}
```

Similarly, setting **clientType** to 'DUMMY' means you'll use the dummy implementation. Other properties that I'm too lazy to document right now control what will be returned when the application requests identity and access tokens and how long those requests will take.

It's possible you already have some kind of configuration object to hold your app's configuration - something that looks like this:

```js
{foo: "bar",
 baz: 42,
 apiConfig: {url: "localhost:3000/api"}}
 ```

If you do, just stick your `Azure auth config` object into your app config as `azureAuthConfig`, like this:

```js
{foo: "bar",
 baz: 42,
 apiConfig: {url: "localhost:3000/api"},
 azureAuthConfig: {clientType: "ADAL",
                   clientId: "ae33c32e-d2f2-4992-a4b2-51d03e7c8677",
                   tenantId: "c834c34e-bbd3-4ea1-c2c2-51daeff91aa32",
                   domain: "bar.com"}}
 ```

 If not, you can just use the standalone `Azure auth config` to create a client instance.

### Creating a client

Once you have your `Azure auth config` object - either as part of your `app config` object or by itself - add the following import where ever you do your configuring:

```js
import AuthClient from 'azure-auth-client'
```

Then, use your config to create an `AuthClient`:

```js
const authClient = AuthClient.build(config);
```

You can pass this guy around wherever you want. In a React or Redux app, injecting it via props or state seems like the best way to use it. Eventually, there should be examples of doing this somewhere.

### Getting tokens

Now you can use this auth client object anywhere to get an identity token or access tokens for different resources, like this:

```js
const response = authClient.getIdentityToken();
  => {ok: true, token: "eyJ0eXAiOi...", decodedToken: ...}

const response = authClient.getAccessToken("foo");
  => {ok: true, token: "eyJ0eXAiOi...", decodedToken: ...}
```

If we fail to get a token for some reason, the `ok` property of the response will be false, and there will probably be some information attached to the response.

### Using tokens

Assuming you didn't get redirected to log in and nothing broke, he response from these functions contains both the raw token and all the data contained inside of it, which means you don't have to decode it yourself or do any other manual parsing.

Here's an example of what you might get. Results will vary depending on how your application is registered in Azure Active Directory.

```js
{
  "ok": true,
  "token": "eyJ0eXAiOi...",
  "decodedToken": {
    "aud": "abc4e33e-e805-0992-a4b2-59f1ef7e8e7a",
    "iss": "https://foo.com/bar",
    "iat": 1518675421,
    "nbf": 1518675421,
    "exp": 1518679421,
    "aio": "XyZaew323grgerEWfewfWEewf4trs6455645eWE222f#fwefewfe",
    "amr": [
      "wia"
    ],
    "family_name": "JONES",
    "given_name": "FRED",
    "in_corp": "true",
    "ipaddr": "123.45.678.0",
    "name": "JONES, FRED",
    "nonce": "42af75a6-c56c-4bd4-bbg9-ba42d67ad7ea",
    "oid": "beef43456e6dc-54se-423e-bea3112eda57",
    "onprem_sid": "S-1-5-21-1231242546-6544342323-3252352364-23235324",
    "sub": "VewaeEFW-wewEW2-3220230aeawefw032a2a3332a32",
    "roles": ["Ninja", "Lawyer"],
    "tid": "ab23cdef-ad22-ab43-a32d-e56r12233aea",
    "unique_name": "fredjones@baz.com",
    "upn": "fredjones@baz.com",
    "uti": "awer2323aeweaQW323eawf",
    "ver": "1.0"
  },
  "name": "JONES, FRED",
  "familyName": "FRED",
  "givenName": "JONES"
  "roles": ["Ninja", "Lawyer"]
}
```

When you have an identity token for your own application's backend API or an access token for some other API, you can pass it in an `Authorization` header with an authentication scheme of `Bearer` to call the appropriate secured service:

```js
const tokenResponse = authClient.getAccessToken("abc4e33e-e805-0992-a4b2-59f1ef7e8e7a");

if(!tokenResponse.ok) {
  throw new Error("Failed to get access token.");
}

const headers = new Headers();

headers.append("Authorization", `Bearer ${tokenResponse.token}`);

const options = {method: 'GET', headers, mode: "cors"};

const barResponse = await fetch("https://foo.com/bar", options);
```
