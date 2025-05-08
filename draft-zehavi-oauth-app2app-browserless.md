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
# area: "Security"
# workgroup: "Web Authorization Protocol"
keyword:
 - native-apps
 - oauth
 - app2app
 - browserless
venue:
#  group: "Web Authorization Protocol"
#  type: "Working Group"
#  mail: "oauth@ietf.org"
#  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "yaron-zehavi/oauth-app2app-browserless"
  latest: "https://yaron-zehavi.github.io/oauth-app2app-browserless/draft-zehavi-oauth-app2app-browserless.html"

author:
 -
    fullname: Yaron Zehavi
    organization: Raiffeisen Bank International
    email: yaron.zehavi@rbinternational.com
 -
    fullname: Henrik Kroll
    organization: Raiffeisen Bank International
    email: henrik.kroll@rbinternational.com
 -
    fullname: Grese Hyseni
    organization: Raiffeisen Bank International
    email: grese.hyseni@rbinternational.com

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

This document describes a protocol enabling native apps from different app publishers, using the App2App pattern to act as OAuth Client And Authorization Server, native browser-less user navigation.

The native experience is retained also when the Client uses any number of brokers to federate across trust networks, while retaining highest levels of security.

--- middle

# Introduction

This document, OAuth 2.0 App2App Browserless Flow (Native App2App), discusses the {{App2App}} pattern where two applications act as OAuth Client and Authorization Server respectively, to natively authenticate and authorize an end-user.

It addresses the challenges introduced when a web browser is involved in OAuth flows through one or more Brokering Authorization Servers, which are needed because Client App is not an OAuth client of the User-Interacting Authorization Server.

Since no app owns OAuth Brokers' urls, App2App flows involving brokers require using a web browser, which degrades the user experience.

This document presents a protocol enabling native App2App **browser-less** navigation, through any number of brokers, without compromising on any security property.

## Difference from {{OpenID.Native-SSO}}

{{OpenID.Native-SSO}} also offers a native SSO flow across applications without requiring the browser. However, it is dealing with the specific sub-case when both apps are published by the same issuer and leverage this fact to share information.

## Terminology

In addition to the terms defined in referenced specifications, this document uses
the following terms:

"OAuth":
: In this document, "OAuth" refers to OAuth 2.0, {{RFC6749}} and {{RFC6750}} as well as {{OpenID}}, both in their **authorization code flow**.

"PKCE":
: Proof Key for Code Exchange (PKCE) {{RFC7636}}, a mechanism
  to prevent various attacks on OAuth authorization codes.

"OAuth Broker":
: A component acting as an Authorization Server for its clients, as well as an OAuth Client towards Downstream Authorization Servers.
Brokers are used to facilitate a trust relationship when there is no direct relation between an OAuth Client and the final Authorization Server where end-user authenticates and authorizes.
Brokers are an established pattern for establishing trust in federation use cases, such as in Academia and in the business world across corporations.
Brokers may be replaced in the future with dynamic trust establishment leveraging {{OpenID.Federation}}.

"Client App":
: Native app implementing {{RFC8252}} as OAuth client of Primary Broker, and whose redirect_uri is claimed as a deep link.

"Primary Broker":
: An OAuth Broker serving as Authorization Server of Client App.
Which is also an OAuth client of a Downstream Authorization Server.
Primary Broker performs additional handling for App2App use-case, covered in {{protocol-flow}}.

"Downstream Authorization Server":
: An Authorization Server which may be a *Secondary Broker* or a *User-Interacting Authorization Server*.

"Secondary Broker":
: A Broker redirecting the flow, which does not perform user authentication and authorization.

"User-Interacting Authorization Server":
: The Authorization Server which interacts with end-user to perform authentication and authorization. May or may not offer App2App via a native app claiming it's urls as deep links.
Such app may or may not be installed on end-user's device.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Challenge of App2App with Brokers

## App2App with brokers requires a web browser

~~~ aasvg
{::include art/app2app-w-brokers-and-browser.ascii-art}
~~~
{: #app2app-w-brokers-and-browser title="App2App with brokers and browser" }

Since OAuth Brokers reside on web domains which no native app claims as Deep Links, OAuth requests to Brokers and responses to Broker's redirect_uri will be handled by a web browser.

## Impact of using a web browser

Using a web browser downgrades the user experience in several ways. The browser may be noticed by end-user as it is loading urls and redirecting to native apps.

The browser may prompt end-user for consent before opening deep links, introducing additional friction.

App developers have limited control as to which browser will be opened on the return redirect to the Broker, so any cookies used to bind session identifiers (nonce, state or PKCE verifier) to the user agent may be lost, causing the flow to break.

Finally, the browser may be left after the flow ends with "orphan" browser tabs used for redirection. While these do not impact the process directly, they can be seen as clutter which degrades the overall UX's cleanliness.

# App2Web
~~~ aasvg
{::include art/app2web-w-brokers.ascii-art}
~~~
{: #app2web-w-brokers title="App2Web with brokers" }

When the user's device does not have an app owning the User-Authenticating Authorization Server's urls as deep links, the flow requires the help of a browser.

This is the case when the User-Authenticating Authorization Server offers no native app, or when such an app exists but is not installed on the end-user's device.

This is similar to the flow described in {{RFC8252}}, and referred to in {{App2App}} as **App2Web**.

# Browser-less App2App with Broker

## Flow Diagram
~~~ aasvg
{::include art/app2app-browserless-w-brokers.ascii-art}
~~~
{: #app2app-browserless-w-brokers title="Browser-less App2App with Broker" }

- (1) Client App uses HTTP to call Primary Broker's authorization endpoint with an authorization request including *app2app* scope.
- (2) Primary Broker prepares authorization request for Downstream Authorization Server including scope app2app:*client_app_deep_link*
- (3) Client App loops through Brokers, natively calling their authorization endpoint over HTTP using previously obtained authorization request urls, and processing their HTTP 3xx redirect directives, until a deep link owned by an app on the device is reached.
- (4) Client App natively invokes User-Authenticating App via its deep link.
- (5) User-Authenticating App authenticates user and authorizes the request. It identifies app2app mode and uses client_app_deep_link to override the request's redirect_uri.
- (6) User-Authenticating App natively invokes Client App using client_app_deep_link, handing it redirect_uri response.
- (7) Client App loops through Brokers, starting from the redirect_uri it received from User-Authenticating App. It natively calls them using HTTP traversing through their redirect directives, until Primary Broker redirects to Client App's own deep link.
- (8) Client App exchanges code for tokens.

## Protocol Flow {#protocol-flow}

### Client App calls Primary Broker

Client App calls Primary Broker's authorization_endpoint to initiate an authorization code flow, it SHALL indicate App2App flow using the dedicated scope app2app.

Client App's redirect_uri SHALL be claimed as a deep link and will be referred to as *client_app_deep_link*.

### Primary Broker returns authorization request to Downstream Authorization Server

* Primary Broker SHALL validate Client's request and prepare an authorization request to Downstream Authorization Server's authorization_endpoint.
* Primary Broker SHALL provide *client_app_deep_link* to Downstream Authorization Server as a suffix to the dedicated scope app2app. The combined scope is: app2app:**client_app_deep_link**.
* Primary Broker SHALL respond with HTTP 302 with the authorization request url towards Downstream Authorization Server in the Location header.

### Client App traverses Brokers with request

Client App SHALL use OS mechanisms to detect if the authorization request URL it received is handled by an app installed on the device.
If so, Client App SHALL natively invoke the app claiming the url to process the authorization request. This achieves native navigation across applications.
If an app handling the authorization request URL is not found, Client App SHALL natively call the authorization request URL using HTTP GET and processe the response:

* If the response is successful (HTTP Code 2xx), it is assumed to be the User-Interacting Authorization Server. This means the Client App "over-stepped" and MUST downgrade to App2Web.
* If the response is a redirect instruction (HTTP Code 3xx + Location header), a Secondary Broker was reached and Client App SHALL repeat the logic previously described:

  * Check if an app owns the obtained url, and if so natively invoke it.
  * Otherwise natively call the obtained url and analyze the response.
* Handle error response (HTTP 4xx / 5xx) for example by displaying the error.

As the Client App traverses through Brokers, it SHALL maintain a list of all the domains it traverses, which serves later as the Allowlist when traversing the response.

#### Secondary Brokers

Secondary Brokers engaged in the journey MUST retain structured scope app2app:**client_app_deep_link** in downstream authorization requests they create.

#### Downgrade to App2Web

If Client App reaches a User-Interacting Authorization Server with no app handling its urls, it may be impossible to relaunch the last authorization request URL on the browser as it might have included a single use request_uri which by now has been used and is therefore invalid.

In such a case the Client App MUST start over, generating a new authorization request without App2App indication, which is then launched on the browser.
The remaining flow follows {{RFC8252}} and is therefore not further elaborated in this document.

### Processing by User-Interacting Authorization Server:

The User-Interacting Authorization Server SHALL handle the authorization request using its native app:

* Native app authenticates end user and authorizes the request.
* The *client_app_deep_link* provided in the strcutured scope, SHALL override the request's original redirect_uri:

  * User-Interacting Authorization Server's native app SHALL validate that an app owning *client_app_deep_link* is on the device
  * If so it SHALL natively invoke it, handing it the redirect url with its response parameters
  * If such an app does not exist it is an error and the flow SHALL terminate

* To establish trust towards client_app_deep_link, User-Interacting Authorization Server MAY use mechanisms outside the scope of this document, or OpenID Federation:

  * SHALL strip url path from *client_app_deep_link* (retaining the domain).
  * SHALL add url path /.well-known/openid-federation and perform trust chain resolution.
  * SHALL inspect Client's metadata for redirect_uri's and validate *client_app_deep_link* is included.

### Client App traverses Brokers in reverse order

Client App is natively invoked by User-Interacting Authorization Server App, with the request's redirect_uri.

Client App MUST validate this url, and any url subsequently obtained through a 3xx redirect instruction from the brokers it traverses, against the Allowlist generated for this flow, and MUST fail if any url is not included in the Allowlist.

Client App SHALL invoke the url it received using HTTP GET:

* If the response is a redirect instruction (HTTP Code 3xx + Location header), Client App SHALL repeat the logic and proceed to call obtained urls until reaching its own redirect_uri (*client_app_deep_link*).
* SHALL handle any other HTTP code (2xx / 4xx / 5xx) as a failure.

### Client App obtains response

Once Client App's own redirect_uri is returned in a redirect 3xx directive, the traversal is complete.

Client App SHALL proceed according to OAuth to exchange code for tokens, or handle error responses.

# Detecting Presence of Native Apps Owning Urls

Native Apps on iOS and Android MAY use OS SDK's to detect if an app owns a url.
The general method is the same - App calls an SDK to open the url as deep link and handles an exception thrown if no matching app is found.

## Android

App SHALL invoke Android {{android.method.intent}} method with FLAG_ACTIVITY_REQUIRE_NON_BROWSER, which throws ActivityNotFoundException if no matching app is found.

## iOS

App SHALL invoke iOS {{iOS.method.openUrl}} method with options {{iOS.option.universalLinksOnly}} which ensures URLs must be universal links and have an app configured to open them.
Otherwise the method returns false in completion.success

# Security Considerations

## OAuth request forgery and manipulation

It is RECOMMENDED Client App acts as a confidential OAuth client.

## Secure Native application communication

If Client App uses a Backend it is RECOMMENDED to communicate with it securely:

* Use TLS in up to date versions and ciphers.
* Use DNSSEC.
* Perform certificate pinning.

## Deep link hijacking

It is RECOMMENDED that all apps in this specification shall protect their deep links using Android universal links / iOS App Links including the most specific package identifiers to prevent deep link hijacking by malicious apps.

## Open redirection

Client App SHALL construct an Allowlist of domains it traverses through while processing the request, used to enforce all urls it later traverses through during response processing.
This mitigates open redirection attacks as urls outside of this Allowlist will be rejected.

In addition Client App MUST ignore any invocation for response processing which is not in the context of a request it initiated.
It is RECOMMENDED the Allowlist is managed as a single-use object and destructed after each protocol flow ends.

It is RECOMMENDED Client App allows only one OAuth request processing at a time.

## Authorization code theft and injection

It is RECOMMENDED that PKCE is used and that the code_verifier is tied to the Client App instance.

## Handling of Cookies

It can be assumed that Authorization Servers will use Cookies to bind security elements (state, nonce, PKCE) to the user agent, and will break if these cookies are later missing.

Therefore, Client App MUST handle Cookies as a web browser would:

* SHALL store cookies it obtains on HTTP responses.
* SHALL send cookies on subsequent HTTP requests to servers that returned cookies.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank the attendees of the OAuth Security Workshop 2025 session in which this was discussed, as well as the following individuals who contributed ideas, feedback, and wording that shaped and formed the final specification:
