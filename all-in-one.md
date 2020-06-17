# Authgear

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
