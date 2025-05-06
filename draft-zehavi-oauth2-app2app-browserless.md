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
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - native apps
 - oauth
 - app2app
 - browserless
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "oauth-wg/oauth-app2app-browserless"
  latest: "https://drafts.oauth.net/oauth-app2app-browserless/draft-ietf-oauth-app2app-browserless.html"

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
      - ins: R. Hedberg
      - ins: M.B. Jones
      - ins: A.A. Solberg
      - ins: J. Bradley
      - ins: G. De Marco    
      - ins: V. Dzhuvinov
informative:
  App2App.Blog:
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

This document, OAuth 2.0 App2App Browserless Flow (Native App2App), deals with applications from different issuers that act as OAuth 2.0 Client and Authorization Server, following the {{App2App.Blog}} pattern, to achieve native authentication and authorization.

This specification aims at tackling the challenge when the Client App is not a direct client of the Authorization Server, but rather reaches it by redirecting through one or more brokering Authorization Servers.

Since such brokers fulfill their redirection and federation function without a user interface, no app on the users mobile device handles this traversal as deep links and therefore App2App flows through brokers necessitate using a web browser, which introduces several negative impacts.

This document presents a protocol enabling native App2App browser-less navigation, without compromising on any security property.

## Note
{{OpenID.Native-SSO}} also offers a native SSO flow across applications without requiring the browser. However it is dealing with the specific sub-case when both apps are published by the same issuer and leverage this fact to share information.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
