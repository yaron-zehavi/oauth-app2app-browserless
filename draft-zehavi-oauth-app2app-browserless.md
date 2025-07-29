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

This document, *OAuth 2.0 App2App Browser-less Flow*, describes a protocol enabling native (**Browser-less**) app navigation of an {{App2App}} OAuth grant across *different Trust Domains*.

When Clients and Authorization Servers are located on *different Trust Domains*, authorization requests are routedusing federation, involving Authorization Servers acting as clients of *Downstream Authorization Servers*.

Such federation setups create trust networks, for example in Academia and in the business world across corporations.

However in {{App2App}} scenarios the web browser must serve as user-agent, because federating Authorization Servers url's are not claimed by any native app.

The use of web browsers in App2App flows, degrades the user experience somewhat.

This document specifies:

* A **Native App2App Profile** *Authorization Servers* SHOULD follow to support browser-less App2App flows.
* A new Authorization Server metadata property: **native_authorization_endpoint**, indicating an *Authorization Server* supports the **Native App2App Profile**.
* A new {{RFC9396}} Authorization Details Type: **https://scheme.example.org/native_callback_uri**.
* A new error code value: **native_app2app_unsupported**

## App2App across trust domains requires a web browser

~~~ aasvg
{::include art/app2app-w-brokers-and-browser-2.ascii-art}
~~~
{: #app2app-w-brokers-and-browser title="App2App across trust domains using browser" }

Since no native app claims the urls of redirecting Authorization Servers (*OAuth Brokers*), mobile Operating Systems default to using the system browser as the User Agent.

## Impact of using a web browser

Using a web browser may degrade the user experience in several ways:

* Some browser's support for deep links is limited by design, or by the settings used.
* Browsers may prompt end-user for consent before opening apps claiming deep links, introducing additional friction.
* Browsers are noticeable by end-users, rendering the UX less smooth.
* Client app developers don't control which browser the *User-Interacting App* uses to provide its response to redirect_uri. Opinionated choices pose a risk that different browsers will use, making necessary cookies used to bind session identifiers to the user agent (nonce, state or PKCE verifier) unavailable, which may break the flow.
* After flow completion, "orphan" browser tabs may remain. They do not directly impact the flow, but can be regarded as unnecessary "clutter".

## Relation to {{OpenID.Native-SSO}}

{{OpenID.Native-SSO}} also offers a native SSO flow across apps. However, it is limited to apps:

* Published by the same issuer, therefore can securely share information.
* Using the same Authorization Server.

## Relation to {{OAuth.First-Party}}

{{OAuth.First-Party}} also deals with native apps, but it MUST only be used by first-party applications, which is when the authorization server and application are controlled by the same entity, which is not true in the case described in this document.

While this document also discusses a mechanism for *Authorization Servers* to guide *Client App* in obtaining user's input to guide routing the request across trust domains, the {{OAuth.First-Party}} required high degree of trust between the authorization server and the client is not fulfilled.

# Protocol Overview

## Flow Diagram
~~~ aasvg
{::include art/app2app-browserless.ascii-art}
~~~
{: #app2app-browserless-w-brokers title="Browser-less App2App across trust domains" }

- (1) *Client App* presents an authorization request to *Authorization Server's* **native_authorization_endpoint**, including the *native_callback_uri* *Authorization Details Type*.
- (2) *Authorization Server* returns either a *native authorization request url* for Downstream Authorization Server which includes the original **native_callback_uri** Authorization Details, or a **Routing Instructions Response**.
- (3) *Client App* handles obtained *Routing Instructions Response* by prompting end-user and providing their response to *Authorization Server*, which then responds with a *native authorization request url*. *Client App* handles obtained *native authorization request urls* by seeking an app on the device claiming the url. If not found, *Client App* loops through invocations of obtained *native authorization request urls*, until a claimed url is reached.
- (4) Once a claimed url is reached *Client App* natively invokes *User-Interacting App*.
- (5) *User-Interacting App* authenticates end-user and authorizes the request.
- (6) *User-Interacting App* natively invokes **native_callback_uri**, providing as a parameter a url-encoded *redirect_uri* with its response parameters.
- (7) *Client App* invokes the obtained *redirect_uri*.
- (8) *Client App* calls any subsequent uri obtained as 30x redirect directive, until it reaches a location header to its own redirect_uri.
- (9) *Client App* exchanges code for tokens and the flow is complete.

## Usage and Applicability

This specification MUST NOT be used when *Client App* detects *Initial Authorization Server's* url is claimed by an app on the device.

In such case *Client App* SHOULD natively invoke the authorization request url.

## Authorization Server Metadata

This document introduces the following authorization server metadata {{RFC8414}} parameter to indicate it supports the *Native App2App Profile*.

**native_authorization_endpoint**:
: URL of the authorization server's native authorization endpoint.

## {{RFC9396}} Authorization Details Type *native_callback_uri*

The protocol described in this document requires **User-Interacting App** to natively navigate end-user back to *Client App*, for which it requires *Client App's* **native_callback_uri**.

To this end this document defines a new Authorization Details Type:

    {
       "type": "https://scheme.example.org/native_callback_uri",
       "locations": [
          "https://app.example.com/native_callback_uri"
       ]
    }

**locations** array MUST include exactly one instance.

## Native App2App Profile

*Authorization servers* providing a **native_authorization_endpoint** MUST follow the **Native App2App Profile's** requirements:

* Accept the {{RFC9396}} Authorization Details Type: **https://scheme.example.org/native_callback_uri**.
* Forward the Authorization Details Type to *Downstream Authorization Servers*.
* Ensure *Downstream Authorization Servers* it federates to, support the *Native App2App profile*, otherwise return error=native_app2app_unsupported.
* Redirect using HTTP 30x (avoid redirecting using HTTP Form Post or scripts embedded in HTML).
* Avoid challenging end-user with bot-detection such as CAPTCHAs when invoked without cookies.
* MAY provide *Routing Instructions Response*.

## Routing Instructions Response

*Authorization servers* supporting the *Native App2App profile*, but requiring end-user input to guide request routing, MAY provide a *Routing Instructions Response*.

Example prompting end-user for multiple-choice:

    HTTP/1.1 200 OK
    Content-Type: application/vnd.oauth.app2app.routing+json

    {
        "id": "request-identifier-1",
        "logo": "uri or base64-encoded logo of Authorization Server",
        "userPrompt": {
            "options": {
                "bank": {
                    "title": "Bank",
                    "description": "Choose your Bank",
                    "values": {
                        "bankOfSomething": {
                            "name": "Bank of Something",
                            "logo": "uri or base64-encoded logo"
                        },
                        "firstBankOfCountry": {
                            "name": "First Bank of Country",
                            "logo": "uri or base64-encoded logo"
                        }
                    }
                },
                "segment": {
                    "title": "Customer Segment",
                    "description": "Choose your Customer Segment",
                    "values": {
                        "retail": "Retail",
                        "smb": "Small & Medium Businesses",
                        "corporate": "Corporate",
                        "ic": "Institutional Clients"
                    }
                }
            }
        },
        "response": {
            "post": "url to POST to using application/x-www-form-urlencoded",
            "get": "url to use for a GET with query params"
        }
    }

Example prompting end-user for input entry:

    HTTP/1.1 200 OK
    Content-Type: application/vnd.oauth.app2app.routing+json

    {
        "id": "request-identifier-2",
        "logo": "uri or base64-encoded logo of Authorization Server",
        "userPrompt": {
            "inputs": {
                "email": {
                    "hint": "Enter your email address",
                    "title": "E-Mail",
                    "description": "Lore Ipsum"
                }
            }
        },
        "response": {
            "post": "url to POST to using application/x-www-form-urlencoded",
            "get": "url to use for a GET with query params"
        }
    }

*Client App* supporting *Routing Instructions Response* identifies the response as such using its Content-Type, then prompts end-user for their input:

: *logo* is OPTIONAL and used to brand the interaction and represent the Authorization Server.
: *userPrompt* MUST specify at least *options* or *inputs* and MAY specify both.
: *options* specifies 1..n multiple-choice prompts.
: *inputs* specifies free-form input.

*Client App* provides end-user's input using *response* which specifies HTTP GET or POST urls.
If provided, *Client App* includes "id" as interaction identifier.

Example *Client App* response following end-user multiple-choice:

    POST /native/routing HTTP/1.1
    Host: example.as.com
    Content-Type: application/x-www-form-urlencoded

    id=request-identifier-1
    &bank=bankOfSomething
    &segment=retail

Example *Client App* response following end-user input entry:

    POST /native/routing HTTP/1.1
    Host: example.as.com
    Content-Type: application/x-www-form-urlencoded

    id=request-identifier-2
    &email=end_user@example.as.com

## Protocol Flow

### Client App calls Authorization Server

Client App uses HTTP to call *Initial Authorization Server's* *native_authorization_endpoint* with an authorization request including the *native_callback_uri* Authorization Details {{RFC9396}} RAR Type.

### Authorization Server

*Authorization Server* evaluates the native authorization request.
It MAY return:

* Error *native_app2app_unsupported* in case the intended *Downstream Authorization Server* does not support the *Native App2App Profile*.
* HTTP 200 with a *Routing Instructions Response*, in case it requires user input to guide choosing a *Downstream Authorization Server*.
* HTTP 30x in case the *Downstream Authorization Server* is known and eligible, with a *native authorization request url* towards *Downstream Authorization Server* in the Location header.

### Client App processes the response

*Client App* SHALL terminate the protocol flow if an error response is obtained, or an HTTP 2xx response other than a *Routing Instructions Response*, or if it does not support a *Routing Instructions Response* it has obtained.

If a *Routing Instructions Response* was obtained and is supported, *Client App* interacts with end-user and provides their response to *Authorization Server*.

If an HTTP 30x redirect response was obtained, *Client App* SHALL use OS SDK's to locate an app claiming the url in the Location header, and if found SHALL natively invoke it to process the *native authorization request*.

If a suitable app is not found, *Client App* SHALL use HTTP to call the authorization request url and process the response as described herein.

Client App repeats these actions until a native app is reached, or an error occurs.

As the *Client App* performs HTTP calls, it SHALL maintain a list of all the DNS domains it interacts with, serving as an Allowlist for later invocations as part of the response handling.

As Authorization Servers MAY use Cookies to bind security elements (state, nonce, PKCE) to the user agent, causing the flow to break if required cookies are missing from subsequent HTTP requests, *Client App* MUST handle cookies:

* Store Cookies it obtains in HTTP responses.
* Send Cookies in subsequent HTTP requests.

### Processing by User-Interacting Authorization Server's App:

The *User-Interacting Authorization Server's* app handles the native authorization request:

* Validates the native authorization request as an OAuth authorization code request.
* Establishes trust in *native_callback_uri*, otherwise terminates the flow.
* Validates that an app claiming **native_callback_uri** is on the device, otherwise terminates the flow.
* Authenticates end-user and authorizes the request.
* MUST use **native_callback_uri** to invoke *Client App*, providing it the redirect url and its response parameters as the url-encoded query parameter **redirect_uri**.

### Client App traverses Authorization Servers in reverse order

*Client App* is natively invoked by *User-Interacting Authorization Server App*, with the request's redirect_uri.

*Client App* MUST validate this url, and any url subsequently obtained via a 30x redirect instruction, against the Allowlist it previously generated, and MUST fail if any url is not included in the Allowlist.

*Client App* SHALL invoke urls it received using HTTP GET:

* If the response is a redirect instruction (HTTP Code 30x + Location header), *Client App* SHALL proceed to call obtained urls until reaching its own redirect_uri.
* SHALL handle any other HTTP code (2xx / 4xx / 5xx) as a failure and terminate the flow.

### Client App obtains response

Once *Client App's* own redirect_uri is reached, the traversal of *Authorization Servers* is complete and *Client App* proceeds to process the response:

* Exchange code for tokens.
* Or handle errors obtained.

### Recovery from failed native App2App flows

~~~ aasvg
{::include art/app2web-w-brokers-2.ascii-art}
~~~
{: #app2web-w-brokers title="App2Web using the browser" }

The *Native App2App flow* described in this document MAY fail when:

* An error response is obtained.
* *Client App* doesn't support an obtained *Routing Instructions Response*.
* An HTTP 2xx response other than a *Routing Instructions Response* is obtained.

In case of such failures, *Client App* MAY recover by launching a new (non-native) authorization request on a web browser, in accordance with "OAuth 2.0 for Native Apps" {{RFC8252}}.

Note - Failure because an HTTP 2xx response, other than a *Routing Instructions Response* was obtained, suggests the *User-Interacting App* is not installed on end-user's device.
*Client App* MAY choose in future to retry the *Native App2App* flow, to benefit if in the meantime the missing app has been installed.

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

The mechanism for providing routing instructions MUST NOT be used to request end-user to provide any authentication credentials.

## Open redirection by Authorization Server's User-Interacting App

To mitigate open redirection attacks, trust establishment in *native_callback_uri* is RECOMMENDED by *User-Interacting App*.
Any federating *Authorization Server* MAY also wish to establish trust.

The sepcific trust establishment mechanisms are outside the scope of this document.
For example purposes, one way to validate could leverage {{OpenID.Federation}}:

* Strip url path from **native_callback_uri** (retaining the DNS domain).
* Add the url path /.well-known/openid-federation and perform trust chain resolution.
* Inspect Client's metadata for redirect_uri's and validate **native_callback_uri** is included.

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
* removed error native_callback_uri_not_claimed
* Added Routing Instructions Response
* Added native_authorization_endpoint and matching AS profile
* Added Authorization Details Type as container for native_callback_uri

-04

* Phrased the challenge in Trust Domain terminology
* Discussed interim Authorization Server interacting the end-user, which is not the User-Authenticating Authorization Server
* Moved Cookies topic to Protocol Flow
* Mentioned that Authorization Servers redirecting not through HTTP 30x force the use of a browser
* Discussed Embedded user agents security consideration

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
