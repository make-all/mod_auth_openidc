Upstream
[![Build Status](https://travis-ci.org/zmartzone/mod_auth_openidc.svg?branch=master)](https://travis-ci.org/zmartzone/mod_auth_openidc)

mod_auth_openidc
================

*mod_auth_openidc* is an authentication/authorization module for the Apache 2.x
HTTP server that functions as an **OpenID Connect Relying Party**, authenticating users against an
OpenID Connect Provider.

> This version of the module can also authenticate against an OAuth2.0
> server using the Code Authorization Grant flow and a UserInfo
> endpoint. This has been tested with the Phabricator OAuthServer
> implementation.  Other servers implementing Oauth 2.0 Code
> Authorization Grant Flow, and which provide a suitable API for
> getting UserInfo as a JSON object may also work.  I tried to make
> these changes without breaking any existing functionality, but have
> not tested against any OpenID Connect servers, so it cannot be
> guaranteed.

> Note that the changes to support OAuth 2.0 Code Authorization Grant
> relax some checks, so if your server implements OpenID Connect, it
> is safer to use the upstream module from
> https://github.com/pingidentity/mod_auth_openidc 

> Use of this version is not recommended with public servers that are not
> completely under your control, as it may open possible attack
> vectors that may allow an attacker to impersonate a valid user using
> tokens issued for a different application.  Please read :
> https://oauth.net/articles/authentication/ and understand the risks
> before deploying this version of the module.

Overview
--------

This module enables an Apache 2.x web server to operate as an [OpenID Connect](http://openid.net/specs/openid-connect-core-1_0.html)
*Relying Party* (RP) to an OpenID Connect *Provider* (OP). It authenticates users against an OpenID Connect Provider,
receives user identity information from the OP in a so called ID Token and passes on the identity information
(a.k.a. claims) in the ID Token to applications hosted and protected by the Apache web server.

> If the server is a non-OpenID Connect server that implements the OAuth 2.0 Code
> Authorization Grant flow, the authentication can still be performed, with all user
> identity information coming from a UserInfo endpoint. In this case, you almost certainly
> need to set the OIDCOAuthRemoteUserClaim to something that is returned from the
> UserInfo endpoint.  Some additional options are provided for this case to allow more
> flexibility in the UserInfo endpoint implementation.
> OIDCProviderUserInfoResponseSubkey can be used to extract the claims from an object
> inside the JSON response, instead of extracting them directly from the response object.
> This was required to work with Phabricator's user.whoami API method, which returns
> user information inside a "result" object.

The protected content and/or applications can be served by the Apache server itself or it can be served from elsewhere
when Apache is configured as a Reverse Proxy in front of the origin server(s).

By default the module sets the `REMOTE_USER` variable to the `id_token` `[sub]` claim, concatenated with the OP's Issuer
identifier (`[sub]@[iss]`). Other `id_token` claims are passed in HTTP headers and/or environment variables together with those
(optionally) obtained from the UserInfo endpoint.

It allows for authorization rules (based on standard Apache `Require` primitives) that can be matched against the set
of claims provided in the `id_token`/ `userinfo` claims.

*mod_auth_openidc* supports the following specifications:
- [OpenID Connect Core 1.0](http://openid.net/specs/openid-connect-core-1_0.html) *(Basic, Implicit, Hybrid and Refresh flows)*
- [OpenID Connect Discovery 1.0](http://openid.net/specs/openid-connect-discovery-1_0.html)
- [OpenID Connect Dynamic Client Registration 1.0](http://openid.net/specs/openid-connect-registration-1_0.html)
- [OAuth 2.0 Multiple Response Type Encoding Practices 1.0](http://openid.net/specs/oauth-v2-multiple-response-types-1_0.html)
- [OAuth 2.0 Form Post Response Mode 1.0](http://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html)
- [RFC7 7636 - Proof Key for Code Exchange by OAuth Public Clients](https://tools.ietf.org/html/rfc7636)
- [OpenID Connect Session Management 1.0](http://openid.net/specs/openid-connect-session-1_0.html) *(implementers draft; see the [Wiki](https://github.com/zmartzone/mod_auth_openidc/wiki/OpenID-Connect-Session-Management) for information on how to configure it)*
- [OpenID Connect Front-Channel Logout 1.0](http://openid.net/specs/openid-connect-frontchannel-1_0.html) *(implementers draft)*
- [OpenID Connect Back-Channel Logout 1.0](https://openid.net/specs/openid-connect-backchannel-1_0.html) *(implementers draft)*
- [Encoding claims in the OAuth 2 state parameter using a JWT](https://tools.ietf.org/html/draft-bradley-oauth-jwt-encoded-state-08) *(draft spec)*
- [OpenID Connect Token Bound Authentication](https://openid.net/specs/openid-connect-token-bound-authentication-1_0.html) *(draft spec; when combined with [mod_token_binding](https://github.com/zmartzone/mod_token_binding))*
- [OAuth 2.0 Token Binding for Authorization Codes for Web Server Clients](https://tools.ietf.org/html/draft-ietf-oauth-token-binding-07#section-5.2) *(draft spec)*

For an exhaustive description of all configuration options, see the file `auth_openidc.conf`
in this directory. This file can also serve as an include file for `httpd.conf`.

Support
-------

#### Community Support
For generic questions, see the Wiki pages with Frequently Asked Questions at:  
  [https://github.com/zmartzone/mod_auth_openidc/wiki](https://github.com/zmartzone/mod_auth_openidc/wiki)  
There is a Google Group/mailing list at:  
  [mod_auth_openidc@googlegroups.com](mailto:mod_auth_openidc@googlegroups.com)  
The corresponding forum/archive is at:  
  [https://groups.google.com/forum/#!forum/mod_auth_openidc](https://groups.google.com/forum/#!forum/mod_auth_openidc)  
Any questions/issues should go to the mailing list.

#### Commercial Services
For commercial Support contracts, Professional Services, Training and use-case specific support you can contact:  
  [sales@zmartzone.eu](mailto:sales@zmartzone.eu)  

How to Use It  
-------------

### OpenID Connect SSO with Google+ Sign-In

Sample configuration for using Google as your OpenID Connect Provider running on
`www.example.com` and `https://www.example.com/example/redirect_uri` registered
as the *redirect_uri* for the client through the Google API Console. You will also
have to enable the `Google+ API` under `APIs & auth` in the [Google API console](https://console.developers.google.com).

```apache
OIDCProviderMetadataURL https://accounts.google.com/.well-known/openid-configuration
OIDCClientID <your-client-id-administered-through-the-google-api-console>
OIDCClientSecret <your-client-secret-administered-through-the-google-api-console>

# OIDCRedirectURI is a vanity URL that must point to a path protected by this module but must NOT point to any content
OIDCRedirectURI https://www.example.com/example/redirect_uri
OIDCCryptoPassphrase <password>

<Location /example/>
   AuthType openid-connect
   Require valid-user
</Location>
```

Note if you want to securely restrict logins to a specific Google Apps domain you would not only
add the `hd=<your-domain>` setting to the `OIDCAuthRequestParams` primitive for skipping the Google Account
Chooser screen, but you must also ask for the `email` scope using `OIDCScope` and use a `Require claim`
authorization setting in the `Location` primitive similar to:

```apache
OIDCScope "openid email"
Require claim hd:<your-domain>
```

The above is an authorization example of an exact match of a provided claim against a string value.
For more authorization options see the [Wiki page on Authorization](https://github.com/zmartzone/mod_auth_openidc/wiki/Authorization).

### Quickstart with a generic OpenID Connect Provider

1. install and load `mod_auth_openidc.so` in your Apache server
1. configure your protected content/locations with `AuthType openid-connect`
1. set `OIDCRedirectURI` to a "vanity" URL within a location that is protected by mod_auth_openidc
1. register/generate a Client identifier and a secret with the OpenID Connect Provider and configure those in `OIDCClientID` and `OIDCClientSecret` respectively
1. and register the `OIDCRedirectURI` as the Redirect or Callback URI with your client at the Provider
1. configure `OIDCProviderMetadataURL` so it points to the Discovery metadata of your OpenID Connect Provider served on the `.well-known/openid-configuration` endpoint
1. configure a random password in `OIDCCryptoPassphrase` for session/state encryption purposes

```apache
LoadModule auth_openidc_module modules/mod_auth_openidc.so

OIDCProviderMetadataURL <issuer>/.well-known/openid-configuration
OIDCClientID <client_id>
OIDCClientSecret <client_secret>

# OIDCRedirectURI is a vanity URL that must point to a path protected by this module but must NOT point to any content
OIDCRedirectURI https://<hostname>/secure/redirect_uri
OIDCCryptoPassphrase <password>

<Location /secure>
   AuthType openid-connect
   Require valid-user
</Location>
```
For details on configuring multiple providers see the [Wiki](https://github.com/zmartzone/mod_auth_openidc/wiki/Multiple-Providers).

### Quickstart for Other Providers

See the [Wiki](https://github.com/zmartzone/mod_auth_openidc/wiki) for configuration docs for other OpenID Connect Providers:
- [GLUU Server](https://github.com/zmartzone/mod_auth_openidc/wiki/Gluu-Server)
- [Keycloak](https://github.com/zmartzone/mod_auth_openidc/wiki/Keycloak)
- [Azure AD](https://github.com/zmartzone/mod_auth_openidc/wiki/Azure-OAuth2.0-and-OpenID)
- [Sign in with Apple](https://github.com/zmartzone/mod_auth_openidc/wiki/Sign-in-with-Apple)
- [Curity Identity Server](https://github.com/zmartzone/mod_auth_openidc/wiki/Curity-Identity-Server)
- [LemonLDAP::NG](https://github.com/zmartzone/mod_auth_openidc/wiki/LemonLDAP::NG)
- [GitLab](https://github.com/zmartzone/mod_auth_openidc/wiki/GitLab-OAuth2)
- [Globus](https://github.com/zmartzone/mod_auth_openidc/wiki/Globus)
and [more](https://github.com/zmartzone/mod_auth_openidc/wiki/Useful-Links)

###OAuth 2.0 Code Authorization Grant Flow

> Another example config for using a non-OpenID Connect server to perform
> authorization using the OAuth 2.0 Code Authorization Grant flow.  As no
> useful info is available for identifying the REMOTE_USER from plain OAuth 2.0,
> this relies on having a UserInfo endpoint to return at least a user id of
> some sort.  This need not be fully compliant with the OpenID spec for UserInfo
> endpoints, it just needs to return a JSON object.  The example server used here
> is a Phabricator server (https://phabricator.com/), which provides a simple
> OAuth 2.0 server implementation, and a user.whoami API method to obtain information
> about the logged in user (under a result object in the JSON response).

```apache
OIDCSSLValidateServer Off
OIDCClientID PHID-OASC-abcdefghijklmnopqrst
OIDCClientSecret abc123DEFghijklmnop4567rstuvwxyzZYXWUT8910SRQPOnmlijhoauthplaygroundapplication
OIDCProviderIssuer phabricator.example.com
OIDCRedirectURI https://localhost/example/redirect_uri/
OIDCCryptoPassphrase <password>
OIDCScope "userName realName primaryEmail"
OIDCProviderAuthorizationEndpoint https://phabricator.example.com/oauthserver/auth/
OIDCProviderTokenEndpoint https://phabricator.example.com/oauthserver/token/
OIDCProviderTokenEndpointAuth client_secret_post
OIDCProviderUserInfoEndpoint https://phabricator.example.com/api/user.whoami
OIDCUserInfoTokenMethod post_param
OIDCProviderUserInfoResponseSubkey result
OIDCORemoteUserClaim userName


<Location /example/>
   AuthType openid-connect
   Require valid-user
</Location>
```

Support
-------

> Some support resources below for the upstream project may help with any
> queries you have, but please feedback any issues related to OAuth 2.0
> Code Authorization Grant to https://github.com/make-all/mod_auth_openidc
> through the github issue tracker.

See the Wiki pages with Frequently Asked Questions at:  
  https://github.com/zmartzone/mod_auth_openidc/wiki   
There is a Google Group/mailing list at:  
  [mod_auth_openidc@googlegroups.com](mailto:mod_auth_openidc@googlegroups.com)  
The corresponding forum/archive is at:  
  https://groups.google.com/forum/#!forum/mod_auth_openidc  
Any questions/issues should go to the mailing list.

#### Commercial Services
For commercial support and consultancy you can contact:  
  [Globus](https://github.com/zmartzone/mod_auth_openidc/wiki/Globus)
and [more](https://github.com/zmartzone/mod_auth_openidc/wiki/Useful-Links)

Disclaimer
----------

*The upstream software is open sourced by ZmartZone IAM. For commercial support, please use the upstream version and
you can contact [ZmartZone IAM](https://www.zmartzone.eu) as described above in the [Support](#support) section.*

Any questions/issues for this fork should go to the Github issues tracker directly.
