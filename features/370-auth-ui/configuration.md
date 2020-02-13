# Configuration

Auth UI respects some of the existing configuration.

- It respects the set of allowed redirect URLs of SSO.
- It respects the display app name. The value of display app name is available to the page templates.

```yaml
display_app_name: My App
sso:
  oauth:
    allowed_callback_urls:
    - https://myapp.com
auth_ui:
  css: |
    .primary-btn {
      background-color: purple;
    }
  content_secure_policy: "frame-ancestors https://thirdpartyapp.com 'self'"
```

- `css`: A string of CSS to be included in all pages of Auth UI.
- `content_secure_policy`: If present, override the default HTTP Header `Content-Secure-Policy` set by Auth UI.
