---
layout: default
title:  "Auth Bypass via SAML Attacks"
author: Robert
---

# MiniOrange Authentication Bypass via SAML Manipulation - CVE-2022-26493

## Versions

**Known Versions Affected**

| Drupal 9 |  Drupal 8     | Drupal 7 |
| ----------- | ----------- | ----------- |
| MiniOrange Premium < 30.5 | MiniOrange Premium < 30.5      | MiniOrange Premium < 30.2       |

This vulnerability was discovered using Drupal 9.3.4 and 9.3.6, in miniOrange Premium versions 8.x-30.3 and 8.x-30.4. It has been confirmed to affect the Drupal
SAML SP modules provided by MiniOrange.  Other authentication schemas may also be affected.

MiniOrange Enterprise versions prior to version 40.4 (Drupal 8) and version 40.2 (Drupal 7) may be affected but were not tested. Xecurify has released patches for all variants of MiniOrange since the discovery
of this vulnerability: Standard, Premium, and Enterprise.  It is advised to upgrade to the newest available version of MiniOrange, regardless of free or paid subscription service. 

## Vulnerability

This module has an authentication and authorization bypass vulnerability. Any individual with access to a HTTP request intercepting method is able to bypass 
authentication and authorization - impersonating existing users and any existing role. All that is required is for the attacker to know (or guess) a valid username.

Due to the SAML SP module's failure to verify *the existence* of a signature, any applications using the affected versions of miniOrange are effectively without authentication or authorization controls.

## Explanation

MiniOrange versions 8.x-30.3 and 8.x-30.4 do not properly check for the existence of a valid signature in SAML authentication communications.  Manipulating
the contents of the SAML response from an Identity Provider (IDP) will affect the signature - rendering the SAML communication invalid.  This occurs because the MiniOrange 
SAML module can be configured to check the signature provided in SAML communications. If the signature displays indications of modification, the authentication
request is denied.  

However, removal of this signature will bypass the signature check entirely. As a result,an attacker may simply remove the signature in the SAML response from an IDP, and modify the username and/or role and forward the packet on to the web server.
The web server will interpret this manipulated package as a valid communication from the IDP, and allow the attacker to access the application with whatever user account and role he chooses.

SAML (Security Assertion Markup Language) is an authentication standard that is used to transmit authentication data between two parties.  In the case of this vulnerability - 
and in many authentication schemas - SAML allows a web application to be hosted indepently of the authorization service.  SAML works by facilitating communcation between the web application servers 
and the authentication servers.  

Let's look at a simplified example:

A web application possesses a login button that redirects the user to an external authentication source.  As the user is redirected, a token or session cookie is often attached to their request.  The username and
password provided by the user are never sent to the web application - but to the Identity Provider (IDP) authentication service.  This service will validate the user's credentials and - if present - conduct
a two-factor authentication step.  Once authenticated, the IDP will bundle up information about the user such as username, user role, status of authentication attempt (successful or failed) in a SAML Assertion response. This assertion is signed
to validate that (a) the IDP and only the IDP generated the data and (b) that the integrity of the data has not been compromised.

This SAML Assertion is then forwarded from the IDP to the web application.  The web application will read this response (and signature) to confirm that the authentication was successful, obtain the correct username for the user, and apply the correct user role.

In a functioning SAML schemas - any manipulation to the data in-trasit results in a small but significant change to the signature.  The signature contained in the SAML response is compared with what the web application possess
and any differences between the two will result in the authentication attempt failing.  In the versions of MiniOrange discussed in this write-up, this functionality did not fail.

However, the versions of MiniOrange investigated did not properly check for the existance of the signature.  If the signature was removed, the signature check failed open. This leaves the affected versions of MiniOrange vulnerabile
to SAML manipulation attacks.

## SAML Manipulation

During the authentication process, and attacker can modify the contents of a SAML assertion to impersonate users or obtain unauthorized role access. This is 
accomplished by intercepting the SAML response from the IDP, modifying it in-transit, and forwarding it to the web application.  In the vulnerability discussed here, **SAML replay attacks**
were also possible.  As such, once a single SAML Assertion is obtained by an attacker - it can be reused repeatedly.  If an attacker is able to intercept or obtain 
a SAML assertion for an application, the application authentication and authorization logic can be bypassed.

![SAML Manipulation Attack](/_img/SAML_cap2.PNG)

When modifying a SAML Assertion, we can capture the traffic in an HTTP Proxy like Burp Suite and view the SAML data. In order to bypass authentication in these versions
of MiniOrange, and attacker only needs to know or guess an existing, valid username.  This can be fairly easily accomplished through fuzzing different common usernames such as WPAdmin, Admin,
Drupal_admin, etc.  Once identified in the SAML Assertion, the username only needs to be changed.  SAML Assertions are encoded. Encoding is not encryption however, and it is trivial to decode.

Once decoded, we can locate and modify the username.  Here we see I changed my username to an admin account:

![SAML Decoded](/_img/evil_admin.png)

If an attacker does not know or does not want to guess an existing username they can just as easily modify the user role.  In this instance, I modified my user role to "Admin".

![SAML Role](/_img/SAML_role_1.PNG)

After modifying the username and/or the user role, an attacker may remove the signature and forward the packet on to its final destination at the web application server. In this instance, we are met with something like this: A successful authentication and a redirect to the webpage - now as an authorized user.

![SAML Success](/_img/success.png)

## Discussion





