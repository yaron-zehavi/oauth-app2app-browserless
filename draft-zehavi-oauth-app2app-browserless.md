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
  RFC8252:
  RFC8414:
  RFC9126:
  RFC9396:
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

This document describes a protocol allowing a *Client App* to obtain an OAuth grant from an *Authorization Server's Native App* using the {{App2App}} pattern, providing **native** app navigation user-experience (no web browser used), despite both apps belonging to different trust domains.

--- middle

# Introduction

This document describes a protocol enabling native (**Browser-less**) app navigation of an {{App2App}} OAuth grant across *different Trust Domains*.

When *Clients* and *Authorization Servers* are located on *different Trust Domains*, authorization requests traverse across domains using federation, involving *Authorization Servers* acting as clients of *Downstream Authorization Servers*.

Such federation setups create trust networks, for example in Academia and in the business world across corporations.

However in {{App2App}} scenarios the web browser must serve as user-agent, because federating Authorization Servers url's are not claimed by any native app.

The use of web browsers in App2App flows degrades the user experience somewhat.

This document specifies:

**native_authorization_endpoint**:
: A new Authorization Server endpoint and corresponding metadata property REQUIRED to support the browser-less App2App flow.

**native_callback_uri**:
: A new *native authorization request* parameter, specifying the deep link of *Client App*.

**native_app2app_unsupported**:
: A new error code value.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

In addition to the terms defined in referenced specifications, this document uses
the following terms:

**OAuth**:
: In this document, "OAuth" refers to OAuth 2.0, {{RFC6749}} in the **authorization code flow**.

**OAuth Broker**:
: An Authorization Server federating to other trust domains by acting as an OAuth Client of  *Downstream Authorization Servers*.

**Client App**:
: A Native app OAuth client of *Authorization Server*. In accordance with "OAuth 2.0 for Native Apps" {{RFC8252}}, client's redirect_uri is claimed by the app.

**Downstream Authorization Server**:
: An Authorization Server downstream of another *Authorization Server*. It may be an *OAuth Broker* or the *User-Interacting Authorization Server*.

**User-Interacting Authorization Server**:
: An Authorization Server which interacts with end-user. The interaction may be interim navigation (e.g: user input is required to guide where to redirect), or performs user authentication and request authorization.

**User-Interacting App**:
: Native App of *User-Interacting Authorization Server*.

**Deep Link**:
: A url claimed by a native application.

# Protocol

## Flow Overview
~~~ aasvg
{::include art/app2app-browserless.ascii-art}
~~~
{: #app2app-browserless-w-brokers title="Browser-less App2App across trust domains" }

- (1) *Client App* presents an authorization request to *Authorization Server's* **native_authorization_endpoint**, including a **native_callback_uri**.
- (2) *Authorization Server* returns either:
  - A *native authorization request url* for *Downstream Authorization Server*.
  - A request for end-user input to guide request routing.
  - A *deep link* url to *User-Interacting App*.
- (3) *Client App*:
  - Calls *native authorization request urls* it obtains, so long as such responses are obtained, until a *deep link* url to *User-Interacting App* is obtained.
  - Handles requests for end-user input by prompting end-user and providing their input to *Authorization Server*.
  - Handles *deep links*, by invoking the app claiming the url, if present on the device.
- (4) Once a *deep link* claimed on the device is obtained, *Client App* natively invokes *User-Interacting App*.
- (5) *User-Interacting App* authenticates end-user and authorizes the request.
- (6) *User-Interacting App* returns to *Client App* by natively invoking **native_callback_uri** and provides as a parameter the url-encoded *redirect_uri* with its response parameters.
- (7) *Client App* invokes the *redirect_uri* it obtained.
- (8) *Client App* calls any subsequent uris obtained until its own redirect_uri is obtained.
- (9) *Client App* exchanges code for tokens and the flow is complete.

## Authorization Server Metadata

This document introduces the following parameter as authorization server metadata {{RFC8414}}, indicating support of *Native App2App*:

**native_authorization_endpoint**:
: URL of the authorization server's native authorization endpoint.

## native_authorization_endpoint {#native-authorization-endpoint}

This is an OAuth authorization endpoint, interoperable with other OAuth RFCs.
It supports the following additional request parameters:

**native_callback_uri**:
: REQUIRED. *Client App's* deep link, to be invoked by *User-Interacting App*. When invoking *native_callback_uri*, it accepts the following parameter:

  **redirect_uri**:
  : REQUIRED. url-encoded redirect_uri from *User-Interacting App* responding to its OAuth client, including its respective response parameters.

The following additional requirements apply to native_authorization_endpoint, in line with common REST APIs:

* SHALL NOT use cookies.
* SHALL return Content-Type header with the value "application/json", and a JSON http body.
* SHALL NOT return HTTP 30x redirects.
* SHALL NOT respond with bot-detection challenges such as CAPTCHAs.

### Native Authorization Request

An OAuth authorization request, interoperable with other OAuth RFCs, which also includes the *native_callback_uri* parameter.

*Authorization servers* processing a *native authorization request* MUST also:

* Forward the *native_callback_uri* in their requests to *Downstream Authorization Servers*.
* Ensure that the *Downstream Authorization Servers* it federates to, offers a *native_authorization_endpoint*, otherwise return an error response with error code *native_app2app_unsupported*.

### Native Authorization Response

The authorization server responds with *application/json* and either 200 OK or 4xx/5xx.

#### Federating response

If the *Authorization Server* decides to federate to another party such as *Downstream Authorization Server* or its OAuth client, it responds with 200 OK and the following JSON response body:

action:
: REQUIRED. A string with the value "call" to indicate that *url* is to be called with HTTP GET.

url:
: REQUIRED. A string holding a native authorization request for *Downstream Authorization Server*, or redirect_uri of an OAuth client with a response.

Example:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "action": "call",
        "url": "https://next-as.com/auth/native",
    }

*Client App* SHALL add all DNS domains of *urls* it encounters during each flow to an Allowlist, used to validate urls in the response handling phase, after being invoked by the *User-Interacting Authorization Server' App*.

It then MUST make an HTTP GET request to the returned *url* and process the response as defined in this document.

#### Deep Link Response

If the *Authorization Server* wishes to authenticate the user and authorize the request, using its *User-Interacting App*, it responds with 200 OK and the following JSON response body:

action:
: REQUIRED. A string with the value "deep_link" to indicate that *url* is to be called with HTTP GET.

url:
: REQUIRED. A string holding the deep link url claimed by the *User-Interacting App*.

Example:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "action": "deep_link",
        "url": "uri of native authorization request handled by *User-Interacting App*",
    }

*Client App* MUST use OS mechanisms to invoke the deep link received in *url* and open the *User-Interacting Authorization Server's App*. If no app claiming the deep link is be found, *Client App* MUST terminate the flow and MAY attempt a non-native flow. See {{fallback}}.

#### Routing Response

If the *Authorization Server* requires user input to determine where to federate, it responds with 200 OK and the following JSON body:

id:
: OPTIONAL. A string holding an interaction identifier used by *Authorization Server* to link the response to the request.

action:
: REQUIRED. A string with the value "prompt" to indicate that the client app must prompt the user for input before proceeding.

logo:
: OPTIONAL. URL or base64-encoded logo of *Authorization Server*, for branding purposes.

userPrompt:
: REQUIRED. A JSON object containing the prompt definition. The following parameters MAY be used:

- options: OPTIONAL. A JSON object that defines a dropdown/select input with various options to choose from. Each key is the parameter name to be sent in the response and each value defines the option:

  - title: OPTIONAL. A string holding the input's title.
  - description: OPTIONAL. A string holding the input's description.
  - values: REQUIRED. A JSON object where each key is the selection value and each value holds display data for that value:

    - name: REQUIRED. A string holding the display name of the selection value.
    - logo: OPTIONAL. A string holding a URL or base64-encoded image for that selection value.
- inputs: OPTIONAL. A JSON object that defines an input field. Each key is the parameter name to be sent in the response and each value defines the input field:

  - title: OPTIONAL. A string holding the input's title.
  - hint: OPTIONAL. A string holding the input's hint that is displayed if the input is empty.
  - description: OPTIONAL. A string holding the input's description.

response:
: REQUIRED. A JSON object that holds the URL to which the user input MUST be sent. It only supports two keys, which are mutually exclusive:

* get: The corresponding value is the URL to use for a GET request with user input appended as query parameters.
* post: The corresponding value is the URL to use for a POST request with user input sent in the request body, as application/x-www-form-urlencoded.

Example of prompting end-user for 2 multiple-choice inputs:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "action": "prompt",
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

Example of prompting end-user for text input entry:

    HTTP/1.1 200 OK
    Content-Type: application/vnd.oauth.app2app.routing+json

    {
        "action": "prompt",
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

*Client App* MUST prompt the user according to the response received.
It then MUST send the user input to the response endpoint using the requested method including the interaction id, if provided.

Example of *Client App* response following end-user multiple-choice:

    POST /native/routing HTTP/1.1
    Host: example.as.com
    Content-Type: application/x-www-form-urlencoded

    id=request-identifier-1
    &bank=bankOfSomething
    &segment=retail

Example of *Client App* response following end-user input entry:

    POST /native/routing HTTP/1.1
    Host: example.as.com
    Content-Type: application/x-www-form-urlencoded

    id=request-identifier-2
    &email=end_user@example.as.com

#### Error Response

If *Authorization Server* encounters an error whose audience is its OAuth client, it returns 200 OK with the following JSON body:

action:
: REQUIRED. A string with the value "call" to indicate that *url* is to called with HTTP GET.

url:
: REQUIRED. A string holding the redirect_uri of the OAuth client, including the OAuth error.

Example:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "action": "call",
        "url": "https://previous-as.com/auth/redirect?error=...&error_description=...&iss=...&state=..."
    }

*Client App* MUST make an HTTP GET request to the returned *url* and process the response as defined in this document.

If *Authorization Server* encounters an error, that it cannot/or must not send to its OAuth client, it responds with 4xx/5xx and the following JSON body:

error:
: REQUIRED. The error code as defined in {{RFC6749}} and other OAuth RFCs.

error_description:
: OPTIONAL. The error description as defined in {{RFC6749}}.

Example:

    HTTP/1.1 500 OK
    Content-Type: application/json

    {
        "error": "native_app2app_unsupported",
    }

*Client App* SHOULD display an appropriate error message to the user and terminate the flow.
In case of *native_app2app_unsupported*, *Client App* MUST terminate the flow and MAY retry with a non-native flow. See {{fallback}}.

## User-Interacting Authorization Server's App

The *User-Interacting Authorization Server's* app handles the native authorization request:

* Validates the native authorization request.
* Establishes trust in *native_callback_uri*, otherwise terminates the flow.
* Validates that an app claiming *native_callback_uri* is on the device, otherwise terminates the flow.
* Authenticates end-user and authorizes the request.
* MUST use *native_callback_uri* to invoke *Client App*, providing it the redirect url and its response parameters as the url-encoded query parameter **redirect_uri**.

## Client App response handling

*Client App* is natively invoked by *User-Interacting Authorization Server App*.

If it is invoked with an *error* (and optional *error_description*) parmeter, or no parameter at all, it MUST terminate the flow.
It MUST ignore any unknown parameters.

If invoked with a url-encoded **redirect_uri** as parameter, the *Client App* MUST validate *redirect_uri*, and any url subsequently obtained, using the Allowlist it previously generated, and MUST terminate the flow if any url is not found in the Allowlist.

*Client App* SHALL invoke *redirect_uri*, and any validated subsequent urls received using HTTP GET.

**Authorization Servers** processing *Native App2App* MUST respond to redirect_uri invocations:

* According to the REST API guidelines specified in {{native-authorization-endpoint}}.
* Returning a JSON body instructing the next url to call.

Example:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "action": "call",
        "url": "redirect_uri of an OAuth Client, including response parameters",
    }

*Client App* MUST handle any other response (2xx with other content-types / 3xx / 4xx / 5xx) as a failure and terminate the flow.

## Flow completion

Once *Client App's* own redirect_uri is obtained, *Client App* processes the response:

* Exchanges code for tokens.
* Or handles errors obtained.

And the *Native App2App* flow is complete.

# Implementation Considerations

## Detecting Presence of Native Apps claiming Urls

Native Apps on iOS and Android MAY use OS SDK's to detect if an app owns a url.
The general method is the same - App calls an SDK to open the url as deep link and handles an exception thrown if no matching app is found.

See {{Appendix-A}} for more details.

## Recovery from failed native App2App flows {#fallback}

~~~ aasvg
{::include art/app2web-w-brokers-2.ascii-art}
~~~
{: #app2web-w-brokers title="App2Web using the browser" }

The *Native App2App flow* described in this document MAY fail when:

* An error response is obtained.
* Required *User-Interacting App* is not installed on end-user's device.

*Client App* MAY recover by launching a new (non-native) authorization request on a web browser, in accordance with "OAuth 2.0 for Native Apps" {{RFC8252}}.

Note - Failure because *User-Interacting App* is not installed on end-user's device, might succeed in future, if the missing app has been installed. *Client App* MAY choose if and when to retry the *Native App2App flow* after such a failure.

# Security Considerations

## Embedded User Agents

{{RFC8252}} Security Considerations advises against using *embedded user agents*. The main concern is preventing theft through keystroke recording of end-user's credentials such as usernames and passwords.

*Client App* when interacting with end-user to provide routing guiding input MUST NOT be used to request authentication credentials or any other sensitive information.

## Open redirection by Authorization Server's User-Interacting App

To mitigate open redirection attacks, trust establishment in *native_callback_uri* is RECOMMENDED by *User-Interacting App*.
Any federating *Authorization Server* MAY also wish to establish trust.

The specific trust establishment mechanisms are outside the scope of this document.
For example purposes only, one possible way to establish trust is {{OpenID.Federation}}:

* Strip url path from **native_callback_uri** (retaining the DNS domain).
* Add the url path /.well-known/openid-federation and perform trust chain resolution.
* Inspect Client's metadata for redirect_uri's and validate **native_callback_uri** is included.

## Open redirection by Client App

Client App SHALL construct an Allowlist of DNS domains it traverses while processing the request, used to enforce all urls it later traverses during response processing.
This mitigates open redirection attacks as urls not in this Allowlist SHALL be rejected.

In addition *Client App* MUST ignore any invocation for response processing which is not in the context of a request it initiated.
It is RECOMMENDED the Allowlist be managed as a single-use object, destructed after each protocol flow ends.

It is RECOMMENDED *Client App* allows only one OAuth request processing at a time.

## Deep link hijacking

It is RECOMMENDED that all apps in this specification shall use https-scheme deep links (Android App Links / iOS universal links). Apps SHOULD implement the most specific package identifiers mitigating deep link hijacking by malicious apps.

## Authorization code theft and injection

It is RECOMMENDED that PKCE is used and that the code_verifier is tied to the *Client App* instance, as mitigation to authorization code theft and injection attacks.

# Background and relation to other standards

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

# IANA Considerations

This document has no IANA actions.

--- back

# Appendix A - Detecting Presence of Native Apps claiming Urls on iOS and Android {#Appendix-A}

## iOS

App SHALL invoke iOS {{iOS.method.openUrl}} method with options {{iOS.option.universalLinksOnly}} which ensures URLs must be universal links and have an app configured to open them.
Otherwise the method returns false in completion.success.

## Android

App SHALL invoke Android {{android.method.intent}} method with FLAG_ACTIVITY_REQUIRE_NON_BROWSER, which throws ActivityNotFoundException if no matching app is found.

# Acknowledgments

The authors would like to thank the following individuals who contributed ideas, feedback, and wording that shaped and formed the final specification: George Fletcher, Arndt Schwenkschuster, Henrik Kroll, Grese Hyseni.
As well as the attendees of the OAuth Security Workshop 2025 session in which this topic was discussed for their ideas and feedback.

# Document History

\[\[ To be removed from the final specification ]]

-latest
* Replaced Authorization Details Type with new parameter
* native_authorization_endpoint as REST API - no cookies or HTTP 30x responses

-05
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
