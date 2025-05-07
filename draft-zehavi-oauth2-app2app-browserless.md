---
title: "OAuth 2.0 App2App Browserless Flow"
abbrev: "Native OAuth App2App"
category: std

docname: draft-zehavi-oauth2-app2app-browserless-latest
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
  latest: "https://yaron-zehavi.github.io/oauth-app2app-browserless/draft-zehavi-oauth2-app2app-browserless.html"

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

--- abstract

This document defines a protocol enabling native apps from different app publishers, using the App2App pattern to act as OAuth Client And Authorization Server, native browserless user navigation.

The native experience is retained also when the Client uses any number of brokers to federate across trust networks, while retaining highest levels of security.

--- middle

# Introduction

This document, OAuth 2.0 App2App Browserless Flow (Native App2App), discusses applications from different issuers that act as OAuth 2.0 Client and Authorization Server, following the {{App2App}} pattern, to achieve native authentication and authorization.

This specification addresses the challenge arising when a Client App reaches the User-Authenticating Authorization Server's app through one or more brokering Authorization Servers.

Since OAuth Brokers's urls are not owned by any apps as deep links, App2App flows through brokers require using a web browser, which degrades the user experience.

This document presents a protocol enabling native App2App browser-less navigation, through any number of brokers, without compromising on any security property.

## Difference from {{OpenID.Native-SSO}}

{{OpenID.Native-SSO}} also offers a native SSO flow across applications without requiring the browser. However, it is dealing with the specific sub-case when both apps are published by the same issuer and leverage this fact to share information.

## Terminology

In addition to the terms defined in referenced specifications, this document uses
the following terms:

"OAuth":
: In this document, "OAuth" refers to OAuth 2.0, {{RFC6749}} and {{RFC6750}} as well as {{OpenID}} referring to their **authorization code flow**.

For consistency and readability, it shall use OAuth terminology - **Client** and **Authorization Server**, equally interchangeable with OpenID Connect **Relying Party** and **OpenID Provider** when OpenID Connect is used.

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
:A Broker redirecting the flow, which does not perform user authentication and authorization.

"User-Interacting Authorization Server":
: The Authorization Server which interacts with end-user to perform authentication and authorization. May or may not offer App2App via a native app claiming it's urls as deep links.
Such app may or may not be installed on end-user's device.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Challenge of App2App with Brokers

## App2App with Brokers - Flow Diagram
~~~ aasvg
{::include art/app2app-w-brokers-and-browser.ascii-art}
~~~
{: #app2app-w-brokers-and-browser title="App2App with brokers and browser" }

## App2App with brokers requires a web browser

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


Whenever the user's device does not have an app owning the User-Authenticating Authorization Server's urls as deep links, the flow requires the help of a browser.

This is the case when the User-Authenticating Authorization Server offers no native app, or when such an app exists but is not installed on the end-user's device.

This is similar to the flow described in {{RFC8252}}, and referred to in {{App2App}} as **App2Web**.

# Browser-less App2App with Broker

## Flow Diagram
~~~ aasvg
{::include art/app2app-browserless-w-brokers.ascii-art}
~~~
{: #app2app-browserless-w-brokers title="Browser-less App2App with Broker" }

## Protocol flow {#protocol-flow}

1. Client App calls Primary Broker
: Client App calls Primary Broker's authorization_endpoint to initiate an authorization code flow, indicating App2App flow by use of a dedicated scope such as app2app.
Client App's redirect_uri is claimed as a deep link and will be referred to as *client_app_deep_link*.

2. Primary Broker returns authorization request to Downstream Authorization Server
: Primary Broker validates Client's request and prepares an authorization request to Downstream Authorization Server's authorization_endpoint.
Primary Broker provides *client_app_deep_link* to Downstream Authorization Server in the dedicated structured scope: app2app:**client_app_deep_link**.
Primary Broker responds with HTTP 302 and the authorization request url towards Downstream Authorization Server in the Location header.

### Client App traverses Brokers with request

Client App uses OS mechanisms to detect if the authorization request URL it received is handled by an app installed on the device.

If so, Client App natively invokes the app to process the authorization request, achieving from the user's perspective native navigation across applications.

If an app handling the authorization request URL is not found, Client App natively calls the authorization request URL using HTTP GET and processes the response:

* If the response is successful (HTTP Code 2xx), it is probably the User-Interacting Authorization Server. This means the Client App "over-stepped" and needs to downgrade to App2Web.

* If the response is a redirect instruction (HTTP Code 3xx + Location header), a Secondary Broker was reached and Client App repeats the logic previously described:

  * Check if an app owns the obtained url, and if so natively invoke it.
  * Otherwise natively call the obtained url and analyze the response.
* Handles error response (HTTP 4xx / 5xx) for example by displaying the error.

As the Client App traverses through Brokers, it maintains a list of all the domains it traverses, which shall serve as the Allowlist when later traversing the response.

#### Secondary Brokers

Secondary Brokers engaged in the journey need to retain structured scope app2app:**client_app_deep_link** in downstream authorization requests they create.

#### Note - Downgrade to App2Web

If Client App reaches a User-Interacting Authorization Server with no app handling its urls, it may not be possible to relaunch the last authorization request URL on the browser as it might have included a single use request_uri which by now has been used and is therefore invalid.

In such a case the Client App needs to start over, generating a new authorization request without App2App indication.

This request is then launched on the browser.

The remaining flow follows {{RFC8252}} and is therefore not further elaborated in this document.

### Processing by User-Interacting Authorization Server

The User-Interacting Authorization Server processes the authorization request using its native app:

* Native app displays the UI for user authentication and authorization.

* The *client_app_deep_link* provided in the strcutured scope, overrides the request's original redirect_uri:

  * User-Interacting Authorization Server's native app validates that an app owning *client_app_deep_link* is on the device
  * If so it natively invokes it, handing it the redirect url with its response parameter
  * If such an app does not exist it is an error and the flow terminates
* To establish trust towards client_app_deep_link, User-Interacting Authorization Server shall use OpenID Federation:

  * Strips url path from *client_app_deep_link* (retaining the domain).
  * Adds /.well-known/openid-federation and performs trust chain resolution.
  * Inspects Client's metadata for redirect_uri's and validates *client_app_deep_link* is included.

### Client App traverses Brokers in reverse order

Client App is invoked by User-Interacting Authorization Server App with a url as parameter which is the request's redirect_uri.

Client App validates this url, and any url later obtained as a 3xx redirect instruction from the brokers it traverses, against the Allowlist it previously generated and fails if any url is not included in the Allowlist.

Client App invokes the url it received using HTTP GET:

* If the response is a redirect instruction (HTTP Code 3xx + Location header), Client App repeats the logic and proceeds to call obtained urls until reaching its own redirect_uri (*client_app_deep_link*).
* Otherwise (HTTP Code 2xx / 4xx / 5xx) is a failure.

### Client App obtains response

Once Client App's own redirect_uri is obtained in a redirect 3xx directive, Client App proceeds according to OAuth to exchange code for tokens or handle error responses.

# Security Considerations

## OAuth request forgery and manipulation

It is recommended Client App shall be a confidential OAuth client.

## Secure Native application communication

If Client App uses a Backend it is recommended to communicate with it securely:

* Use TLS recommended version and ciphers.
* Use DNSSEC.
* Perform certificate pinning.

## Deep link hijacking

It is recommended that all apps in this specification shall protect their deep links using Android universal links / iOS App Links including the most specific package identifiers to prevent deep link hijacking by malicious apps.

## Open redirection

Client App constructs an Allowlist of domains it traverses through while processing the request, for enforcing urls it shall later traverse through during response processing.
This serves to mitigate open redirection attacks as urls outside of this Allowlist will be rejected.

In addition Client App should ignore any invocation for response processing which is not in the context of a request it initiated.
One way to achieve this is by treating the Allowlist as a single-use object and destruct it after each protocol flow ends.

Client App should allow only one OAuth request processing at a time.

## Authorization code theft and injection

It is recommended that PKCE is used and that the code_verifier is tied to the Client App instance.

## Handling of Cookies

It can be assumed that Authorization Servers will use Cookies to bind security elements (state, nonce, PKCE) to the user agent, and will break if these cookies are later missing.

Therefore, Client App needs to handle Cookies as a web browser would:

* Store cookies it obtains on HTTP responses.
* Send cookies on subsequent HTTP requests to servers that returned cookies.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank the attendees of the OAuth Security Workshop 2025 session in which this was discussed, as well as the following individuals who contributed ideas, feedback, and wording that shaped and formed the final specification:
