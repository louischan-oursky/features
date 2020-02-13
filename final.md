# Auth UI

## Background

Auth UI is a standalone component that aims to be a OpenID Provider. It is a traditional web application that does **NOT** share the session with Auth Gear. However, it provides access to most of the Auth Gear features.

## Domain

Auth UI is available for both default domain and custom domain.

### Default domain

If the app is using default domain, then Auth UI is served at `https://accounts.<appname>.skygearapp.com`. Note that this domain format may conflict with deployment version. Auth UI has higher precedence than deployment version. The choice of `accounts` is borrowed from some other internet services.

### Custom domain

If the app is using custom domain, then Auth UI is served at the developer's wish.

### Within the same domain of the app

Auth UI cannot be served in the same domain with the app. So Auth UI is not available at `https://<appname>.skygearapp.com/authorize` nor `https://www.myapp.com`.

### Domain Example

|   |Default domain|Auth UI default domain|www.myapp.com|accounts.myapp.com|
|---|---|---|---|---|
|App|`https://myapp.skygearapp.com`|N/A|`https://www.myapp.com`|N/A|
|Auth Gear|`https://myapp.skygearapp.com/_auth/`|N/A|`https://www.myapp.com/_auth/`|N/A|
|Auth UI|N/A|`https://accounts.myapp.skygearapp.com`|N/A|`https://accounts.myapp.com`|

## API client

API client is now treated as OAuth application. Existing API clients are treated as OAuth applications with the largest scopes. For example, using an existing access token can access the session management API of Auth Gear, to list sessions of all API clients.

## OAuth Authorization

Auth UI does not OAuth authorization at the moment. All access tokens it issue behave in the same way as existing access tokens. That is, the tokens are issued with the largest scopes.

## Auth UI session

Auth UI has its own session management. Auth UI session is stateful and is stored in HTTP cookie. The session management API of Auth Gear cannot list, get or delete Auth UI session. Auth UI session and Auth Gear session are not interchangeable.

## Session compatibility with Auth Gear and the SDK

In handling the Authentication Request in the Authorization Endpoint, Auth UI creates a session and store it in cookie. Finally it redirects the user agent back to the app with a authorization code.

In a typical Authorization Code Flow, the app should send a Token Request to Auth UI to obtain a Token Response. However, Token Response is compatible with Auth Gear and the SDK in which the session transport can either be cookie or header.

To mitigate this compatibility problem, Auth Gear, which shares the same domain with microservices, has an endpoint to proxy the Token Request to Auth UI and transform the Token Response to a compatible response.

Below is a flow chart demonstrating the flow.

```
app                                                     Auth UI                                       Auth Gear
 +                                                         +                                              +
 |               Send Authentication Request               |                                              |
 +--------------------------------------------------------->                                              |
 |                                                         |                                              |
 |                           +-----------------------------+                                              |
 |                           |    Authenticate the user    |                                              |
 |                           +----------------------------->                                              |
 |                                                         |                                              |
 |                                                         |                                              |
 |                      Write cookie                       |                                              |
 |     Redirect to the app with Authentication Response    |                                              |
 <---------------------------------------------------------+                                              |
 |                                                         |               Send Token Request             |
 +------------------------------------------------------------------------------------------------------->+
 |                                                         |                                              |
 |                                                         |             Forward Token Request            |
 |                                                         <----------------------------------------------+
 |                                                         |                                              |
 |                                                         |             Return Token Response            |
 |                                                         +---------------------------------------------->
 |                                                         |                                              |
 |                                                         |            Transform Token Response          |
 |                                                         |              Write cookie or body            |
 <--------------------------------------------------------------------------------------------------------+
 |                                                         |                                              |
 |                                                         |                                              |
 |                                                         |                                              |
 |                                                         |                                              |
 |                                                         |                                              |
 |                                                         |                                              |
 +                                                         +                                              +
```

This flow allows both Auth UI and Auth Gear to write host-only cookie.

## Logout

Since Auth UI and Auth Gear do not share session, the user logs out with the existing Auth Gear API. Auth UI does **NOT** provide the Logout Endpoint.

OpenID Connect does have 3 draft specifications related to logout. They are

- [Session Management](https://openid.net/specs/openid-connect-session-1_0.html)
- [Front-Channel Logout](https://openid.net/specs/openid-connect-frontchannel-1_0.html)
- [Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html)

These specs is useful for implementing Single logout.
