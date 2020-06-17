# Authgear

## Authenticator

Authgear supports various kinds of authenticator. Authenticator can be primary, secondary or both.

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

TODO: Document secondary_authentication_mode

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

## SSO

Authgear integrates with various OAuth 2 IdPs. Since Authgear is a server, the only implemented flow is the authorization code flow.

If the specific OAuth 2 IdP implements OIDC, then the integration conforms to OIDC instead.

### OIDC IdPs

The following IdPs are integrated with OIDC:

- Google
- Apple
- Azure AD

### OAuth 2 IdPs

The following IdPs does not support OIDC. The integration is provider-specific.

- LinkedIn
- Facebook

## Session Management

Sessions in Authgear are stateful. The user can manage sessions. The developer can configure session characteristics such as lifetime and idle timeout.

### Session Attributes

Session has the following attributes

- ID
- User ID
- [AMR](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)
- Creation Time
- Last Access Time
- Creation IP
- Last Access IP
- User Agent

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
