Upstream [![Build Status](https://travis-ci.org/pingidentity/mod_auth_openidc.svg?branch=master)](https://travis-ci.org/pingidentity/mod_auth_openidc)

mod_auth_openidc
================

*mod_auth_openidc* is an authentication/authorization module for the Apache 2.x
HTTP server that functions as an **OpenID Connect Relying Party**, authenticating users against an
OpenID Connect Provider. It can also function as an **OAuth 2.0 Resource Server**, validating 
OAuth 2.0 access tokens presented by OAuth 2.0 Clients.

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
receives user identity information from the OP in a so called ID Token and passes the identity information
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

It can also be configured as an OAuth 2.0 *Resource Server* (RS), consuming bearer access tokens and introspecting/validating
them against a token introspection endpoint of an OAuth 2.0 Authorization Server and authorizing the Clients based on the
introspection results.

The protected content and/or applications can be served by the Apache server itself or it can be served from elsewhere
when Apache is configured as a Reverse Proxy in front of the origin server(s).

By default the module sets the `REMOTE_USER` variable to the `id_token` `[sub]` claim, concatenated with the OP's Issuer
identifier (`[sub]@[iss]`). Other `id_token` claims are passed in HTTP headers and/or environment variables together with those
(optionally) obtained from the UserInfo endpoint.

It allows for authorization rules (based on standard Apache `Require` primitives) that can be matched against the set
of claims provided in the `id_token`/ `userinfo` claims.

This module supports all defined OpenID Connect flows, including *Basic Client Profile*, *Implicit Client Profile*,
*Hybrid Flows* and the *Refresh Flow*. It supports connecting to multiple OpenID Connect Providers through reading/writing
provider metadata files in a specified metadata directory.

*mod_auth_openidc* supports the following specifications:
- [OpenID Connect](http://openid.net/specs/openid-connect-core-1_0.html)
- [OpenID Connect Dynamic Client Registration](http://openid.net/specs/openid-connect-registration-1_0.html)
- [OpenID Provider Discovery](http://openid.net/specs/openid-connect-discovery-1_0.html)
- [OAuth 2.0 Form Post Response Mode](http://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html)
- [Proof Key for Code Exchange by OAuth Public Clients](https://tools.ietf.org/html/rfc7636)
- [OpenID Connect Session Management](http://openid.net/specs/openid-connect-session-1_0.html). See the [Wiki](https://github.com/pingidentity/mod_auth_openidc/wiki/Session-Management) for information
on how to configure it.

Alternatively the module can operate as an OAuth 2.0 Resource Server to an OAuth 2.0 Authorization Server,
introspecting/validating bearer Access Tokens conforming to [OAuth 2.0 Token Introspection](https://tools.ietf.org/html/rfc7662) or similar.
The `REMOTE_USER` variable setting, passing claims in HTTP headers and authorization based on `Require` primitives
works in the same way as described for OpenID Connect above. See the [Wiki](https://github.com/pingidentity/mod_auth_openidc/wiki/OAuth-2.0-Resource-Server) for information
on how to configure it.

For an exhaustive description of all configuration options, see the file `auth_openidc.conf`
in this directory. This file can also serve as an include file for `httpd.conf`.

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
For more authorization options see the [Wiki page on Authorization](https://github.com/pingidentity/mod_auth_openidc/wiki/Authorization).

### OpenID Connect SSO with multiple OpenID Connect Providers

Sample configuration for multiple OpenID Connect providers, which triggers OpenID
Connect Discovery first to find the user's OP.

`OIDCMetadataDir` points to a directory that contains files that contain per-provider
configuration data. For each provider, there are 3 types of files in the directory:

1. `<urlencoded-issuer-value-with-https-prefix-and-trailing-slash-stripped>.provider`  
contains (standardized) OpenID Connect Discovery OP JSON metadata where each
name of the file is the url-encoded issuer name of the OP that is described
by the metadata in that file.

2. `<urlencoded-issuer-value-with-https-prefix-and-trailing-slash-stripped>.client`  
contains statically configured or dynamically registered Dynamic Client Registration
specific JSON metadata (based on the OpenID Connect Client Registration specification)
and the filename is the url-encoded issuer name of the OP that this client is registered
with. Sample client metadata for issuer `https://localhost:9031`, so the client metadata
filename is `localhost%3A9031.client`:

        {
            "client_id" : "ac_oic_client",
            "client_secret" : "abc123DEFghijklmnop4567rstuvwxyzZYXWUT8910SRQPOnmlijhoauthplaygroundapplication"
        }

3. `<urlencoded-issuer-value-with-https-prefix-and-trailing-slash-stripped>.conf`  
contains *mod_auth_openidc* specific custom JSON metadata that can be used to overrule
some of the settings defined in `auth_openidc.conf` on a per-client basis. The filename
is the URL-encoded issuer name of the OP that this client is registered with.

Entries that can be included in the .conf file are:

    "ssl_validate_server"                overrides OIDCSSLValidateServer (value 0 or 1...)
    "scope"                              overrides OIDCScope 
    "response_type"                      overrides OIDCResponseType 
    "response_mode"                      overrides OIDCResponseMode 
    "pkce_method"                        overrides OIDCPKCEMethod
    "client_name"                        overrides OIDCClientName 
    "client_contact"                     overrides OIDCClientContact 
    "idtoken_iat_slack"                  overrides OIDCIDTokenIatSlack
    "session_max_duration"               overrides OIDCSessionMaxDuration
    "jwks_refresh_interval"              overrides OIDCJWKSRefreshInterval
    "client_jwks_uri"                    overrides OIDCClientJwksUri
    "id_token_signed_response_alg"       overrides OIDCIDTokenSignedResponseAlg
    "id_token_encrypted_response_alg"    overrides OIDCIDTokenEncryptedResponseAlg
    "id_token_encrypted_response_enc"    overrides OIDCIDTokenEncryptedResponseEnc
    "userinfo_signed_response_alg"       overrides OIDCUserInfoSignedResponseAlg
    "userinfo_encrypted_response_alg"    overrides OIDCUserInfoEncryptedResponseAlg
    "userinfo_encrypted_response_enc"    overrides OIDCUserInfoEncryptedResponseEnc
    "auth_request_params"                overrides OIDCAuthRequestParams
    "token_endpoint_params"              overrides OIDCProviderTokenEndpointParams
    "token_endpoint_auth"                overrides OIDCProviderTokenEndpointAuth
    "registration_endpoint_json"         overrides OIDCProviderRegistrationEndpointJson
    "userinfo_refresh_interval"          overrides OIDCUserInfoRefreshInterval
    "userinfo_token_method"              overrides OIDCUserInfoTokenMethod
    "request_object"                     overrides OIDCRequestObject
    "auth_request_method"                overrides OIDCProviderAuthRequestMethod
    "registration_token"                 an access_token that will be used on client registration calls for the associated OP

Sample client metadata for issuer `https://localhost:9031`, so the *mod_auth_openidc*
configuration filename is `localhost%3A9031.conf`:

    {
        "ssl_validate_server" : 0,
        "scope" : "openid email profile"
    }
  
And the related *mod_auth_openidc* Apache config section:

```apache
OIDCMetadataDir <somewhere-writable-for-the-apache-process>/metadata

OIDCRedirectURI https://www.example.com/example/redirect_uri/
OIDCCryptoPassphrase <password>

<Location /example/>
   AuthType openid-connect
   Require valid-user
</Location>
```

If you do not want to use the internal discovery page (you really shouldn't...), you
can have the user being redirected to an external discovery page by setting
`OIDCDiscoverURL`. That URL will be accessed with a number parameters: `oidc_callback`, `target_link_uri`,
`method` and `x_csrf`. All parameters (except `oidc_callback`) need to be returned to the `oidc_callback` URL
together with an `iss` parameter that contains the URL-encoded issuer value of the selected Provider, or a
URL-encoded account name for OpenID Connect Discovery purposes (aka. e-mail style identifier), or a domain name.

Sample callback:

    <oidc_callback>?target_link_uri=<target_link_uri>&iss=[<issuer>|<domain>|<e-mail-style-account-name>][&login_hint=<name>][&scopes=<space-separated-scopes>][&auth_request_params=<urlencoded-query-string>]

This is also the OpenID Connect specified way of triggering 3rd party initiated SSO 
to a specific provider when multiple OPs have been configured. In that case the callback
may also contain a "login_hint" parameter with the login identifier the user might use to log in.

An additional *mod_auth_openidc* specific parameter named `auth_request_params` may also be passed
in, see the [Wiki](https://github.com/pingidentity/mod_auth_openidc/wiki#13-how-can-i-add-custom-parameters-to-the-authorization-request)
for its usage.

### OpenID Connect SSO & OAuth 2.0 Access Control with PingFederate

Another example config for using PingFederate as your OpenID Connect OP and/or
OAuth 2.0 Authorization server, based on the OAuth 2.0 PlayGround 3.x default
configuration and doing claims-based authorization. (running on `localhost` and
`https://localhost/example/redirect_uri/` registered as *redirect_uri* for the
client `ac_oic_client`)

```apache
OIDCProviderMetadataURL https://macbook:9031/.well-known/openid-configuration

OIDCSSLValidateServer Off
OIDCClientID ac_oic_client
OIDCClientSecret abc123DEFghijklmnop4567rstuvwxyzZYXWUT8910SRQPOnmlijhoauthplaygroundapplication

OIDCRedirectURI https://localhost/example/redirect_uri/
OIDCCryptoPassphrase <password>
OIDCScope "openid email profile"

OIDCOAuthIntrospectionEndpoint https://macbook:9031/as/token.oauth2
OIDCOAuthIntrospectionEndpointParams grant_type=urn%3Apingidentity.com%3Aoauth2%3Agrant_type%3Avalidate_bearer
OIDCOAuthIntrospectionEndpointAuth client_secret_basic
OIDCOAuthRemoteUserClaim Username
	
OIDCOAuthSSLValidateServer Off
OIDCOAuthClientID rs_client
OIDCOAuthClientSecret 2Federate

<Location /example/>
   AuthType openid-connect
   #Require valid-user
   Require claim sub:joe
</Location>

<Location /example-api>
   AuthType oauth20
   #Require valid-user
   Require claim Username:joe
   #Require claim scope~\bprofile\b
</Location>
```

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
  https://github.com/pingidentity/mod_auth_openidc/wiki   
There is a Google Group/mailing list at:  
  [mod_auth_openidc@googlegroups.com](mailto:mod_auth_openidc@googlegroups.com)  
The corresponding forum/archive is at:  
  https://groups.google.com/forum/#!forum/mod_auth_openidc  
For commercial support and consultancy you can contact:  
  [info@zmartzone.eu](mailto:info@zmartzone.eu)  

Any questions/issues should go to the mailing list or the
primary author [hans.zandbelt@zmartzone.eu](mailto:hans.zandbelt@zmartzone.eu).  
The Github issues tracker should be used only for bugs reports and feature requests.

Disclaimer
----------

*The upstream software is open sourced by Ping Identity but not supported commercially
by Ping Identity, see also the DISCLAIMER file in this directory.  For commercial support for the upstream code
you can contact [ZmartZone IAM](https://www.zmartzone.eu) as described above.*

Any questions/issues for this fork should go to the Github issues tracker directly.
