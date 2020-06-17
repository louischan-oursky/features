# Authgear

## User

A user has many identities. A user has many authenticators.

## Identity

An identity is used to look up a user.

3 types of identity are supported.

- Login ID
- OAuth
- Anonymous

### Identity Claims

The information of an identity are mapped to [Standard Claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims)

Currently, only `email` is mapped.

The claims are used to detect duplicate identity. For example, an Email Login ID and the email claim of an OAuth Identity. This prevents duplicate user when the user forgets the original authentication method.

### OAuth Identity

OAuth identity is external identity from supported OAuth 2 IdPs. Only authorization code flow is supported. If the provider supports OIDC, OIDC is preferred over provider-specific OAuth 2 protocol.

#### OIDC IdPs

The following IdPs are integrated with OIDC:

- Google
- Apple
- Azure AD

#### OAuth 2 IdPs

The following IdPs does not support OIDC. The integration is provider-specific.

- LinkedIn
- Facebook

### Anonymous Identity

A user either has no anonymous identity, or have exactly one anonymous identity. A user with anonymous identity is considered as anonymous user.

Anonymous identity has the following fields:

- Public Key: It is represented as a JWK and stored in the database.
- Private Key: It is kept privately and securely in the device storage.
- Key ID: A unique random string for efficient lookup.

From the user point of view, they do not perform any explicit authentication. Therefore

- Anonymous user cannot have secondary authenticators
- Anonymous user cannot access the settings page

#### Anonymous Identity JWT

The server verifies the validity of the key-pair by verify a JWT. A challenge is requested by client on demand, it is one-time use and short-lived. The JWT is provided in the [login_hint](#login-hint)

#### Anonymous Identity JWT headers

- `typ`: Must be the string `vnd.skygear.auth.anonymous-request`.

#### Anonymous Identity JWT payload

- `challenge`: The challenge returned by the server.
- `action`: either `auth` or `promote`

### Anonymous Identity Promotion

Anonymous user can be promoted to normal user by adding a new identity. When an anonymous user is promoted:

- A new non-anonymous identity is added.
- The anonymous identity is deleted.
- A new session is created.

The promotion flow is the same as the normal OIDC authorization code flow.

### Login ID Identity

A login ID has the following attributes:

- Key
- Type
- Normalized value
- Original value
- Unique key

A user can have many login IDs. For example, a user can have both an email and a phone number as their login IDs.

### Login ID Key

Login ID key is symbolic name assigned by the developer.

### Login ID Type

Login ID type determines the validation, normalization and unique key generation rules.

#### Email Login ID

##### Validation of Email Login ID

- [RFC5322](https://tools.ietf.org/html/rfc5322#section-3.4.1)
- Disallow `+` sign in the local part (Configurable, default OFF)

##### Normalization of Email Login ID

- Case fold the domain part
- Case fold the local part (Configurable, default ON)
- Perform NFKC on the local part
- Remove all `.` signs in the local part (Configurable, default OFF)

##### Unique key generation of Email Login ID

- Encode the domain part of normalized value to punycode (IDNA 2008)

#### Username Login ID

##### Validation of Username Login ID

- Disallow confusing homoglyphs
- Validate against PRECIS IdentifierClass profile
- Disallow builtin reserved usernames (Configurable, default ON)
- Disallow developer-provided reserved usernames (Configurable, default empty list)
- Check ASCII Only (`a-zA-Z0-9_-.`) (Configurable, default ON)

##### Normalization of Username Login ID

- Case fold (Configurable, default ON)
- Perform NFKC

##### Unique key generation of Username Login ID

The unique key is the normalized value.

#### Phone Login ID

##### Validation of Phone Login ID

- Check E.164 format

##### Normalization of Phone Login ID

Since well-formed phone login ID is in E.164 format, the normalized value is the original value.

##### Unique key generation of Phone Login ID

The unique key is the normalized value.

#### Raw Login ID

Raw login ID does not any validation or normalization. The unique key is the same as the original value. Most of the use case of login ID should be covered by the above login ID types.

### Optional Login ID Key during authentication

The login ID provided by the user is normalized against the configured set of login ID keys. If exact one identity is found, the user is identified. Otherwise the login ID is ambiguous. Under default configuration, Email, Phone and Username login ID are disjoined sets so no ambiguity will occur. (Email must contain `@`; Username does not contain `@` or `+`; Phone must contain `+` and does not contain `@`)

### The purpose of unique key

If the domain part of a Email login ID is internationalized, there is 2 ways to represent the login ID, either in Unicode or punycode-encoded. To ensure the same logical Email login ID always refer to the same user, unique key is generated.

## Authenticator

Authgear supports various types of authenticator. Authenticator can be primary, secondary or both.

Authenticators has priorities. The first authenticator is the default authenticator in the UI.

### Primary Authenticator

Primary authenticator authenticates the identity. Each identity has specific applicable primary authenticators. For example, OAuth Identity does not have any applicable primary authenticators.

### Secondary Authenticator

Secondary authenticators are additional authentication methods to ensure higher degree of confidence in authenticity.

### Password Authenticator

Password authenticator is a primary authenticator. Every user has at most 1 password authenticator.

### TOTP Authenticator

TOTP authenticator is either primary or secondary.

TOTP authenticator is specified in [RFC6238](https://tools.ietf.org/html/rfc6238) and [RFC4226](https://tools.ietf.org/html/rfc4226).

In order to be compatible with existing authenticator applications like Google Authenticator, the following parameters are chosen:

- The algorithm is always HMAC-SHA1.
- The code is always 6-digit long.
- The valid period of a code is always 30 seconds.

To deal with clock skew, the code generated before or after the current time are also accepted.

### OOB-OTP Authenticator

Out-of-band One-time-password authenticator is either primary or secondary.

OOB-OTP authenticator is bound to a verified recipient. The recipient can be a verified email address or a verified phone number that can receive SMS messages.

The OTP is 4-digit long.

### Bearer Token Authenticator

Bearer token authenticator is secondary.

A bearer token is generated during MFA when the user opt to skip MFA next time.

The token is a cryptographically secure random string of 256 bits (32 bytes) in hex encoding.

### Recovery Code Authenticator

Recovery code authenticator is secondary.

Recovery codes are generated when the user adds a secondary authenticator first time.

The codes is cryptographically secure random 10-letter string in Crockford's Base32 alphabet.

## Authentication

- The developer can configure enabled identity types. By default, all supported identity types are enabled.
- The developer can configure enabled primary authenticators. By default, Password Authenticator is enabled.
- The developer can configure enabled secondary authenticators. By default, TOTP, OOB-OTP and Bearer Token Authenticator are enabled.
- The developer can configure whether secondary authentication is necessary.
  - `required`: secondary authentication is required. Every user must have at least one secondary authenticator.
  - `if_exists`: secondary authentication is opt-in. If the user has at least one secondary authenticator, then the user must perform secondary authentication.
  - `if_requested`: secondary authentication is purely optional even the user has at least one secondary authenticator.

## Interaction

Manipulation of user, identities and authenticators are driven by interaction. An interaction starts with an intent and has various steps. When all required steps have been gone through, the interaction is committed to the database.

### Login intent

The login intent authenticate existing user. It involves the following steps:

- Select identity
- Authenticate with primary authenticator
- Authenticate with secondary authenticator / Setup secondary authenticator

For example,

Login with login ID and password

- Select identity by providing a login ID
- Authenticate with password

### Signup intent

The signup intent creates a new user. It involves the following steps:

- Create identity
- Setup primary authenticator
- Setup secondary authenticator

For example,

Login in with Google

- Create identity by perform OIDC authorization code flow with Google
- No primary authenticator is required

### Add Identity intent

The add identity intent adds a new identity to a user. It involves the following steps:

- Create identity
- Setup primary authenticator

For example,

Add Email login ID to a user with 1 OAuth Identity

- The user provides the email address
- Setup OOB-OTP authenticator of the given email address

## OIDC

Authgear acts as OpenID Provider (OP).

### OAuth 2 and OIDC Conformance

Only [Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) with [PKCE](https://tools.ietf.org/html/rfc7636) is implemented.

### Client Metadata

#### Standard Client Metadata

Supported [standard client metadata](https://openid.net/specs/openid-connect-registration-1_0.html#ClientMetadata) are as follows:

- `redirect_uris`
- `grant_types`
- `response_types`

#### Custom Client Metadata

- `client_id`: OIDC client ID.
- `access_token_lifetime`: Access token lifetime in seconds, default to 1800.
- `refresh_token_lifetime`: Refresh token lifetime in seconds, default to max(access_token_lifetime, 86400). It must be greater than or equal to `access_token_lifetime`.

##### Generic RP Client Metadata example

```yaml
redirect_uris:
- "https://appbackend.com"
grant_types:
- authorization_code
- refresh_token
response_types:
- code
```

Standard configuration for authorization code flow to work properly.

##### Native application Client Metadata example

```yaml
redirect_uris:
- "com.myapp://host/path"
grant_types:
- authorization_code
- refresh_token
response_types:
- code
```

Standard configuration for authorization code flow to work properly. Note that the redirect URI is of custom scheme.

##### Web application sharing the same root domain Client Metadata example

```yaml
redirect_uris:
- "https://www.myapp.com"
grant_types: []
response_types:
- "none"
```

The web application shares the cookies so authorization code is unnecessary.

##### Silent Authentication Client Metadata example

```yaml
redirect_uris:
- "https://client-app-endpoint.com"
grant_types:
- "authorization_code"
response_types:
- "code"
```

Refresh token is not used.

### Authentication Request

#### scope

- `openid`: It is required by the OIDC spec
- `offline_access`: It is required to issue refresh token.

#### response_type

- `code`: Authorization Code Flow
- `none`: [Authgear acting as authentication server with web application](#authgear-acting-as-authentication-server-with-web-application)

#### display

The only supported value is `page`.

#### prompt

The following values are supported.

- `login`
- `none`

#### max_age

Unsupported.

#### id_token_hint

No difference from the spec, for `prompt=none` case.

#### login_hint

Developer can optionally pre-select the identity to use using `login_hint` parameter. `login_hint` should be a URL of form `https://auth.skygear.io/login_hint?<query params>`.

The following are recognized query parameters:
- `type`: Identity type
- `user_id`: User ID
- `email`: Email claim of the user
- `oauth_provider`: OAuth provider ID
- `oauth_sub`: Subject ID of OAuth provider
- `jwt`: JWT object

For examples:
- To login with email `user@example.com`:
    `https://auth.skygear.io/login_hint?type=login_id&email=user%40example.com`
- To login with Google OAuth provider:
    `https://auth.skygear.io/login_hint?oauth_provider=google`
- To signup/login as anonymous user:
    `https://auth.skygear.io/login_hint?type=anonymous&jwt=...`

The UI tries to match an appropriate identity according to the provided parameters. If exactly one identity is matched, the identity is selected. Otherwise `login_hint` is ignored.

Unknown parameters are ignored, and invalid parameters are rejected. However, if user is already logged in and the provided hint is not valid for the current user, it will be ignored instead.

#### acr_values

Unsupported yet.

#### code_challenge_method

Only `S256` is supported. `plain` is not supported.

### Token Request

#### grant_type

- `authentication_code`
- `refresh_token`
- `urn:skygear-auth:params:oauth:grant-type:anonymous-request`

The custom grant type is for authenticating and issuing tokens directly for anonymous user.

#### jwt

Required when the grant type is `urn:skygear-auth:params:oauth:grant-type:anonymous-request`. The value is specified [here](#anonymous-identity-jwt)

### Token Response

#### token_type

It is always the value `bearer`.

#### refresh_token

Present only if authorized scopes contain `offline_access`.

#### scope

It is always absent.

### The metadata endpoint

[OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata)

```
<endpoint>/.well-known/openid-configuration
```

[Authorization Server Metadata](https://tools.ietf.org/html/rfc8414#section-2)

```
<endpoint>/.well-known/oauth-authorization-server
```

The following sections list out the supported metadata fields.

#### authorization_endpoint

The value is `<endpoint>/oauth2/authorize`.

#### token_endpoint

The value is `<endpoint>/oauth2/token`.

#### userinfo_endpoint

The value is `<endpoint>/oauth2/userinfo`.

#### revocation_endpoint

The value is `<endpoint>/oauth2/revoke`.

#### jwks_uri

The value is `<endpoint>/oauth2/jwks`.

#### scopes_supported

See [scope](#scope)

#### response_types_supported

See [response_type](#response_type)

#### grant_types_supported

See [grant_type](#grant_type)

#### subject_types_supported

The value is `["public"]`.

#### id_token_signing_alg_values_supported

The value is `["RS256"]`.

#### claims_supported

The value is `["sub", "iss", "aud", "exp", "iat"]`.

#### code_challenge_methods_supported

The value is `["S256"]`

### ID Token

### amr

To indicate authenticator used in authentication, `amr` claim is used in OIDC ID token.

`amr` claim is an array of string. It includes authentication method used:
- If secondary authentication is performed: `mfa` is included.
- If password authenticator is used: `pwd` is included.
- If any OTP (TOTP/OOB-OTP) is used: `otp` is included.
- If WebAuthn is used: `hwk` is included.

If no authentication method is to be included in `amr` claim, `amr` claim would be omitted from the ID token.

### acr

If any secondary authenticator is performed, `acr` claim would be included in ID token with value `http://schemas.openid.net/pape/policies/2007/06/multi-factor`.

To perform step-up authentication, developer can pass a `acr_values` of  `http://schemas.openid.net/pape/policies/2007/06/multi-factor` to the authorize endpoint.


### External application acting as RP while Authgear acting as OP

[![](https://mermaid.ink/img/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gIHBhcnRpY2lwYW50IENsaWVudEFwcFxuICBwYXJ0aWNpcGFudCBBcHBCYWNrZW5kXG4gIHBhcnRpY2lwYW50IEF1dGhnZWFyXG4gIENsaWVudEFwcC0-PkFwcEJhY2tlbmQ6IFVzZXIgY2xpY2sgbG9naW5cbiAgQXBwQmFja2VuZC0-PkF1dGhnZWFyOiBBdXRob3JpemF0aW9uIGNvZGUgcmVxdWVzdFxuICBBdXRoZ2Vhci0-PkNsaWVudEFwcDogUmVkaXJlY3QgdG8gYXV0aG9yaXphdGlvbiBlbmRwb2ludFxuICBDbGllbnRBcHAtPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBhbmQgY29uc2VudFxuICBBdXRoZ2Vhci0-PkFwcEJhY2tlbmQ6IEF1dGhvcml6YXRpb24gY29kZVxuICBBcHBCYWNrZW5kLT4-QXV0aGdlYXI6IEF1dGhvcml6YXRpb24gY29kZSArIGNsaWVudCBpZCArIGNsaWVudCBzZWNyZXRcbiAgQXV0aGdlYXItPj5BdXRoZ2VhcjogVmFsaWRhdGUgYXV0aG9yaXphdGlvbiBjb2RlICsgY2xpZW50IGlkICsgY2xpZW50IHNlY3JldFxuICBBdXRoZ2Vhci0-PkFwcEJhY2tlbmQ6IFRva2VuIHJlc3BvbnNlIChJRCB0b2tlbiArIGFjY2VzcyB0b2tlbiArIHJlZnJlc2ggdG9rZW4pXG4gIEFwcEJhY2tlbmQtPj5BdXRoZ2VhcjogUmVxdWVzdCB1c2VyIGRhdGEgd2l0aCBhY2Nlc3MgdG9rZW5cbiAgQXV0aGdlYXItPj5BcHBCYWNrZW5kOiBSZXNwb25zZSB1c2VyIGRhdGFcbiAgQXBwQmFja2VuZC0-PkFwcEJhY2tlbmQ6IENyZWF0ZSBBcHBCYWNrZW5kIG1hbmFnZWQgc2Vzc2lvblxuICBBcHBCYWNrZW5kLT4-Q2xpZW50QXBwOiBSZXR1cm4gQXBwQmFja2VuZCBtYW5hZ2VkIHNlc3Npb25cbiIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0Iiwic2VxdWVuY2UiOnsic2hvd1NlcXVlbmNlTnVtYmVycyI6dHJ1ZX19fQ)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gIHBhcnRpY2lwYW50IENsaWVudEFwcFxuICBwYXJ0aWNpcGFudCBBcHBCYWNrZW5kXG4gIHBhcnRpY2lwYW50IEF1dGhnZWFyXG4gIENsaWVudEFwcC0-PkFwcEJhY2tlbmQ6IFVzZXIgY2xpY2sgbG9naW5cbiAgQXBwQmFja2VuZC0-PkF1dGhnZWFyOiBBdXRob3JpemF0aW9uIGNvZGUgcmVxdWVzdFxuICBBdXRoZ2Vhci0-PkNsaWVudEFwcDogUmVkaXJlY3QgdG8gYXV0aG9yaXphdGlvbiBlbmRwb2ludFxuICBDbGllbnRBcHAtPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBhbmQgY29uc2VudFxuICBBdXRoZ2Vhci0-PkFwcEJhY2tlbmQ6IEF1dGhvcml6YXRpb24gY29kZVxuICBBcHBCYWNrZW5kLT4-QXV0aGdlYXI6IEF1dGhvcml6YXRpb24gY29kZSArIGNsaWVudCBpZCArIGNsaWVudCBzZWNyZXRcbiAgQXV0aGdlYXItPj5BdXRoZ2VhcjogVmFsaWRhdGUgYXV0aG9yaXphdGlvbiBjb2RlICsgY2xpZW50IGlkICsgY2xpZW50IHNlY3JldFxuICBBdXRoZ2Vhci0-PkFwcEJhY2tlbmQ6IFRva2VuIHJlc3BvbnNlIChJRCB0b2tlbiArIGFjY2VzcyB0b2tlbiArIHJlZnJlc2ggdG9rZW4pXG4gIEFwcEJhY2tlbmQtPj5BdXRoZ2VhcjogUmVxdWVzdCB1c2VyIGRhdGEgd2l0aCBhY2Nlc3MgdG9rZW5cbiAgQXV0aGdlYXItPj5BcHBCYWNrZW5kOiBSZXNwb25zZSB1c2VyIGRhdGFcbiAgQXBwQmFja2VuZC0-PkFwcEJhY2tlbmQ6IENyZWF0ZSBBcHBCYWNrZW5kIG1hbmFnZWQgc2Vzc2lvblxuICBBcHBCYWNrZW5kLT4-Q2xpZW50QXBwOiBSZXR1cm4gQXBwQmFja2VuZCBtYW5hZ2VkIHNlc3Npb25cbiIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0Iiwic2VxdWVuY2UiOnsic2hvd1NlcXVlbmNlTnVtYmVycyI6dHJ1ZX19fQ)

1. User clicks login and call App Backend
1. App Backend generates authorization code request to Authgear
1. Authgear redirects user to authorization page
1. User authorizes and consents
1. Authgear creates IdP session and redirects authorization code result back to App Backend
1. App Backend sends the token request to Authgear with authorization code + client id + client secret
1. Authgear validates authorization code + client id + client secret
1. Authgear returns token response to App Backend
1. App Backend requests user data by using the access token
1. Authgear returns the user data
1. App Backend creates self managed session based on user data
1. App Backend returns self managed session to Client App

### Authgear acting as authentication server with native application

[![](https://mermaid.ink/img/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gIHBhcnRpY2lwYW50IENsaWVudEFwcFxuICBwYXJ0aWNpcGFudCBBdXRoZ2VhciBhcyBBdXRoZ2VhciAoaHR0cHM6Ly9hY2NvdW50cy5teWFwcC5jb20pXG4gIHBhcnRpY2lwYW50IEFwcEJhY2tlbmQgYXMgQXBwQmFja2VuZCAoaHR0cHM6Ly9hcGkubXlhcHAuY29tKVxuICBDbGllbnRBcHAtPj5DbGllbnRBcHA6IEdlbmVyYXRlIGNvZGUgdmVyaWZpZXIgKyBjb2RlIGNoYWxsZW5nZVxuICBDbGllbnRBcHAtPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBjb2RlIHJlcXVlc3QgKyBjb2RlIGNoYWxsZW5nZVxuICBBdXRoZ2Vhci0-PkNsaWVudEFwcDogUmVkaXJlY3QgdG8gYXV0aG9yaXphdGlvbiBlbmRwb2ludFxuICBDbGllbnRBcHAtPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBhbmQgY29uc2VudFxuICBBdXRoZ2Vhci0-PkNsaWVudEFwcDogQXV0aG9yaXphdGlvbiBjb2RlXG4gIENsaWVudEFwcC0-PkF1dGhnZWFyOiBBdXRob3JpemF0aW9uIGNvZGUgKyBjb2RlIHZlcmlmaWVyXG4gIEF1dGhnZWFyLT4-QXV0aGdlYXI6IFZhbGlkYXRlIGF1dGhvcml6YXRpb24gY29kZSArIGNvZGUgdmVyaWZpZXJcbiAgQXV0aGdlYXItPj5DbGllbnRBcHA6IFRva2VuIHJlc3BvbnNlIChpZCB0b2tlbiArIGFjY2VzcyB0b2tlbiArIHJlZnJlc2ggdG9rZW4pXG4gIENsaWVudEFwcC0-PkFwcEJhY2tlbmQ6IFNlbmQgYXBpIHJlcXVlc3Qgd2l0aCBhY2Nlc3MgdG9rZW4sIHJldmVyc2UgcHJveHkgZGVsZWdhdGUgdG8gQXV0aGdlYXIgdG8gcmVzb2x2ZSBzZXNzaW9uXG4gIGxvb3AgV2hlbiBhcHAgbGF1bmNoIG9yIGNsb3NlIHRvIGV4cGlyZWRfaW5cbiAgICBOb3RlIG92ZXIgQ2xpZW50QXBwLEFwcEJhY2tlbmQ6IFJlbmV3IGFjY2VzcyB0b2tlblxuICAgIENsaWVudEFwcC0tPj5BdXRoZ2VhcjogVG9rZW4gcmVxdWVzdCB3aXRoIHJlZnJlc2ggdG9rZW5cbiAgICBBdXRoZ2Vhci0tPj5DbGllbnRBcHA6IFRva2VuIHJlc3BvbnNlIHdpdGggbmV3IGFjY2VzcyB0b2tlblxuICBlbmRcbiIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0Iiwic2VxdWVuY2UiOnsic2hvd1NlcXVlbmNlTnVtYmVycyI6dHJ1ZX19LCJ1cGRhdGVFZGl0b3IiOmZhbHNlfQ)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gIHBhcnRpY2lwYW50IENsaWVudEFwcFxuICBwYXJ0aWNpcGFudCBBdXRoZ2VhciBhcyBBdXRoZ2VhciAoaHR0cHM6Ly9hY2NvdW50cy5teWFwcC5jb20pXG4gIHBhcnRpY2lwYW50IEFwcEJhY2tlbmQgYXMgQXBwQmFja2VuZCAoaHR0cHM6Ly9hcGkubXlhcHAuY29tKVxuICBDbGllbnRBcHAtPj5DbGllbnRBcHA6IEdlbmVyYXRlIGNvZGUgdmVyaWZpZXIgKyBjb2RlIGNoYWxsZW5nZVxuICBDbGllbnRBcHAtPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBjb2RlIHJlcXVlc3QgKyBjb2RlIGNoYWxsZW5nZVxuICBBdXRoZ2Vhci0-PkNsaWVudEFwcDogUmVkaXJlY3QgdG8gYXV0aG9yaXphdGlvbiBlbmRwb2ludFxuICBDbGllbnRBcHAtPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBhbmQgY29uc2VudFxuICBBdXRoZ2Vhci0-PkNsaWVudEFwcDogQXV0aG9yaXphdGlvbiBjb2RlXG4gIENsaWVudEFwcC0-PkF1dGhnZWFyOiBBdXRob3JpemF0aW9uIGNvZGUgKyBjb2RlIHZlcmlmaWVyXG4gIEF1dGhnZWFyLT4-QXV0aGdlYXI6IFZhbGlkYXRlIGF1dGhvcml6YXRpb24gY29kZSArIGNvZGUgdmVyaWZpZXJcbiAgQXV0aGdlYXItPj5DbGllbnRBcHA6IFRva2VuIHJlc3BvbnNlIChpZCB0b2tlbiArIGFjY2VzcyB0b2tlbiArIHJlZnJlc2ggdG9rZW4pXG4gIENsaWVudEFwcC0-PkFwcEJhY2tlbmQ6IFNlbmQgYXBpIHJlcXVlc3Qgd2l0aCBhY2Nlc3MgdG9rZW4sIHJldmVyc2UgcHJveHkgZGVsZWdhdGUgdG8gQXV0aGdlYXIgdG8gcmVzb2x2ZSBzZXNzaW9uXG4gIGxvb3AgV2hlbiBhcHAgbGF1bmNoIG9yIGNsb3NlIHRvIGV4cGlyZWRfaW5cbiAgICBOb3RlIG92ZXIgQ2xpZW50QXBwLEFwcEJhY2tlbmQ6IFJlbmV3IGFjY2VzcyB0b2tlblxuICAgIENsaWVudEFwcC0tPj5BdXRoZ2VhcjogVG9rZW4gcmVxdWVzdCB3aXRoIHJlZnJlc2ggdG9rZW5cbiAgICBBdXRoZ2Vhci0tPj5DbGllbnRBcHA6IFRva2VuIHJlc3BvbnNlIHdpdGggbmV3IGFjY2VzcyB0b2tlblxuICBlbmRcbiIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0Iiwic2VxdWVuY2UiOnsic2hvd1NlcXVlbmNlTnVtYmVycyI6dHJ1ZX19LCJ1cGRhdGVFZGl0b3IiOmZhbHNlfQ)

1. SDK generates code verifier + code challenge
1. SDK sends authorization code request with code challenge
1. Authgear directs user to authorization page
1. User authorizes and consents in authorization page
1. Authgear creates IdP session and redirects the authorization code back to SDK
1. SDK sends token request with authorization code + code verifier
1. Authgear validates authorization code + code verifier
1. Authgear returns token response to SDK with id token + access token + refresh token
1. SDK injects authorization header for subsequent requests, the reverse proxy delegate to Authgear to resolve the session.
1. When app launches or access token expires, SDK sends token request with refresh token
1. Authgear returns token response with new access token

### Authgear acting as authentication server with web application

**Authgear and the web application must under the same root domain**

[![](https://mermaid.ink/img/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gIHBhcnRpY2lwYW50IEJyb3dzZXJcbiAgcGFydGljaXBhbnQgQXV0aGdlYXIgYXMgQXV0aGdlYXI8YnIvPihhY2NvdW50cy5leGFtcGxlLmNvbSlcbiAgcGFydGljaXBhbnQgQXBwQmFja2VuZCBhcyBBcHBCYWNrZW5kPGJyLz4od3d3LmV4YW1wbGUuY29tKVxuICBCcm93c2VyLT4-QXV0aGdlYXI6IEF1dGhvcml6YXRpb24gcmVxdWVzdCB3aXRoIHJlc3BvbnNlX3R5cGU9bm9uZSArIGNsaWVudF9pZFxuICBBdXRoZ2Vhci0-PkJyb3dzZXI6IFJlZGlyZWN0IHRvIGF1dGhvcml6YXRpb24gZW5kcG9pbnRcbiAgQnJvd3Nlci0-PkF1dGhnZWFyOiBBdXRob3JpemF0aW9uIGFuZCBjb25zZW50XG4gIEF1dGhnZWFyLT4-QnJvd3NlcjogU2V0IElkcCBzZXNzaW9uIGluIGVUTEQrMSBhbmQgcmVkaXJlY3QgYmFjayB0byBBcHBCYWNrZW5kXG4gIEJyb3dzZXItPj5BcHBCYWNrZW5kOiBTZW5kIGFwaSByZXF1ZXN0IHdpdGggSWRwIHNlc3Npb24sIHJldmVyc2UgcHJveHkgZGVsZWdhdGVzIHRvIEF1dGhnZWFyIHRvIHJlc29sdmUgc2Vzc2lvblxuIiwibWVybWFpZCI6eyJ0aGVtZSI6ImRlZmF1bHQiLCJzZXF1ZW5jZSI6eyJzaG93U2VxdWVuY2VOdW1iZXJzIjp0cnVlfX0sInVwZGF0ZUVkaXRvciI6ZmFsc2V9)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gIHBhcnRpY2lwYW50IEJyb3dzZXJcbiAgcGFydGljaXBhbnQgQXV0aGdlYXIgYXMgQXV0aGdlYXI8YnIvPihhY2NvdW50cy5leGFtcGxlLmNvbSlcbiAgcGFydGljaXBhbnQgQXBwQmFja2VuZCBhcyBBcHBCYWNrZW5kPGJyLz4od3d3LmV4YW1wbGUuY29tKVxuICBCcm93c2VyLT4-QXV0aGdlYXI6IEF1dGhvcml6YXRpb24gcmVxdWVzdCB3aXRoIHJlc3BvbnNlX3R5cGU9bm9uZSArIGNsaWVudF9pZFxuICBBdXRoZ2Vhci0-PkJyb3dzZXI6IFJlZGlyZWN0IHRvIGF1dGhvcml6YXRpb24gZW5kcG9pbnRcbiAgQnJvd3Nlci0-PkF1dGhnZWFyOiBBdXRob3JpemF0aW9uIGFuZCBjb25zZW50XG4gIEF1dGhnZWFyLT4-QnJvd3NlcjogU2V0IElkcCBzZXNzaW9uIGluIGVUTEQrMSBhbmQgcmVkaXJlY3QgYmFjayB0byBBcHBCYWNrZW5kXG4gIEJyb3dzZXItPj5BcHBCYWNrZW5kOiBTZW5kIGFwaSByZXF1ZXN0IHdpdGggSWRwIHNlc3Npb24sIHJldmVyc2UgcHJveHkgZGVsZWdhdGVzIHRvIEF1dGhnZWFyIHRvIHJlc29sdmUgc2Vzc2lvblxuIiwibWVybWFpZCI6eyJ0aGVtZSI6ImRlZmF1bHQiLCJzZXF1ZW5jZSI6eyJzaG93U2VxdWVuY2VOdW1iZXJzIjp0cnVlfX0sInVwZGF0ZUVkaXRvciI6ZmFsc2V9)

1. SDK sends authorization request with response_type=none + client id
1. Authgear redirects user to authorization page
1. User authorizes and consents
1. Authgear creates IdP session in eTLD+1, redirect empty result back to client SDK
1. Since IdP session is set in eTLD+1, cookies header will be included when user send request to AppBackend too. The reverse proxy delegates to Authgear to resolve the session.

### Silent Authentication

Silent authentication is alternative way to refresh access token without a refresh token. It is achieved with `prompt=none&id_token_hint=...`. In web environment, local storage is not a secure place to store refresh token.

Since [Cookie sharing approach](#authgear-acting-as-authentication-server-with-web-application) covers all basic use cases, this is not yet implemented.

#### Comparison with cookie sharing approach

[Cookie sharing approach](#authgear-acting-as-authentication-server-with-web-application)

- Pros:
  - Cookie is used so Server Side Rendering (SSR) app and Single Page App (SPA) are both supported.
- Cons:
  - Not fully OIDC compliant. The IdP session is used directly.
  - App must be first party app. That is they are sharing the eTLD+1 domain.

Silent Authentication

- Pros:
  - OIDC compliant
  - App does not need to be first party app.
- Cons:
  - Only SPA is supported.

#### Details of Silent Authentication

[![](https://mermaid.ink/img/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gIHBhcnRpY2lwYW50IEJyb3dzZXJcbiAgcGFydGljaXBhbnQgQXV0aGdlYXJcbiAgcGFydGljaXBhbnQgQXBwQmFja2VuZFxuICBCcm93c2VyLT4-QnJvd3NlcjogR2VuZXJhdGUgY29kZSB2ZXJpZmllciArIGNvZGUgY2hhbGxlbmdlXG4gIEJyb3dzZXItPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBjb2RlIHJlcXVlc3QgKyBjb2RlIGNoYWxsZW5nZVxuICBBdXRoZ2Vhci0-PkJyb3dzZXI6IFJlZGlyZWN0IHRvIGF1dGhvcml6YXRpb24gZW5kcG9pbnRcbiAgQnJvd3Nlci0-PkF1dGhnZWFyOiBBdXRob3JpemF0aW9uIGFuZCBjb25zZW50XG4gIEF1dGhnZWFyLT4-QnJvd3NlcjogQXV0aG9yaXphdGlvbiBjb2RlXG4gIEJyb3dzZXItPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBjb2RlICsgY29kZSB2ZXJpZmllclxuICBBdXRoZ2Vhci0-PkF1dGhnZWFyOiBWYWxpZGF0ZSBhdXRob3JpemF0aW9uIGNvZGUgKyBjb2RlIHZlcmlmaWVyXG4gIEF1dGhnZWFyLT4-QnJvd3NlcjogVG9rZW4gcmVzcG9uc2UgKElEIHRva2VuICsgYWNjZXNzIHRva2VuKVxuICBCcm93c2VyLT4-QXBwQmFja2VuZDogUmVxdWVzdCBBcHBCYWNrZW5kIHdpdGggYWNjZXNzIHRva2VuXG4gIGxvb3AgV2hlbiBhcHAgbGF1bmNoIG9yIGNsb3NlIHRvIGV4cGlyZWRfaW5cbiAgICBOb3RlIG92ZXIgQnJvd3NlcixBcHBCYWNrZW5kOiBSZW5ldyBhY2Nlc3MgdG9rZW5cbiAgICBCcm93c2VyLT4-QnJvd3NlcjogR2VuZXJhdGUgY29kZSB2ZXJpZmllciArIGNvZGUgY2hhbGxlbmdlXG4gICAgQnJvd3Nlci0-PkF1dGhnZWFyOiBJbmplY3QgaWZyYW1lIHRvIHNlbmQgYXV0aG9yaXphdGlvbiByZXF1ZXN0XG4gICAgQXV0aGdlYXItPj5BdXRoZ2VhcjogUmVkaXJlY3QgYXV0aG9yaXphdGlvbiBjb2RlIHJlc3VsdCB0byBBdXRoZ2VhciBlbmRwb2ludFxuICAgIEF1dGhnZWFyLT4-QnJvd3NlcjogUG9zdCBtZXNzYWdlIHdpdGggYXV0aG9yaXphdGlvbiBjb2RlIHJlc3VsdFxuICAgIEJyb3dzZXItLT4-QnJvd3NlcjogSWYgSWRwIFNlc3Npb24gaXMgaW52YWxpZCwgbG9nb3V0XG4gICAgQnJvd3Nlci0-PkF1dGhnZWFyOiBTZW5kIHRva2VuIHJlcXVlc3Qgd2l0aCBjb2RlICsgY29kZSB2ZXJpZmllclxuICAgIEF1dGhnZWFyLT4-QnJvd3NlcjogVG9rZW4gcmVzcG9uc2UgKGlkIHRva2VuICsgYWNjZXNzIHRva2VuKVxuICBlbmRcbiIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0Iiwic2VxdWVuY2UiOnsic2hvd1NlcXVlbmNlTnVtYmVycyI6dHJ1ZX19LCJ1cGRhdGVFZGl0b3IiOmZhbHNlfQ)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gIHBhcnRpY2lwYW50IEJyb3dzZXJcbiAgcGFydGljaXBhbnQgQXV0aGdlYXJcbiAgcGFydGljaXBhbnQgQXBwQmFja2VuZFxuICBCcm93c2VyLT4-QnJvd3NlcjogR2VuZXJhdGUgY29kZSB2ZXJpZmllciArIGNvZGUgY2hhbGxlbmdlXG4gIEJyb3dzZXItPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBjb2RlIHJlcXVlc3QgKyBjb2RlIGNoYWxsZW5nZVxuICBBdXRoZ2Vhci0-PkJyb3dzZXI6IFJlZGlyZWN0IHRvIGF1dGhvcml6YXRpb24gZW5kcG9pbnRcbiAgQnJvd3Nlci0-PkF1dGhnZWFyOiBBdXRob3JpemF0aW9uIGFuZCBjb25zZW50XG4gIEF1dGhnZWFyLT4-QnJvd3NlcjogQXV0aG9yaXphdGlvbiBjb2RlXG4gIEJyb3dzZXItPj5BdXRoZ2VhcjogQXV0aG9yaXphdGlvbiBjb2RlICsgY29kZSB2ZXJpZmllclxuICBBdXRoZ2Vhci0-PkF1dGhnZWFyOiBWYWxpZGF0ZSBhdXRob3JpemF0aW9uIGNvZGUgKyBjb2RlIHZlcmlmaWVyXG4gIEF1dGhnZWFyLT4-QnJvd3NlcjogVG9rZW4gcmVzcG9uc2UgKElEIHRva2VuICsgYWNjZXNzIHRva2VuKVxuICBCcm93c2VyLT4-QXBwQmFja2VuZDogUmVxdWVzdCBBcHBCYWNrZW5kIHdpdGggYWNjZXNzIHRva2VuXG4gIGxvb3AgV2hlbiBhcHAgbGF1bmNoIG9yIGNsb3NlIHRvIGV4cGlyZWRfaW5cbiAgICBOb3RlIG92ZXIgQnJvd3NlcixBcHBCYWNrZW5kOiBSZW5ldyBhY2Nlc3MgdG9rZW5cbiAgICBCcm93c2VyLT4-QnJvd3NlcjogR2VuZXJhdGUgY29kZSB2ZXJpZmllciArIGNvZGUgY2hhbGxlbmdlXG4gICAgQnJvd3Nlci0-PkF1dGhnZWFyOiBJbmplY3QgaWZyYW1lIHRvIHNlbmQgYXV0aG9yaXphdGlvbiByZXF1ZXN0XG4gICAgQXV0aGdlYXItPj5BdXRoZ2VhcjogUmVkaXJlY3QgYXV0aG9yaXphdGlvbiBjb2RlIHJlc3VsdCB0byBBdXRoZ2VhciBlbmRwb2ludFxuICAgIEF1dGhnZWFyLT4-QnJvd3NlcjogUG9zdCBtZXNzYWdlIHdpdGggYXV0aG9yaXphdGlvbiBjb2RlIHJlc3VsdFxuICAgIEJyb3dzZXItLT4-QnJvd3NlcjogSWYgSWRwIFNlc3Npb24gaXMgaW52YWxpZCwgbG9nb3V0XG4gICAgQnJvd3Nlci0-PkF1dGhnZWFyOiBTZW5kIHRva2VuIHJlcXVlc3Qgd2l0aCBjb2RlICsgY29kZSB2ZXJpZmllclxuICAgIEF1dGhnZWFyLT4-QnJvd3NlcjogVG9rZW4gcmVzcG9uc2UgKGlkIHRva2VuICsgYWNjZXNzIHRva2VuKVxuICBlbmRcbiIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0Iiwic2VxdWVuY2UiOnsic2hvd1NlcXVlbmNlTnVtYmVycyI6dHJ1ZX19LCJ1cGRhdGVFZGl0b3IiOmZhbHNlfQ)

1. SDK generates code verifier + code challenge
1. SDK sends authorization code request with code challenge
1. Authgear directs user to authorization page
1. User authorizes and consents in authorization page
1. Authgear creates Idp session and redirects the result (authorization code) back to client SDK
1. SDK send token request with authorization code + code verifier
1. Authgear validates authorization code + code verifier
1. Authgear retursn token response to SDK with id token + access token
1. SDK inject authorization header for subsequent requests to AppBackend, the reverse proxy delegate to Authgear to resolve the session.
1. Trigger silent authentication to obtain new access token when app launches or access token expiry, generate code verifier + code challenge for new authorization flow
1. Inject iframe with Authgear authorization endpoint, the authorization request includes code request + code challenge + id_token_hint + prompt=none
1. Authgear redirects the result (authorization code) back to an Authgear specific endpoint
1. Authgear specific endpoint posts the result back to parent window (SDK)
1. SDK reads the result message, logout if the result indicates IdP Session is invalid
1. If authorization code request result is successful, send the token request to Authgear with code + code verifier
1. Authgear returns token response to SDK with id token + new access token

## Templates

Authgear serves web pages and send email and SMS messages. Templates allow the developer to provide localization or even customize the default ones.

### Template

Each template must have a type, optionally a key and a language tag.

#### Template Type

Each template must have a type. The list of types are predefined. Here are some examples

```
forgot_password_email.html
forgot_password_email.txt

user_verification_message.html
user_verification_message.txt
```

#### Template Key

Some template may require a key. The key is used differentiate different instances of the same type of the template. For example, the verification message template of email message should be different from that of SMS message.

#### Template Language Tag

Each template may optionally have a language tag. The language tag is specified in [BCP47](https://tools.ietf.org/html/bcp47).

### Template Resolution

To resolve a template, the input is the template type, optionally the template key and finally the user preferred languages. The type and key is determined by the feature while the user preferred languages is provided by the user.

All templates have default value so template resolution always succeed.

The templates are first resolved by matching the type and the key. And then select the best language according to the user preferred languages.

### Component Templates

Some template may depend on other templates which are included during rendering. This enables customizing a particular component of a template. The dependency is expressed by a whitelist that is hard-coded by the Skygear developer. It can be assumed there is no dependency cycle.

For example, `auth_ui_login.html` depend on `auth_ui_header.html` and `auth_ui_footer.html` to provide the header and footer. If the developer just wants to customize the header, they do not need to provide customized templates for ALL pages. They just need to provide `auth_ui_header.html`.

### Localization of the text of the template

In addition to the template language tag, sometimes it is preferred to localize the text of the template rather the whole template.

For example, `auth_ui_login.html` defines the HTML structure and is used for all languages. What the developer wants to localize is the text.

### localize

A special function named `localize` can be used to format a localized string.

```html
<input type="password" placeholder="{{ localize "enter.password" }}">
<!-- <input type="password placeholder="Enter Password"> -->
```

```html
<p>{{ localize "email.sent" .email .name }}</p>
<!-- <p>Hi John, an email has been sent to john.doe@example.com</p -->
```

`localize` takes a translation key, followed any arguments required by that translation key. If the key is not found, the key itself is returned.

### Translation file

The translation file is a template itself. It is simply a flat JSON object with string keys and string values. The value is in ICU MessageFormat. Not all ICU MessageFormat arguments are supported. The supported are `select`, `plural` and `selectordinal`.

Here is an example of the translation file.

```json
{
  "email.sent": "Hi {1}, an email has been sent to {0}"
}
```

#### Translation Resolution

Translation resolution is different from template resolution. Template resolution is file-based while translation resolution is key-based.

For example,

```json
// The zh variant of auth_ui_translation.json
{
  "enter.password": "輸入密碼",
  "enter.email": "輸入電郵地址"
}
```

```json
// The zh-Hant-HK variant of auth_ui_translation.json
{
  "enter.password": "入你嘅密碼"
}
```

And the user preferred languages is `["zh-Hant-HK"]`.

`"enter.password"` resolves to `"入你嘅密碼"` and `"enter.email"` resolves to `"輸入電郵地址"`.

## The resolve endpoint

Popular reverse proxy server supports delegating request authentication by initiating subrequest.

In nginx, it is the `auth_request` directive while in Traefik, it is `ForwardAuth`.

The resolve endpoint `/resolve` looks at `Cookie:` and `Authentication:` to authenticate the request. `Cookie:` has higher precedence.

The resolve endpoint does not write body. Instead it adds the following headers in the response.

### x-skygear-session-valid

Tell whether the session of the original request is valid.

If this header is absent, it means the original request is not associated with any session.

If the value is `true`, it indicates the original request has valid session. More headers will be included.

If the value is `false`, it indicates the original request has invalid session.

### x-skygear-user-id

The user id.

### x-skygear-user-anonymous

The value `true` means the user is anonymous. Otherwise it is a normal user.

### x-skygear-session-acr

See [the acr claim](#acr).

### x-skygear-session-amr

See [the amr claim](#amr). It is comma-separated.

## UI

The UI creates new users and authenticates existing ones. The user can manage their account in the settings page.

### Theming

The developer can provide a CSS stylesheet to customize the theme of the UI. Or they override the templates with their own ones.

### The phone input widget

The developer can customize the list of country calling code and the default country calling code of the phone input widget via configuration. By default the list includes all country calling codes globally. The default value is the first one in the list.

### frame-ancestors

The `frame-ancestors` directive of the HTTP header `Content-Security-Policy:` is derived from the `redirect_uris` of all clients. If the `redirect_uri` is of scheme `https`, the host is added to to frame-ancestors. If the `redirect_uri` is `http` and the host is loopback address or the domain ends with `.localhost`, the host is also added to frame-ancestors.

### The login page

The login page authenticates the user. It lists out the configured IdPs. It shows a text field for login ID. The login ID field is either a plain text input or a phone number input, depending on the type of the first login ID key. Link to the forgot password page is shown if Password Authenticator is enabled.

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
Forgot password?
```

### The enter password page

The enter password page displays a visibility toggleable password field.

```
|-----------------------|  |----------|
| Enter a password here |  | Continue |
|-----------------------|  |----------|
```

### The signup page

The signup page creates new user. It looks like the login page. It displays the first login ID key by default. Other login ID keys are available to choose from.

```
Sign up with email
|--------------------------|  |----------|
| Enter in your email here |  | Continue |
|--------------------------|  |----------|

Sign up with phone instead.
Sign up with username instead.
```

### The create password page

The create password page displays a visibility toggleable password field with password requirements.

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

### The forgot password page

The forgot password page displays an email text field. When the user enter a valid Email Login ID, a reset password link to sent to that email address.

### The reset password page

The reset password page looks like the create password page.

### The OOB OTP page

The OOB OTP page lets the user to input OOB OTP. A resend button with cooldown is shown as well.

### The identity page

The identity page lists out the candidates of identity and the status.

```
|---------------------------------------|
| Google                        Connect |
|---------------------------------------|
| Email                                 |
| user@example.com               Change |
|---------------------------------------|
| Phone                             Add |
|---------------------------------------|
```

## Session Management

Sessions in Authgear are stateful. The user can manage sessions. The developer can configure session characteristics such as lifetime and idle timeout.

### Session Attributes

Session has the following attributes:

- ID
- User ID
- AMR
- ACR
- Creation Time
- Last Access Time
- Creation IP
- Last Access IP
- User Agent

In particular, session does not have reference to involved identity and authenticators in the authentication. Removal of identity and authenticators does not invalidate session.

### Session Management

The user manages their sessions in the settings page. They can list the sessions and revoke them.

TODO(session): support session name. Default session name should be device name.

### Session token

Session token must be treated as opaque string.

### Session Types

#### IdP Session

When the user authenticates successfully, an IdP session is created.

Idp session has configurable lifetime. IdP session may optionally have idle timeout. The session must be accessed before the timeout or the session is expired.

IdP session token is stored in the user agent cookie storage. The cookie domain attribute is configurable. The default value is eTLD + 1. As long as the web application is under the same domain with Authgear, the IdP session is shared across between the two. The cookie is a persistent cookie by default. The cookie is http-only and is not configurable. The cookie is SameSite=lax and is not configurable. The cookie is secure by default.

The IdP session configuration is global.

#### Offline Grant

Each OAuth client has its own configuration of offline grant.

Offline grant consists of a refresh token and an access token. As long as the refresh token remains valid, access tokens can be refreshed with the refresh token independent of the IdP session. Offline grant is intended for use in native application.

Access token has configurable lifetime.

Refresh token has configurable lifetime. It cannot be refreshed. The old access token is invalidated during refresh. At any time there is at most one valid access token.

The lifetime of offline grant is the lifetime of its refresh token.

## Configuration Conventions

This section outlines the configuration conventions Authgear must follow.

### Prefer list over map

Instead of

```yaml
login_id_keys:
  email:
    type: email
  phone:
    type: phone
  username:
    type: username
```

Do this

```yaml
login_id_keys:
- key: email
  type: email
- key: phone
  type: phone
- key: username
  type: username
```

### Introduce flag only if necessary

Add `enabled` or `disabled` flag only if necessary, such as toggling on/off of a feature.

### References

- [Kubernetes api conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
