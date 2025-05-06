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

This document, OAuth 2.0 App2App Browserless Flow (Native App2App), deals with applications from different issuers that act as OAuth 2.0 Client and Authorization Server, following the {{App2App}} pattern, to achieve native authentication and authorization.

This specification tackles the challenge when a Client App reaches the Authorization Server's app by redirecting through one or more brokering Authorization Servers, as it is not a direct client of the User-Authenticating Authorization Server.

Since such brokers perform redirection and federation using solely HTTP 3xx redirect instructions, no app on the users mobile device handles their urls as deep links and therefore App2App flows through brokers require using a web browser, which degrades the user experience.

This document presents a protocol enabling native App2App browser-less navigation, through any number of brokers, without compromising on any security property.

## Difference from {{OpenID.Native-SSO}}

{{OpenID.Native-SSO}} also offers a native SSO flow across applications without requiring the browser. However, it is dealing with the specific sub-case when both apps are published by the same issuer and leverage this fact to share information.

## Terminology

In addition to the terms defined in referenced specifications, this document uses
the following terms:

"OAuth":
: In this document, "OAuth" refers to OAuth 2.0, {{RFC6749}} and {{RFC6750}}.

This document is relevant for both {{RFC6749}} and {{OpenID}} as the protocols used, referring to both's **authorization code flow**.

For consistency and readability, it shall use OAuth 2.0 terminology - **Client** and **Authorization Server**, equally interchangeable with OpenID Connect **Relying Party** and **OpenID Provider** when OpenID Connect is used.

"PKCE":
: Proof Key for Code Exchange (PKCE) {{RFC7636}}, a mechanism
  to prevent various attacks on OAuth authorization codes.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Challenge of App2App with Brokers

## App2App with Brokers - Flow Diagram
~~~ aasvg
{::include art/app2app-w-brokers-and-browser.ascii-art}
~~~
{: #app2app-w-brokers-and-browser title="App2App with brokers and browser" }

## OAuth 2.0 / OpenID Connect Broker

A broker is a component acting as Authorization Server for its clients, as well as an OAuth 2.0 Client towards downstream Authorization Servers.
Brokers are used when there is no direct relation between an OAuth Client and the Authorization Server where the end-user authenticates and authorizes.
This is relevant in federation use cases, such as in Academia and in the business world connecting across subsidiaries or B2B relationships across corporations.
Some of the trust requirements may be solved in the future with {{OpenID.Federation}}, but federation through Brokers is an established pattern currently in broad use.

## App2App with brokers requires a web browser

Since OAuth 2.0 brokers reside on https domains, which no native app claims as Deep Links, OAuth requests to Brokers and responses to Broker's redirect_uri will be handled by a web browser.

## Impact of using a web browser

Using a web browser downgrades the user experience in several ways. The browser may be noticed by end-user as it is loading urls and redirecting to native apps.
The browser may prompt end-user for consent before opening deep links, introducing additional friction.
App developers have limited control as to which browser will be opened on the return redirect to the Broker, so any cookies used to bind session identifiers (nonce, state or pkce verifier) to the user agent may be lost, causing the flow to break.
Finally, the browser may be left after the flow ends with "orphan" browser tabs used for redirection. While these do not impact the process directly, they can be seen as clutter which degrades the overall UX's cleanliness.

# App2Web

Whenever the user's device has no app owning the User-Authenticating Authorization Server's urls as deep links, the flow requires the help of a browser.

This is the case when the User-Authenticating Authorization Server offers no native app, or when such an app exists but is not installed on the end-user's device.

This is similar to the flow described in {{RFC8252}}, and referred to in {{App2App}} as **App2Web**.

## App2Web with Brokers - Flow Diagram
~~~ ascii-art
┌───────────────────────────────────────────────────────────────────────────┐
│                                                                           │
│ ┌───────────┐                                                             │
│ │           │                                                             │
│ │   Client  │                        Mobile Browser                       │
│ │    App    │ ┌─────────────────────────────────────────────────────────┐ │
│ │           │ │                  ┌────────┐                             │ │
│ └───────┬───┘ │ ┌─────────┐      │┌────────┐        ┌─────────────────┐ │ │
│         └─────┼─► Primary ┼──────►│┌────────┐       │     User-       │ │ │
│  Authorization│ │ Broker  │Auth. └││──────┘├┼───────► Authenticating  │ │ │
│  Request      │ └─────────┘Req.   └│───────┘│ Auth. │  Authorization  │ │ │
│               │                    └────────┘ Req.  │     Server      │ │ │
│               │                    Secondary        └┬──────────────┬─┘ │ │
│               │                    Brokers           │     User-    │   │ │
│               │                                      │Authenticating│   │ │
│               │                                      │    Web UI    │   │ │
│               │                                      └──────────────┘   │ │
│               └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
                                    Mobile Device
~~~
Figure: App2Web with brokers

# Browser-less App2App with Broker

## Flow Diagram

## Components

### Client App

Native app acting as {{RFC6749}} client of Primary Broker.
Client's redirect_uri is claimed as a deep link of Client App.

### Client App's Primary Broker

A {{RFC6749}} Broker:

* Serving as Authorization Server for Client App
* An OAuth 2.0 client of one or more downstream authorization servers

The Primary Broker performs additional handling for browser-less App2App use-case, covered in {{protocol-flow}}.

### Downstream Authorization Server

An Authorization Server which may be a:

#### Secondary Broker

Additional brokers may be engaged in redirecting the flow, serving upstream brokers as OAuth 2.0 clients.
They do not perform user authentication and authorization.

#### User-Interacting Authorization Server

The Authorization Server which interacts with end-user to perform authentication and authorization.
May or may not offer App2App via a native app claiming it's urls as deep links. Such app may or may not be installed on end-user's device.

## Protocol flow {#protocol-flow}

### Client App calls Primary Broker

Client App calls Primary Broker's authorization_endpoint to initiate an authorization code flow, indicating App2App flow by use of a dedicated scope such as app2app.
Client App's redirect_uri is claimed as a deep link and will be referred to as client_app_deep_link.

### Primary Broker returns authorization request to Downstream Authorization Server

Primary Broker validates Client's request and prepares an authorization request to Downstream Authorization Server's authorization_endpoint.
Primary Broker provides client_app_deep_link to Downstream Authorization Server in the dedicated structured scope: app2app:*client_app_deep_link*.
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

#### Note - Downgrade to App2Web

### Processing by User-Interacting Authorization Server

### Client App traverses Brokers in reverse order

### Client App obtains response

# Security Considerations

## OAuth request forgery and manipulation

## Secure Native application communication

## Deep link hijacking

## Open redirection

## Authorization code theft and injection

## Handling of Cookies


# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank the attendees of the OAuth Security Workshop 2025 session in which this was discussed, as well as the following individuals who contributed ideas, feedback, and wording that shaped and formed the final specification:
