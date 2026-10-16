---
date: "2022-09-08T09:35:26+08:00"
draft: false
title: "Hacking Staysafe Philippines Contact Tracing Platform"

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

## **Description**
During the pandemic, I found vulnerabilities in the StaySafe Android application: a user location disclosure issue, a full account takeover via access token, and admin panel access exposing millions of users’ PII.

When a user goes to the StaySafe website to create an account or log in, I chose to log in with my Facebook account. After authenticating with Facebook, I noticed it redirected to StaySafe with an endpoint like this:
```
https://www.staysafe.ph/via-facebook?#access_token=[token]&data_access_expiration_time=1624186784&expires_in=5155980
```

I noticed this exposed my user access token directly in the URL, and that this token alone could be used to access a StaySafe user’s account. I tested it multiple times and was able to log in using only the access token no email or password required. The token remained valid for up to 59 days, 16 hours, 26 minutes, and 20 seconds before expiring. If this token leaked, anyone could access that user’s StaySafe account and data for nearly two months.

Next, after obtaining my own access token, I downloaded the StaySafe mobile application (package name: `ph.staysafe.mobileapp`, version name: `2.1`) to dig deeper. I reverse-engineered the application using Jadx and apktool to look for additional information and entry points. At the time, StaySafe PH had over 5 million downloads on the Google Play Store; it has since been removed from the store.

![Actual photos](/images/reverse_staysafe.png)


While reviewing the decompiled code, I found a large amount of sensitive information, including this URL pattern with a token parameter:

```
https://www.staysafe.ph/wb/logbook?token=[]
```

I tested this endpoint by appending my own access token to the URL and opening it. What I found was essentially a Bluetooth contact-tracing log, it showed nearby users’ COVID-19 exposure status (Probable, Suspect, Confirmed, Cleared, Initial) with color coding, organized by date, including historical contacts. This meant the endpoint could expose other users’ locations and infection status without authorization.

![Actual photo](/images/staysafe.jpg)

I then reviewed the website’s source code for further leads and extracted all endpoints referenced in the main JavaScript file. This extraction revealed 20+ API endpoints, which could be used to reset a user’s password, verify OTPs, activate/deactivate accounts, delete accounts, and disclose user information, among other actions.
\
![Actual photos](/images/staysafe2.png)


Finally, I located the web directory where the backend source code was exposed. I was able to access and edit portions of the source code, as well as the path to the Laravel admin panel, which disclosed both web and mobile user data including full names, email addresses, passwords, company names, full locations, phone numbers, OTPs, IP addresses, and more.

![Actual photos](/images/staysafe3.png)

## Technical Details

1. OAuth access token exposed in URL (Facebook login redirect)
```
https://www.staysafe.ph/via-facebook?#access_token=[token]&data_access_expiration_time=1624186784&expires_in=5155980
```
* Token usable standalone to authenticate as the user (no password required)
* Token lifetime: ~59 days, 16 hours, 26 minutes, 20 seconds

2. Mobile app reverse engineering
* Package name: `ph.staysafe.mobileapp`
* Version name: `2.1`
* Tools used: `Jadx`, `apktool`

3. Contact-tracing log endpoint (token-authenticated)
```
https://www.staysafe.ph/wb/logbook?token=[access_token]
```
* Returns nearby users’ exposure status (Probable, Suspect, Confirmed, Cleared, Initial), color-coded and organized by date
* Includes historical contact data, not just current-day

4. Exposed API endpoints (extracted from main JS bundle)
* 20+ endpoints identified, including functionality for:
    * Password reset
    * OTP verification
    * Account activation / deactivation
    * Account deletion
    * User data disclosure

5. Exposed backend directory / Laravel admin panel
* Backend source code was publicly accessible and editable
* Admin panel path disclosed millions of PIIs for both web and mobile users: 
    * Full names
    * Family members
    * Email addresses
    * Passwords
    * Company names
    * Previous/current locations
    * Phone numbers
    * OTPs
    * Vaccination status
    * Locations, and more

## **Disclosure Timeline**
* **March 22, 2021** — Emailed StaySafe regarding these vulnerabilities; received a same-day response requesting further details.
* **March 23, 2021** — Sent full documentation of the findings along with proof-of-concept details.
* **March 25, 2021** — Emailed the MultiSys security team and CEO David Almirol; received a reply from StaySafe’s Derit Ginovin confirming they were collaborating on a fix.
* **December 31, 2021** — Followed up with the help of Art Samaniego Jr. for a status update.
* **September 2, 2022** — Followed up again with the help of John Patrick Lita.
* **September 5, 2022** — Vulnerability fixed.