# Implementation Details

This target audience of this document is the implementers.

## Relation between Auth UI and the existing JSON-based API

The existing JSON-based API is a simple API that is unaware of OpenID Connect with PKCE. The implementation of Auth UI will share most of the underlying logic with the JSON API. Both will be maintained.

## Validation on the query parameter redirect_uri

If the scheme and the host:port matches the current request, then `/_auth/_/*` is always allowed. This ensures Auth UI can redirect to itself without any configuration. This is crucial for verification to work.

## Cookie extraction in React Native application

On Android, it is possible to retrieve the cookies with ForwardingCookieHandler. The idea is to write a Native Module to instantiate a ForwardingCookieHandler. It have public methods to retrieve cookies for a given URL.

On iOS, it is possible to retrieve the cookies with `[NSHTTPCookieStorage sharedHTTPCookieStorage]`. WKWebview, however, has its own cookie storage. So we need to pull the cookies from NSHTTPCookieStorage and put them in WKHTTPCookieStore.
