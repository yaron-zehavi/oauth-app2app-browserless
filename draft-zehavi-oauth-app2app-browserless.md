---
title: "OAuth 2.0 App2App Browserless Flow"
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
  RFC9126:
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

This document describes a protocol connecting 2 native apps via the OAuth {{App2App}} pattern, to achieve native user navigation (no web browser required), regardless of any number of OAuth brokers federating the request across trust domains.

--- middle

# Introduction

This document, *OAuth 2.0 App2App Browserless Flow* (Native App2App), presents a protocol enabling native OAuth {{App2App}} **browser-less** navigation across apps.

It addresses the challenges presented when using a web browser to navigate through **one or more** Brokering Authorization Servers:

* Such *OAuth Brokers* are required when *Client App* is not an OAuth client of the *User-Interacting Authorization Server* and so they federate the flow across trust domains.
* Since no app owns *OAuth Brokers'* urls, App2App flows involving brokers require a web browser, which degrades the user experience.

This document specifies:

* A new parameter to the authorization endpoint: **native_callback_uri**.
* A new scope value: **app2app**.
* A new error_description value: **native_callback_uri_not_claimed**.

## Difference from OpenID.Native-SSO

{{OpenID.Native-SSO}} also offers a native SSO flow across apps. However, it is limited to apps:

* Published by the same issuer, therefore can securely share information.
* Using the same Authorization Server.

## Terminology

In addition to the terms defined in referenced specifications, this document uses
the following terms:

"OAuth":
: In this document, "OAuth" refers to OAuth 2.0, {{RFC6749}} and {{RFC6750}} as well as {{OpenID}}, both in their **authorization code flow**.

"PKCE":
: Proof Key for Code Exchange (PKCE) {{RFC7636}}, a mechanism
  to prevent various attacks on OAuth authorization codes.

"OAuth Broker":
: A component acting as an Authorization Server for its clients, as well as an OAuth Client towards *Downstream Authorization Servers*.
Brokers are used to facilitate a trust relationship when there is no direct relation between an OAuth Client and the final Authorization Server where end-user authenticates and authorizes.
This pattern is in current use to establish trust in federation use cases in Academia and in the business world, across corporations.
Note: It is possible OAuth Brokers will be superseded in the future by {{OpenID.Federation}}, offering dynamic trust establishment.

"Client App":
: A Native app implementing "OAuth 2.0 for Native Apps" {{RFC8252}} as an OAuth client of *Initial Authorization Server*. Client's redirect_uri is claimed by the app.

"Initial Authorization Server":
: The Authorization Server of *Client App* which acts as an *OAuth Broker* as an OAuth client of a *Downstream Authorization Server*.

"Downstream Authorization Server":
: An Authorization Server which may be an *OAuth Broker* or a *User-Interacting Authorization Server*.

"User-Interacting Authorization Server":
: The Authorization Server which interacts with end-user to perform authentication and authorization.

"User-Interacting App":
: Native App of *User-Interacting Authorization Server*.

"Deep Link":
: A url claimed by a native application.

"Native Callback uri":
: *Client App's* redirect_uri, claimed as a deep link. This deep link is invoked by *User-Interacting App* to natively return to *Client App*.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Challenge of App2App with OAuth Brokers

## App2App with OAuth Brokers requires a web browser

~~~ aasvg
{::include art/app2app-w-brokers-and-browser.ascii-art}
~~~
{: #app2app-w-brokers-and-browser title="App2App with brokers and browser (redirects back to Client not depicted)" }

Since no native app claims *OAuth Brokers'* urls, OAuth requests and redirect_uri responses to and from *OAuth Brokers* are handled by a web browser as the User Agent.

## Impact of using a web browser

Using a web browser may degrade the user experience in several ways:

* Some browser's support for deep links is limited by design, or by the settings used.
* The browser may prompt end-user for consent before opening deep links, introducing additional friction.
* Browser's loading of urls and redirecting may be noticeable by end-user, rendering the UX less smooth.
* App developers cannot control which browser will handle the response redirect_uri. This poses a risk that necessary cookies used to bind session identifiers to the user agent (nonce, state or PKCE verifier) will be unavailable, which may break the flow.
* After the flow is complete, "orphan" browser tabs might remain. While they do not directly impact the flow, they can be regarded as unnecessary "clutter".

# App2Web with brokers
~~~ aasvg
{::include art/app2web-w-brokers.ascii-art}
~~~
{: #app2web-w-brokers title="App2Web with brokers (redirects back to Client not depicted)" }

End-user's device may not have an app claiming *User-Interacting Authorization Server's* urls, when:

* *User-Interacting Authorization Server* offers no native app.
* Or such an app is offered, but is not installed on the end-user's device..

In such case interacting with *User-Interacting Authorization Server's* MUST use the browser as user agent.
This is similar to the flow described in "OAuth 2.0 for Native Apps" {{RFC8252}}, and referred to in {{App2App}} as **App2Web**, and is therefore not discussed further in this document.

# Browser-less App2App with OAuth Brokers

## Flow Diagram
~~~ aasvg
{::include art/app2app-browserless-w-brokers.ascii-art}
~~~
{: #app2app-browserless-w-brokers title="Browser-less App2App with Brokers" }

- (1) *Client App* presents an authorization request to *Initial Authorization Server*, indicating app2app flow using new scope value **app2app**.
- (2) *Initial Authorization Server* returns an authorization request for Downstream Authorization Server, including Client App's redirect_uri as **native_callback_uri**.
- (3) *Client App* checks if the returned authorization request url is claimed by an app on the device and if so proceeds to the next step. Otherwise it loops through Downstream Authorization Servers, calling their authorization endpoints and processing their HTTP 3xx redirect responses, until a url claimed by an app on the device is reached.
- (4) *Client App* natively invokes *User-Interacting App*.
- (5) *User-Interacting App* authenticates user and authorizes the request.
- (6) *User-Interacting App* natively invokes **native_callback_uri** (overriding the request's redirect_uri), providing as a parameter the redirect_uri with its response parameters.
- (7) *Client App* loops through Authorization Servers in reverse order, starting from the redirect_uri it received from the *User-Interacting App*. It calls the first redirect_uri and any subsequent uri obtained as 3xx redirect directive, until it obtains a location header indicating its own redirect_uri.
- (8) *Client App* exchanges code for tokens and the flow is complete.

## New Parameters and Values {#parameters}

The protocol described in this document requires *User-Interacting App* to natively navigate end-user back to Client App, for which it requires Client App's native_callback_uri.

Therefore this document defines new parameters and values.

"**app2app**":
: New scope value, used by *Client App* to request an app2app flow from *Initial Authorization Server*.

*Initial Authorization Server*, processing an app2app flow according to this document, MUST provide Client App's redirect_uri as Native Callback uri to *Downstream Authorization Server* using one of the following options:

  "**native_callback_uri**":
  : OPTIONAL. New authorization endpoint request parameter. When **native_callback_uri** is provided, structured scope **app2app:native_callback_uri** MUST NOT be provided.

  "**app2app:{*native_callback_uri*}**":
  : OPTIONAL. New structured scope value including the **app2app** flag as well as the Client's **native_callback_uri**, separated by a colon. When structured scope **app2app:{*native_callback_uri*}** is provided, **native_callback_uri** MUST NOT be provided.

**native_callback_uri** accepts the following query parameter when invoked by *User-Interacting Authorization Server's App*:

  "**redirect_uri**":
  : url-encoded OAuth redirect_uri with its response parameters.

*Downstream Authorization Server*, processing an app2app flow according to this document:

* MUST retain the **native_callback_uri** in downstream authorization requests created.
* MAY validate **native_callback_uri**.

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

*Initial Authorization Server* SHALL process Client's request and return an HTTP 3xx response with a Location header containing an authorization request url towards *Downstream Authorization Server's* authorization_endpoint.
The request SHALL include Client's redirect_uri as **native_callback_uri** in one of the methods specified in this document.

### Client App invokes app of User-Interacting Authorization Server

Client App SHALL use OS mechanisms to locate an app installed on the device claiming the authorization request url.
If so, *Client App* SHALL natively invoke the app claiming the url to process the authorization request. This achieves native navigation across applications.
If an app handling the authorization request url is not found, *Client App* SHALL use HTTP to call the authorization request url and process the response:

* If the response is successful (HTTP Code 2xx), it is assumed to be the User-Interacting Authorization Server. This means the *Client App* "over-stepped" and MUST downgrade to App2Web.
* If the response is a redirect instruction (HTTP Code 3xx + Location header), *Client App* SHALL repeat the logic previously described:

  * Check if an app owns the obtained url, and if so natively invoke it.
  * Otherwise use HTTP to call the obtained url and analyze the response.
* Handle error response (HTTP 4xx / 5xx) for example by displaying the error.

As the *Client App* traverses through Brokers, it SHALL maintain a list of all the DNS domains it traverses, which serves later as the Allowlist when traversing the response.

#### Downgrade to App2Web

If *Client App* reaches a *User-Interacting Authorization Server* but failed to locate an app claiming its urls, it may be impossible to relaunch the last authorization request on the browser as it might have included a single-use "OAuth 2.0 Pushed Authorization Requests" {{RFC9126}} request_uri which by now has been used and is therefore invalid.

In such a case the *Client App* MUST start over, generating a new authorization request without the **app2app** scope, which is then launched on the browser.
The remaining flow follows "OAuth 2.0 for Native Apps" {{RFC8252}} and is therefore not elaborated further in this document.

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

*Client App* MUST validate this url, and any url subsequently obtained via a 3xx redirect instruction, against the Allowlist it previously generated, and MUST fail if any url is not included in the Allowlist.

*Client App* SHALL invoke the url it received using HTTP GET:

* If the response is a redirect instruction (HTTP Code 3xx + Location header), *Client App* SHALL repeat the logic and proceed to call obtained urls until reaching its own redirect_uri (**native_callback_uri**).
* SHALL handle any other HTTP code (2xx / 4xx / 5xx) as a failure.

### Client App obtains response

Once *Client App's* own redirect_uri is returned in a redirect 3xx directive, the traversal of OAuth Brokers is complete.

*Client App* SHALL proceed according to OAuth to exchange code for tokens, or handle error responses.

# Detecting Presence of Native Apps claiming Urls

Native Apps on iOS and Android MAY use OS SDK's to detect if an app owns a url.
The general method is the same - App calls an SDK to open the url as deep link and handles an exception thrown if no matching app is found.

## Android

App SHALL invoke Android {{android.method.intent}} method with FLAG_ACTIVITY_REQUIRE_NON_BROWSER, which throws ActivityNotFoundException if no matching app is found.

## iOS

App SHALL invoke iOS {{iOS.method.openUrl}} method with options {{iOS.option.universalLinksOnly}} which ensures URLs must be universal links and have an app configured to open them.
Otherwise the method returns false in completion.success

# Security Considerations

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

It is RECOMMENDED that User-Interacting Authorization Server's App establishes trust in **native_callback_uri** to mitigate open redirection attacks and reject untrusted urls.

## Authorization code theft and injection

It is RECOMMENDED that PKCE is used and that the code_verifier is tied to the *Client App* instance.

## Handling of Cookies

It can be assumed that Authorization Servers will use Cookies to bind security elements (state, nonce, PKCE) to the user agent, which will break the flow if these cookies are not present in subsequent HTTP requests.

Therefore, *Client App* MUST handle Cookies:

* Store cookies it obtains on HTTP responses.
* Send cookies on subsequent HTTP requests to Authorization Servers that returned such cookies.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank the following individuals who contributed ideas, feedback, and wording that shaped and formed the final specification: Henrik Kroll, Grese Hyseni.
As well as the attendees of the OAuth Security Workshop 2025 session in which this topic was discussed for their ideas and feedback.
