# SDK

The following new methods are added to access Auth UI.

## React Native

```typescript
interface LoginOptions {
  onUserDuplicate?: "abort" | "merge" | "create";
}

// Open a WebView to login. If it is canceled, the promise will be rejected.
function loginWithWebUI(redirectURI: string, options?: LoginOptions): Promise<User>;

// Open a WebView to access the given Auth UI path.
// It interacts with the native cookie storage to ensure
// the session is attached to the WebView correctly.
function openWebUI(path: string): Promise<void>;
```

## Web

```typescript
interface LoginOptions {
  onUserDuplicate?: "abort" | "merge" | "create";
}

// Redirect the current window to Auth UI.
function loginWithWebUI(redirectURI: string, options?: LoginOptions): Promise<void>;

// Retrieve the result when being redirected back.
function getAuthResult(): Promise<User | null>;

// Construct the URL to the given Auth UI path.
// The developer can then either use it in:
// 1. <iframe src>
// 2. window.location
// 3. window.open()
function getWebUI(path: string): string;
```

## Use cases

### Authenticate in a React Native application

```typescript
import React, { useCallback } from "react";
import skygear from "@skygear/react-native";
import {Button, View} from "react-native";

function AuthenticationScreen({ navigate } : { navigate: (screenName: string) => void }) {
  const onPress = useCallback(() => {
    try {
      const user = await skygear.auth.loginWithWebUI("myappid://anyhost/anypath");
      // If the application requires verification, check if the user is verified.
      if (!user.isVerified) {
        navigate("VerificationScreen");
      } else {
        navigate("HomeScreen");
      }
    } catch (e) {
      // Handle the error properly here.
      // For example, duplicated user.
    }
  }, []);
  return (
    <View>
      <Button title="Sign in" onPress={onPress} />
    </View>
  );
}
```

### Settings in a React Native application

```typescript
import React, { useCallback } from "react";
import skygear from "@skygear/react-native";
import {Button, View} from "react-native";

function SettingsScreen() {
  const onPressEditProfile = useCallback(() => {
    // Application-specific settings
  }, []);

  const onPressSecuritySettings = useCallback(() => {
    skygear.auth.openWebUI("/settings").catch(e => {
      // The error here should be unrecoverable.
    });
  }, []);

  return (
    <View>
      <Button title="Edit Profile" onPress={onPressEditProfile} />
      <Button title="Security Settings" onPress={onPressSecuritySettings} />
    </View>
  );
}
```

### Authenticate in a Web application

```typescript
import React, { useCallback, useEffect } from "react";
import skygear from "@skygear/web";

function AuthenticationScreen() {
  const onClick = useCallback(() => {
    await skygear.auth.loginWithWebUI("https://myapp.skygearapps.com/continue");
  }, []);
  return (
    <div>
      <button onClick={onClick}>Sign in</button>
    </div>
  );
}

function ContinueScreen() {
  useEffect(() => {
    try {
      const user = await skygear.auth.getAuthResult();
      if (user == null) {
        // This route is likely to be accessed manually.
        // Redirect to home
        window.location = "https://myapp.skygearapps.com/home";
      } else {
        if (!user.isVerified) {
          window.location = "https://myapp.skygearapps.com/verify";
        } else {
          window.location = "https://myapp.skygearapps.com/home";
        }
      }
    } catch (e) {
      // Handle the error properly here.
      // For example, duplicated user.
    }
  }, []);
  return (
    <div>Loading</div>
  );
}
```

### Settings in a Web application

```typescript
import React, { useCallback } from "react";
import skygear from "@skygear/web";

function SettingsScreen() {
  const url = skygear.auth.getWebUI("/settings");
  const onClickOpenSettingsAsPopup = useCallback(() => {
    window.open(url);
  }, []);
  const onClickOpenSettings = useCallback(() => {
    window.location = url;
  }, []);
  return (
    <div>
      <button onClick={onClickOpenSettingsAsPopup}>Open Settings as popup</button>
      <button onClick={onClickOpenSettings}>Open Settings</button>
      <iframe src={url} />
    </div>
  );
}
```
