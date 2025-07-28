---
title: "OAuth 2.0 App2App Browser-less Flow"
abbrev: "Native OAuth App2App"
category: std

docname: draft-zehavi-oauth-app2app-browserless-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - native-apps
 - oauth
 - app2app
 - browser-less
 - browserless
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "yaron-zehavi/oauth-app2app-browserless"
  latest: "https://yaron-zehavi.github.io/oauth-app2app-browserless/draft-zehavi-oauth-app2app-browserless.html"

author:
 -
    fullname: Yaron Zehavi
    organization: Raiffeisen Bank International
    email: yaron.zehavi@rbinternational.com

normative:
  RFC6749:
  RFC6750:
  RFC7636:
  RFC8252:
  RFC8414:
  RFC9126:
  RFC9396:
  OpenID:
    title: OpenID Connect Core 1.0
    target: https://openid.net/specs/openid-connect-core-1_0.html
    date: November 8, 2014
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M.B. Jones
      - ins: B. de Medeiros
      - ins: C. Mortimore
  OpenID.Federation:
    title: OpenID Federation 1.0
    target: https://openid.net/specs/openid-federation-1_0.html
    date: March 5, 2025
    author:
      - ins: R. Hedberg, Ed.
      - ins: M.B. Jones
      - ins: A.A. Solberg
      - ins: J. Bradley
      - ins: G. De Marco
      - ins: V. Dzhuvinov
informative:
  App2App:
    title: "Guest Blog: Implementing App-to-App Authorisation in OAuth2/OpenID Connect"
    target: https://openid.net/guest-blog-implementing-app-to-app-authorisation-in-oauth2-openid-connect/
    author:
      - ins: J. Heenan
    date: October 21, 2019
  OpenID.Native-SSO:
    title: OpenID Connect Native SSO for Mobile Apps
    target: https://openid.net/specs/openid-connect-native-sso-1_0.html
    author:
      - ins: G. Fletcher
    date: November 2022
  OAuth.First-Party:
    title: OAuth 2.0 for First-Party Applications
    target: https://www.ietf.org/archive/id/draft-ietf-oauth-first-party-apps-01.html
    author:
      - ins: A. Parecki
      - ins: G. Fletcher
      - ins: P. Kasselman
    date: November 2022
  iOS.method.openUrl:
    title: iOS open(_:options:completionHandler:) Method
    target: https://developer.apple.com/documentation/uikit/uiapplication/open(_:options:completionhandler:)
  iOS.option.universalLinksOnly:
    title: iOS method property universalLinksOnly
    target: https://developer.apple.com/documentation/uikit/uiapplication/openexternalurloptionskey/universallinksonly
  android.method.intent:
    title: Android Intent Method
    target: https://developer.android.com/reference/android/content/Intent
--- abstract

This document describes a protocol allowing a *Client App* to obtain an OAuth grant from a *Native App* using the {{App2App}} pattern, providing **native** app navigation user-experience (no web browser required), despite both apps residing on different trust domains.

--- middle


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

In addition to the terms defined in referenced specifications, this document uses
the following terms:

"OAuth":
: In this document, "OAuth" refers to OAuth 2.0, {{RFC6749}} and {{RFC6750}} as well as {{OpenID}}, both in their **authorization code flow**.

"PKCE":
: Proof Key for Code Exchange (PKCE) {{RFC7636}}, a mechanism
  to prevent various attacks on OAuth authorization codes.

"OAuth Broker":
: An Authorization Server federating to other trust domains by acting as an OAuth Client of  *Downstream Authorization Servers*.

"Client App":
: A Native app acting as client of *Initial Authorization Server*. In accordance with "OAuth 2.0 for Native Apps" {{RFC8252}}, Client's redirect_uri is claimed by the app.

"Initial Authorization Server":
: Authorization Server of *Client App*. As an *OAuth Broker* it is a client of *Downstream Authorization Servers*, to which it federates requests.

"Downstream Authorization Server":
: An Authorization Server downstream of *Initial Authorization Server*. It may be an *OAuth Broker* or the *User-Interacting Authorization Server*.

"User-Interacting Authorization Server":
: An Authorization Server which interacts with end-user. The interaction may be interim navigation (e.g: user input is required to guide where to redirect), or performs user authentication and request authorization.

"User-Interacting App":
: Native App of *User-Interacting Authorization Server*.

"Deep Link":
: A url claimed by a native application.

"Native Callback uri":
: *Client App's* redirect_uri, claimed as a deep link. This deep link is invoked by *User-Interacting App* to natively return to *Client App*.

# Introduction

This document, *OAuth 2.0 App2App Browser-less Flow*, describes a protocol enabling native (**Browser-less**) app navigation of an {{App2App}} OAuth grant.

When Clients and Authorization Servers are located on *different Trust Domains*, authorization requests may traverse across trust domains using federation, involving Authorization Servers acting as clients of *Downstream Authorization Server*.

Such federation setups are used to create trust networks for example in Academia and in the business world across corporations.

However in {{App2App}} scenarios these setups mandate using the web browser as user-agent, because federating Authorization Servers url's are not claimed by any native app.

The use of the web browser in App2App flows, degrades the user experience somewhat.

This document specifies:

* A **Browser-less App2App** profile *Authorization Servers* MUST follow to enable native App2App flows.
* A new Authorization Server metadata property: native_authorization_endpoint, indicating to clients that an *Authorization Server* supports the **Browser-less App2App** profile.
* A new {{RFC9396}} Authorization Details Type: **https://scheme.example.org/native_callback_uri**.
* A new error code value: **native_app2app_unsupported**
* A new error_description value for *invalid_request* error: **native_callback_uri_not_claimed**.

## App2App with OAuth Brokers requires a web browser

~~~ aasvg
{::include art/app2app-w-brokers-and-browser-2.ascii-art}
~~~
{: #app2app-w-brokers-and-browser title="App2App across trust domains using browser" }


Since no native app claims the url's of redirecting Authorization Servers (*OAuth Brokers*), mobile OS' default to using the system browser as the User Agent.

## Impact of using a web browser

Using a web browser may degrade the user experience in several ways:

* Some browser's support for deep links is limited by design, or by the settings used.
* Browsers may prompt end-user for consent before opening apps claiming deep links, introducing additional friction.
* Browsers are noticeable by end-users, rendering the UX less smooth.
* Client app developers don't control which browser the *User-Interacting App* uses to provide its response to redirect_uri. Opinionated choices pose a risk that different browsers will use, making necessary cookies used to bind session identifiers to the user agent (nonce, state or PKCE verifier) unavailable, which may break the flow.
* After flow completion, "orphan" browser tabs may remain. They do not directly impact the flow, but can be regarded as unnecessary "clutter".

## Note - App2Web across trust domains
~~~ aasvg
{::include art/app2web-w-brokers-2.ascii-art}
~~~
{: #app2web-w-brokers title="App2Web across trust domains" }

When end-user's device has **no app** claiming *User-Interacting Authorization Server's* urls, the browser MUST be used to interact with end-user.

This is the case when:

* No native app is offered by *User-Interacting Authorization Server* offers no native app.
* Or such an app is offered, but is not installed on the end-user's device.

In such case the flow is as described in "OAuth 2.0 for Native Apps" {{RFC8252}} and is therefore not discussed further in this document.

## Relation to {{OpenID.Native-SSO}}

{{OpenID.Native-SSO}} also offers a native SSO flow across apps. However, it is limited to apps:

* Published by the same issuer, therefore can securely share information.
* Using the same Authorization Server.

## Relation to {{OAuth.First-Party}}

{{OAuth.First-Party}} also deals with native apps, but it MUST only be used by first-party applications, which is when the authorization server and application are controlled by the same entity, which is not true in the case described in this document.

# Protocol Overview

## Usage and Applicability

This specification MUST NOT be used when *Client App* detects *Initial Authorization Server's* url is claimed by an app on the device.

In such case *Client App* SHOULD natively invoke the authorization request url.

## Authorization Server Metadata

This document introduces the following authorization server metadata {{RFC8414}} parameter to signal support for **native App2App**.

**native_authorization_endpoint**:
URL of the authorization server's native authorization endpoint supporting the **native App2App** profile

## {{RFC9396}} Authorization Details Type **native_callback_uri**

The protocol described in this document requires **User-Interacting App** to natively navigate end-user back to *Client App*, for which it requires *Client App's* **native_callback_uri**. 

To this end this document defines a new Authorization Details Type:

    {
       "type": "https://scheme.example.org/native_callback_uri",
       "locations": [
          "https://app.example.com/native_callback_uri"
       ]
    }

## Native App2App Profile

*Authorization servers* providing a **native_authorization_endpoint** MUST follow the Native App2App profile processing requirements:

* Accept the {{RFC9396}} Authorization Details Type: **https://scheme.example.org/native_callback_uri**.
* Forwarding the Authorization Details Type to *Downstream Authorization Servers*.
* Ensuring *Downstream Authorization Servers* support the Native App2App profile, otherwise respond to redirect_uri with error=native_app2app_unsupported.
* Redirect using HTTP 30x (and not using HTTP Form Post or embedded Javascript in HTML pages).
* Avoid challenging end-user with bot-detection such as CAPTCHAs when invoked without cookies.

## Flow Diagram
~~~ aasvg
{::include art/app2app-browserless-w-brokers.ascii-art}
~~~
{: #app2app-browserless-w-brokers title="Browser-less App2App across trust domains" }

- (1) *Client App* presents an authorization request to *Initial Authorization Server's* **native_authorization_endpoint**, including a **native_callback_uri** Authorization Details {{RFC9396}} RAR Type.
- (2) *Initial Authorization Server* returns an *authorization request url* for Downstream Authorization Server, including the **native_callback_uri** Authorization Details.
- (3) *Client App* seeks an app on the device claiming the obtained *authorization request url*, and if so proceeds to the next step. Otherwise it loops through invocations of obtained *authorization request urls*, processing HTTP 30x redirect responses, until a claimed url is reached.
- (4) *Client App* natively invokes *User-Interacting App*.
- (5) *User-Interacting App* authenticates user and authorizes the request.
- (6) *User-Interacting App* natively invokes **native_callback_uri** (overriding the request's redirect_uri), providing as a parameter the redirect_uri with its response parameters.
- (7) *Client App* loops through *Authorization Servers*, starting from the redirect_uri it received from the *User-Interacting App*. It calls any subsequent uri obtained as 30x redirect directive, until it reaches a location header pointing to itself (indicating its own redirect_uri).
- (8) *Client App* exchanges code for tokens and the flow is complete.

## Validation of native_callback_uri

Validation of **native_callback_uri** by *User-Interacting Authorization Server* and its App is RECOMMENDED, to mitiagte open redirection attacks.

A validating Authorization Server MAY use various mechanisms outside the scope of this document.
For example, validation using {{OpenID.Federation}} is possible:

* Strip url path from **native_callback_uri** (retaining the DNS domain).
* Add the url path /.well-known/openid-federation and perform trust chain resolution.
* Inspect Client's metadata for redirect_uri's and validate **native_callback_uri** is included.

## Protocol Flow {#protocol-flow}

### Client App calls Initial Authorization Server

Client App calls Initial Authorization Server's authorization_endpoint to initiate an authorization code flow and indicates App2App flow using the scope: **app2app**.

### Initial Authorization Server returns authorization request to Downstream Authorization Server

*Initial Authorization Server* SHALL process Client's request and return an HTTP 30x response with a Location header containing an authorization request url towards *Downstream Authorization Server's* authorization_endpoint.
The request SHALL include Client's redirect_uri as **native_callback_uri** in one of the methods specified in this document.

### Client App invokes app of User-Interacting Authorization Server

Client App SHALL use OS mechanisms to attempt to locate an app installed on the device claiming the authorization request url.
If such app is found, *Client App* SHALL natively invoke the app claiming the url to process the authorization request. This achieves the desired native navigation across applications.
If a suitable app is not found, *Client App* SHALL use HTTP to call the authorization request url and process the response:

* If the response code is HTTP 2xx, *Client App* cannot accomplish the browser-less flow and MUST fallback to using the browser. The reason is that it reached a *User-Interacting Authorization Server*, perhaps aiming to authenticate the user and authorize the request, or prompt the user for a routing decision or perhaps to redirect through HTTP Form-Post or Javascript. All of these are not compliant with the browser-less flow.
* If the response is a redirect instruction (HTTP Code 30x + Location header), *Client App* SHALL repeat the logic previously described:

  * Check if an app claims the url in the Location header, and if so natively invoke it.
  * Otherwise use HTTP to call the url and analyze the response.
* Client App SHALL handle error responses (HTTP 4xx / 5xx), for example display the error.

As the *Client App* traverses through Brokers, it SHALL maintain a list of all the DNS domains it traverses, to be used as Allowlist for response handling traversal.

As Authorization Servers MAY use Cookies to bind security elements (state, nonce, PKCE) to the user agent, causing the flow to break if said cookies are not present in subsequent HTTP requests, *Client App* MUST act as a browser would:

* Store Cookies it obtains in HTTP responses.
* Send Cookies in subsequent HTTP requests.

### Fallback to using the browser

If *Client App* obtains an HTTP 2xx response, a new flow MUST be initiated **using the browser**. It is NOT RECOMMENDED to relaunch the current flow's last authorization request on the browser as:

* Single-use elements such as *request_uri* might have been used in the request, and if so relaunching the request on the browser will fail.
* Authorization Servers previously engaged may have returned cookies to *Client App*, which would be unavailable to the browser and cause the flow to fail.

Therefore *Client App* MUST start a new flow by launching on the browser a new authorization request without the **app2app** scope, which then follows "OAuth 2.0 for Native Apps" {{RFC8252}} and does not require further elaboration in this document.

Note - In future interactions *Client App* MAY retry the *browser-less App2App* flow because past usage of the browser might have resulted from lack of *User-Interacting App* on the user's device.
Since in the meantime the app may have been installed, retrying may achieve the desired *browser-less App2App* flow.

### Processing by User-Interacting Authorization Server's App:

The *User-Interacting Authorization Server* SHALL handle the authorization request using its native app:

* Authenticates end-user and authorizes the request.
* SHALL use **native_callback_uri** to override the request's original redirect_uri:

  * *User-Interacting Authorization Server's app* validates that an app claiming **native_callback_uri** is on the device
  * If so it natively invokes **native_callback_uri** with the redirect url and its response parameters as the url-encoded query parameter **redirect_uri**.
  * If such an app does not exist on the device, the flow terminates and *User-Interacting Authorization Server's app* redirects to redirect_uri with:

    * error=invalid_request.
    * error_description=**native_callback_uri_not_claimed**.

### Client App traverses OAuth Brokers in reverse order

*Client App* is natively invoked by *User-Interacting Authorization Server App*, with the request's redirect_uri.

*Client App* MUST validate this url, and any url subsequently obtained via a 30x redirect instruction, against the Allowlist it previously generated, and MUST fail if any url is not included in the Allowlist.

*Client App* SHALL invoke the url it received using HTTP GET:

* If the response is a redirect instruction (HTTP Code 30x + Location header), *Client App* SHALL repeat the logic and proceed to call obtained urls until reaching its own redirect_uri (**native_callback_uri**).
* SHALL handle any other HTTP code (2xx / 4xx / 5xx) as a failure.

### Client App obtains response

Once *Client App's* own redirect_uri is returned in a redirect 30x directive, the traversal of OAuth Brokers is complete.

*Client App* SHALL proceed according to OAuth to exchange code for tokens, or handle error responses.

# Detecting Presence of Native Apps claiming Urls

Native Apps on iOS and Android MAY use OS SDK's to detect if an app owns a url.
The general method is the same - App calls an SDK to open the url as deep link and handles an exception thrown if no matching app is found.

## Android

App SHALL invoke Android {{android.method.intent}} method with FLAG_ACTIVITY_REQUIRE_NON_BROWSER, which throws ActivityNotFoundException if no matching app is found.

## iOS

App SHALL invoke iOS {{iOS.method.openUrl}} method with options {{iOS.option.universalLinksOnly}} which ensures URLs must be universal links and have an app configured to open them.
Otherwise the method returns false in completion.success.

# Security Considerations

## Embedded User Agents

{{RFC8252}} Security Considerations advises against using *embedded user agents*. The main concern is preventing theft through keystroke recording of end-user's credentials such as usernames and passwords.

This risk does not apply to this draft as *Client App* acts as User Agent only for the purpose of flow redirection, and does not interact with end-user's credentials in any way.

## OAuth request forgery and manipulation

It is RECOMMENDED that *Client App* acts as a confidential OAuth client.

## Secure Native application communication

If *Client App* uses a Backend it is RECOMMENDED to communicate with it securely:

* Use TLS in up to date versions and ciphers.
* Use DNSSEC.
* Perform certificate pinning.

## Deep link hijacking

It is RECOMMENDED that all apps in this specification shall use https-scheme deep links (Android App Links / iOS universal links). Apps SHOULD implement the most specific package identifiers mitigating deep link hijacking by malicious apps.

## Open redirection by Client App

Client App SHALL construct an Allowlist of DNS domains it traverses while processing the request, used to enforce all urls it later traverses during response processing.
This mitigates open redirection attacks as urls not in this Allowlist SHALL be rejected.

In addition *Client App* MUST ignore any invocation for response processing which is not in the context of a request it initiated.
It is RECOMMENDED the Allowlist be managed as a single-use object, destructed after each protocol flow ends.

It is RECOMMENDED *Client App* allows only one OAuth request processing at a time.

## Open redirection by User-Interacting Authorization Server's App

It is RECOMMENDED that *User-Interacting Authorization Server's* App establishes trust in **native_callback_uri** to mitigate open redirection attacks and reject untrusted urls.

## Authorization code theft and injection

It is RECOMMENDED that PKCE is used and that the code_verifier is tied to the *Client App* instance, as mitigation to authorization code theft and injection attacks.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments

The authors would like to thank the following individuals who contributed ideas, feedback, and wording that shaped and formed the final specification: George Fletcher, Arndt Schwenkschuster, Henrik Kroll, Grese Hyseni.
As well as the attendees of the OAuth Security Workshop 2025 session in which this topic was discussed for their ideas and feedback.

# Document History

\[\[ To be removed from the final specification ]]

-latest

* Phrased the challenge in Trust Domain terminology
* Discussed interim Authorization Server interacting the end-user, which is not the User-Authenticating Authorization Server
* Moved Cookies topic to Protocol Flow
* Mentioned that Authorization Servers redirecting not through HTTP 30x force the use of a browser
* Discussed Embedded user agents security consideration
* Starting to consider using a rich authorization details type as simpler container for native_callback_uri

-03

* Defined parameters and values
* Added error native_callback_uri_not_claimed

-02

* Clarified wording
* Improved figures

-01

* Better defined terms
* Explained deep link claiming detection on iOS and android

-00

* initial working group version (previously draft-zehavi-oauth-app2app-browserless)
