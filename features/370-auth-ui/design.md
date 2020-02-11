# Auth UI

Table of Content

* [Background](#background)
* [Where Auth UI is served](#where-auth-ui-is-served)
   * [Reason 1: Individual gears should be unaware of Custom Domain](#reason-1-individual-gears-should-be-unaware-of-custom-domain)
   * [Reason 2: Cookies written by Auth Gear is host-only](#reason-2-cookies-written-by-auth-gear-is-host-only)
   * [The actual location of Auth UI](#the-actual-location-of-auth-ui)
* [API key as client_id](#api-key-as-client_id)
* [Page templates](#page-templates)
   * [Customization of page templates](#customization-of-page-templates)
      * [Customizing the theme with CSS](#customizing-the-theme-with-css)
      * [Customizing with own templates](#customizing-with-own-templates)
* [The authorize endpoint](#the-authorize-endpoint)
   * [The authorization page](#the-authorization-page)
   * [The signup page](#the-signup-page)
   * [The signup password page](#the-signup-password-page)
* [The settings endpoint](#the-settings-endpoint)
* [The verification endpoint](#the-verification-endpoint)
* [The logout endpoint](#the-logout-endpoint)
* [Omitted features](#omitted-features)
* [Future improvements](#future-improvements)
   * [Customizing the signup flow or the authentication flow](#customizing-the-signup-flow-or-the-authentication-flow)
* [Support Matrix](#support-matrix)
* [Being a OpenID Provider](#being-a-openid-provider)
   * [Authorization Endpoint](#authorization-endpoint)
   * [Token Endpoint](#token-endpoint)
   * [Token Response](#token-response)
* [Security considerations](#security-considerations)
   * [Being embedded in an HTML iframe](#being-embedded-in-an-html-iframe)
* [Appendix](#appendix)

## Background

Auth UI is the builtin web frontend of Auth Gear.
It provides access to most Auth Gear features as a server side rendered web application.
It is part of Auth Gear.

It is intended to be a conforming OpenID Provider implementation of Auth Gear in the future.
As of the time of writing, the conformance of OpenID Provider is not a goal.
However, the implementation must be prepared for conformance in the future.

## Where Auth UI is served

It is served in the same domain as Auth Gear and Microservices.

### Reason 1: Individual gears should be unaware of Custom Domain

Auth UI is going to be responsible for user verification. The verification email includes an URL to Auth UI. This implies Auth Gear has to know where Auth UI is served. When Custom Domain was introduced, it is a feature solely handled by Gateway. Individual gears such as Auth Gear are unaware of Custom Domain. Serving Auth UI in a different domain will break this property. If Auth UI is served in the same domain, it is trivial for Auth Gear to know where it is.

### Reason 2: Cookies written by Auth Gear is host-only

A host-only cookie only matches the domain by simple string comparison. While a non host-only cookie matches the domain via the [domain-matches](https://tools.ietf.org/html/rfc6265#section-5.1.3) algorithm. To allow Auth UI to be embedded within an iframe in a Skygear App, the session and its cookie must be shared by both. This means the domain has to be either exact match or domain-matches.

If cookies written by Auth Gear have to be not host-only, then the cookie attribute `Domain` must be specified. If the custom domain is `www.example.com`, then the value of the attribute should be the apex domain `example.com`. So Auth Gear cannot avoid being aware of Custom Domain.

### The actual location of Auth UI

Auth UI is served at the path prefix `/_auth/_/`

Some example endpoints:

- `/_auth/_/authorize`
- `/_auth/_/signup`
- `/_auth/_/forgot_password`
- `/_auth/_/reset_password`
- `/_auth/_/verify`

As a side note, Universal Login of Auth0 is served at `https://<app>.auth0.com/u/{login,signup,...}`

## API key as client_id

All pages require the query parameter `client_id`, Auth UI uses the provided API key to initiate necessary requests to Auth Gear.

## Page templates

All pages served by Auth UI are rendered with Go standard library package [html/template](https://golang.org/pkg/html/template/).
This is also the package we use to render HTML email template. So the developer only needs to learn the syntax once.

The content of the template is mainly HTML with some CSS.
It also contains some JavaScript to add some extra functionalities such as dynamically checking password requirements and toggling of password visibility.
All pages must be function properly even without JavaScript.

### Customization of page templates

The developer can customize the pages at 2 levels.
They can keep using the builtin templates and customize the theme.
Or they can replace the builtin ones with their own ones.

#### Customizing the theme with CSS

By studying the UI design of Auth UI, a few class names can generalized:

- `.primary-txt`: The primary text.
- `.secondary-txt`: The secondary text.
- `.primary-btn`: The primary button.
- `.secondary-btn`: The secondary button.
- `.anchor`: The anchor.
- `.text-input`: The text input.

There are a few CSS pseudo classes that are applicable to the UI elements.

- `:link`: Unvisited links.
- `:visited`: Visited links.
- `:hover`: Mouse hover state.
- `:active`: Button being clicked.

Buttons also have two variants.
The primary button has background color while the secondary one has border.

So the following CSS properties should be able to customize.

- `color`
- `background-color`
- `border*`

To customize all combinations, the developer can provide the CSS directly.
The developer is strongly recommended to write CSS covering only
the above mentioned classes, pseudo classes and properties.
There is no validation to enforce this restriction.

#### Customizing with own templates

The developer can also provide their own templates via [the existing template mechanism](../251-templates/index.md).

## The authorize endpoint

The authorize endpoint `/_auth/_/authorize` allows the user to authenticate with configured authentication methods. It handles MFA challenge during authentication. If the user does not have an account yet, the user can sign up directly.

### The authorization page

The authorization page is the landing page of the authorize endpoint. It lists out the configured IdPs. It shows a text field for login ID. The login ID field is either a plain text input or a phone number input, depending on the type of the first login ID key.

```
|---------------------------|
| Login with Google         |
|---------------------------|
| Login with Facebook       |
|---------------------------|

              Or

|--------------------------------------|  |----------|
| Enter in your email or username here |  | Continue |
|--------------------------------------|  |----------|

Login with a phone number instead.
```

If the user selects an IdP, the user is redirected to the authorization endpoint of that IdP. If the user continues with a login ID, the user is directed to the password page.

Separating the login ID and the password into two pages allow future improvements, such as [Open ID Provider Issuer Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html#IssuerDiscovery). For example, the user can type in their Google account name and then be redirected to the authorization endpoint of Google.

### The signup page

The signup page displays the first login ID key default. Other login ID keys are available to choose.

```
Sign up with email
|--------------------------|  |----------|
| Enter in your email here |  | Continue |
|--------------------------|  |----------|

Sign up with phone instead.
Sign up with username instead.
```

### The signup password page

The signup password page displays a visibility toggleable password field with password requirements.

```
|------------------------|  |----------|
| Create a password here |  | Continue |
|------------------------|  |----------|

- [ ] At least one digit
- [ ] At least one uppercase English character
- [ ] At least one lowercase English character
- [ ] At least one symbols ~`!@#$%^&*()-_=+[{]}\|;:'",<.>/?
- [ ] At least 8 characters long
```

## The settings endpoint

The settings endpoint `/_auth/_/settings` allows the user to manage their account. Features include but not limited to:

- Update metadata
- Change password
- Configure MFA
- Add / Update / remove login ID
- Manage identities
- Manage sessions

## The verification endpoint

The verification endpoint `/_auth/_/verify` allows the user to verify their login ID. This page requires authenticated user. Unauthenticated access is redirected to the authorization page. Verification is not enforced by the authentication flow. For example, if auto send verification email is enabled, the email is sent behind the scene and the user is redirected to the application immediately after authentication success.

## The logout endpoint

The logout endpoint `/_auth/_/logout` allows the user to logout. The page has a logout button. Note that native application must clear the cookie storage manually because it is impossible to set cookie via WebView on the native HTTP client.

## Omitted features

Auth UI does not support authenticate with Custom Token.

## Future improvements

### Customizing the signup flow or the authentication flow

Some customization the developer may want to make

- Add inline extra fields in the signup page to fill in metadata
- Add custom pages before the signup page to fill in metadata
- Redirect to an external form page before the signup page and continue to the signup page with metadata
- Add custom pages after authentication, and finally redirect the user back to the application.
- Redirect to an external page after authentication, the external page redirects back to Auth UI, and finally redirect the user back to the application.

Introducing the concept of [authentication flow](https://www.keycloak.org/docs/6.0/server_admin/#_authentication-flows) may fulfill some of the above use cases.

## Support Matrix

Depending on the configuration of session transport, some features may be unavailable

|   |Authorize|Other features|
|---|---|---|
|Cookie|Yes|Yes|
|Header|Yes|No|

The settings and the logout endpoint requires a valid cookie session. If the application is using header transport, it can only use the authorize endpoint.

## Being a OpenID Provider

Some of the terms used in this session are defined in [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html#Terminology).

Auth UI implements the [Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) with [PKCE](https://tools.ietf.org/html/rfc7636)

### Authorization Endpoint

The following sessions list out the request parameters and their difference from the spec.

#### scope

The value must be `openid`.

#### response_type

The value must be `code` because the implementation only supports Authorization Code Flow.

#### client_id

The value must be a valid Skygear API Key.

#### redirect_uri

No difference from the spec.

#### state

It is ignored.

#### nonce

It is ignored.

#### display

It is not supported. The only supported display mode is `page`. Note that this parameter corresponds to UXMode in the SSO feature. In SSO, UXMode can be `redirect` or `popup`, which are equivalent to `page` and `popup` in the OIDC Core spec.

#### prompt

It is not supported.

#### max_age

It is not supported.

#### ui_locales

No difference from the spec.

#### id_token_hint

It is not supported.

#### login_hint

It is not supported.

#### acr_values

It is not supported.

---

#### code_challenge_method

Only `S256` is supported. `plain` is not supported. The reason is the existing implementation of SSO is using `S256` already. Since it defaults to `plain`, it is required and the value must be `S256`.

#### code_challenge

No difference from the spec.

#### Example of valid Authorization Request

`response_type=code&scope=openid&client_id=API_KEY&redirect_uri=REDIRECT_URI&code_challenge_method=S256&code_challenge=CODE_CHALLENGE&ui_locales=en`

### Token Endpoint

The following sessions list out the request parameters and their difference from the spec.

#### grant_type

The value must be `authorization_code` because the implementation only supports Authorization Code Flow.

#### code

No difference from the spec.

#### redirect_uri

No difference from the spec.

#### client_id

It is always required. The value must be a valid Skygear API Key.

#### code_verifier

No difference from the spec.

#### Example of valid Token Request

`grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=REDIRECT_URI&client_id=API_KEY&code_verifier=CODE_VERIFIER`

### Token Response

Since we have to support both cookie and header transport, the difference here is very large.

#### id_token

It is always absent. It is the JWT token that we may want to implement in the future.

#### token_type

If the API key is header transport, its value is `bearer`. Otherwise this key is absent.

#### access_token

If the API key is header transport, its value is the Skygear access token. Otherwise it is absent and the access token is set in HTTP cookie header instead.

#### expires_in

It is always absent.

#### refresh_token

If the API key is header transport, its value is the Skygear refresh token. Otherwise it is absent.

#### scope

It is always absent.

#### Example of Token Response with header transport

```json
{
  "token_type": "bearer",
  "access_token": "access.token",
  "refresh_token": "refresh.token"
}
```

#### Example of Token Response with cookie transport

```json
{}
```

## Security considerations

### Being embedded in an HTML iframe

The HTTP header `Content-Security-Policy` has a directive `frame-ancestors` to specify which parent may embed a page with `iframe`.

All pages served by Auth UI have `Content-Security-Policy: frame-ancestors 'self'` by default. The developer does not need to configure this if the embedding page is of the same Skygear App. The developer can configure the value of this header to whitelist additional domains or add any another directives.

## Appendix

- [Implementation Details](./implementation-details.md)
- [SDK](./sdk.md)
- [Configuration](./configuration.md)
