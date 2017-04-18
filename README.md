nginx-google-oauth
==================

Lua module to add Google OAuth to nginx.

# [Google OpenIDConnect](https://developers.google.com/identity/protocols/OpenIDConnect)

# Server flow

Make sure you set up your app in the API Console to enable it to use these protocols and authenticate your users. When a user tries to log in with Google, you need to:

1. [Create an anti-forgery state token](#1-create-an-anti-forgery-state-token)
2. [Send an authentication request to Google](#2-send-an-authentication-request-to-google)
3. [Confirm the anti-forgery state token](#3-confirm-the-anti-forgery-state-token)
4. [Exchange code for access token and ID token](#4-exchange-code-for-access-token-and-id-token)
5. [Obtain user information from the ID token](#5-obtain-user-information-from-the-id-token)
6. [Authenticate the user](#6-authenticate-the-user)

## 1. Create an anti-forgery state token

You must protect the security of your users by preventing request forgery attacks. The first step is creating a unique session token that holds state between your app and the user's client. You later match this unique session token with the authentication response returned by the Google OAuth Login service to verify that the user is making the request and not a malicious attacker. These tokens are often referred to as cross-site request forgery (CSRF) tokens.

One good choice for a state token is a string of 30 or so characters constructed using a high-quality random-number generator. Another is a hash generated by signing some of your session state variables with a key that is kept secret on your back-end.

```lua
-- TODO: The following is a draft to create an anti-forgery state token
-- https://github.com/openresty/lua-nginx-module#ngxhmac_sha1
local secret_key = ~30 random characters
local state = ngx.encode_base64(ngx.hmac_sha1(secret_key, cb_server_name .. email .. expires))
```

## 2. Send an authentication request to Google

The next step is forming an HTTPS `GET` request with the appropriate URI parameters. Note the use of HTTPS rather than HTTP in all the steps of this process; HTTP connections are refused. You should retrieve the base URI from the [Discovery document](https://developers.google.com/identity/protocols/OpenIDConnect#discovery) using the key `authorization_endpoint`. The following discussion assumes the base URI is https://accounts.google.com/o/oauth2/v2/auth.

```lua
-- TODO: retrieve base URI from the Google Discovery document
-- https://developers.google.com/identity/protocols/OpenIDConnect#discovery
local google_discovery_document_uri = "https://accounts.google.com/.well-known/openid-configuration"

local request = http.new()
request:set_timeout(7000)
local res, err = request:request_uri(google_discovery_document_uri, {
  method = "GET"
})
```

```json
{
 "issuer": "https://accounts.google.com",
 "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
 "token_endpoint": "https://www.googleapis.com/oauth2/v4/token",
 "userinfo_endpoint": "https://www.googleapis.com/oauth2/v3/userinfo",
 "revocation_endpoint": "https://accounts.google.com/o/oauth2/revoke",
 "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
 "response_types_supported": [
  "code",
  "token",
  "id_token",
  "code token",
  "code id_token",
  "token id_token",
  "code token id_token",
  "none"
 ],
 "subject_types_supported": [
  "public"
 ],
 "id_token_signing_alg_values_supported": [
  "RS256"
 ],
 "scopes_supported": [
  "openid",
  "email",
  "profile"
 ],
 "token_endpoint_auth_methods_supported": [
  "client_secret_post",
  "client_secret_basic"
 ],
 "claims_supported": [
  "aud",
  "email",
  "email_verified",
  "exp",
  "family_name",
  "given_name",
  "iat",
  "iss",
  "locale",
  "name",
  "picture",
  "sub"
 ],
 "code_challenge_methods_supported": [
  "plain",
  "S256"
 ]
}
```

For a basic request, specify the following parameters:

- `client_id`, which you obtain from the API Console.
- `response_type`, which in a basic request should be code. (Read more at response_type.)
- `scope`, which in a basic request should be openid email. (Read more at scope.)
- `redirect_uri` should be the HTTP endpoint on your server that will receive the response from Google. You specify this URI in the API Console.
- `state` should include the value of the anti-forgery unique session token, as well as any other information needed to recover the context when the user returns to your application, e.g., the starting URL. (Read more at state.)
- `login_hint` can be the user's email address or the sub string, which is equivalent to the user's Google ID. If you do not provide a login_hint and the user is currently logged in, the consent screen includes a request for approval to release the user’s email address to your app. (Read more at login_hint.)
- Use the `openid.realm` if you are migrating an existing application from OpenID 2.0 to OpenID Connect. For details, see Migrating off of OpenID 2.0.
- Use the `hd` parameter to optimize the OpenID Connect flow for users of a particular G Suite domain. (Read more at hd.)
- For more options see [Authentication URI Parameters](https://developers.google.com/identity/protocols/OpenIDConnect#authenticationuriparameters)

```lua
-- Example of a complete OpenID Connect authentication URI:
--
-- https://accounts.google.com/o/oauth2/v2/auth?
--  client_id=424911365001.apps.googleusercontent.com&
--  response_type=code&
--  scope=openid%20email&
--  redirect_uri=https://oauth2-login-demo.example.com/code&
--  state=security_token%3D138r5719ru3e1%26url%3Dhttps://oauth2-login-demo.example.com/myHome&
--  login_hint=jsmith@example.com&
--  openid.realm=example.com&
--  hd=example.com
--
-- TODO: support domain/whitelist/blacklist
local request = http.new()
request:set_timeout(7000)
local res, err = request:request_uri(google_discovery_document_uri, {
  method = "GET",
  body = ngx.encode_args({
    client_id     = 424911365001.apps.googleusercontent.com,
    response_type = code,
    scope         = "openid email",
    redirect_uri  = "https://oauth2-login-demo.example.com/code",
    state         = "TODO",
    login_hint    = "jsmith@example.com",
    openid.realm  = "example.com",
    hd            = "example.com",
  }),
  headers = {
    ["Content-type"] = "application/x-www-form-urlencoded"
  },
  ssl_verify = true,
})
```

# 3. Confirm anti-forgery state token

The response is sent to the redirect_uri that you specified in the request. All responses are returned in the query string, as shown below:

```
-- https://oa2cb.example.com/code?state=security_token%3D138r5719ru3e1%26url%3Dhttps://oa2cb.example.com/myHome&code=4/P7q7W91a-oMsCeLvIaQm6bTrgtp7
```

On the server, you must confirm that the state received from Google matches the session token you created in Step 1. This round-trip verification helps to ensure that the user, not a malicious script, is making the request.

```lua
local args = ngx.req.get_uri_args()
local want_state = ngx.encode_base64(ngx.hmac_sha1(secret_key, cb_server_name .. email .. expires))
local have_state = args["state"]

if want_state != have_state {
  ngx.exit(ngx.HTTP_FORBIDDEN)
}
```

## Installation

You can copy `access.lua` to your nginx configurations, or clone the
repository. Your installation of nginx must already be built with Lua
support, and you will need the ``json`` and ``luasec`` modules as well.

### Ubuntu

You will need to install the following packages.

```
lua5.1
liblua5.1-0
liblua5.1-0-dev
liblua5.1-sec-dev
liblua5.1-json
```

You will also need to download and build the following and link them
with nginx

```
ngx_devel_kit
lua-nginx-module
```

See ``/chef/source-lua.rb`` for a Chef recipe to install nginx and Lua
with all of the requirements.


## Configuration

Add the access controls in your configuration. Because oauth tickets will be
included in cookies (and you are presumably protecting something very 
important), it is strongly recommended that you use SSL.

```
server {
  server_name supersecret.net;
  listen 443;

  ssl on;
  ssl_certificate /etc/nginx/certs/supersecret.net.pem;
  ssl_certificate_key /etc/nginx/certs/supersecret.net.key;

  set $ngo_client_id "abc-def.apps.googleusercontent.com";
  set $ngo_client_secret "abcdefg-123-xyz";
  set $ngo_token_secret "a very long randomish string";
  set $ngo_secure_cookies "true";
  access_by_lua_file "/etc/nginx/nginx-google-oauth/access.lua";
}

```

The access controls can be configured using nginx variables. The supported
variables are:

- **$ngo_client_id** This is the client id key
- **$ngo_client_secret** This is the client secret
- **$ngo_token_secret** The key used to encrypt the session token stored in the user cookie.  Should be long & unguessable.
- **$ngo_domain** The domain to use for validating users when not using white- or blacklists
- **$ngo_whitelist** Optional list of authorized email addresses
- **$ngo_blacklist** Optional list of unauthorized email addresses
- **$ngo_callback_scheme** The scheme for the callback URL, defaults to that of the request (e.g. ``https``)
- **$ngo_callback_host** The host for the callback, defaults to first entry in the ``server_name`` list (e.g ``supersecret.net``)
- **$ngo_callback_uri** The URI for the callback, defaults to "/_oauth"
- **$ngo_debug** If defined, will enable debug logging through nginx error logger
- **$ngo_secure_cookies** If defined, will ensure that cookies can only be transfered over a secure connection
- **$ngo_css** An optional stylesheet to replace the default stylesheet when using the body_filter
- **$ngo_user** If set, will be populated with the OAuth username returned from Google (portion left of '@' in email)
- **$ngo_email_as_user** If set and $ngo_user is defined, username returned will be full email address

## Configuring OAuth Access

Visit https://console.developers.google.com. If you're signed in to multiple
Google accounts, be sure to switch to the one which you want to host the OAuth
credentials (usually your company's Apps domain). This should match
``$ngo_domain`` (e.g. "yourcompany.com").

From the dashboard, create a new project. After selecting that project, you
should see an "APIs & Auth" section in the left-hand navigation. Within that
section, select "Credentials". This will present a page in which you can
generate a Client ID and configure access. Choose "Web application" for the
application type, and enter all origins and redirect URIs you plan to use.

In the "Authorized Javascript Origins" field, enter all the protocols and
domains from which you plan to perform authorization 
(e.g. ``https://supersecret.net``), separated by a newline.

In the "Authorized Redirect URI", enter all of the URLs which the Lua module
will send to Google to redirect after the OAuth workflow has been completed.
By default, this will be the protocol, server_name and ``/_oauth`` (e.g.
``https://supersecret.net/_oauth``. You can override these defaults using the
``$ngo_callback_*`` settings.

After completing the form you will be presented with the Client ID and 
Client Secret which you can use to configure ``$ngo_client_id`` and 
``$ngo_client_secret`` respectively.

If you need to further limit access within your organization, you can use
``$ngo_whitelist`` and/or ``$ngo_blacklist``. Both should be formatted as
a space-separated list of allowed (whitelist) or rejected (blacklist) email
addresses. If either of these values are defined, the ``$ngo_domain`` will
not be used for validating that the user is authorized to access the protected
resource.

## Body filter

If you want visual confirmation of successful authentication, you can use the
``body_filter.lua`` script to inject a header into your web application. Your
nginx configuration should look something like this:

```
server {
  server_name supersecret.net;
  listen 443;

  set $ngo_client_id 'abc-def.apps.googleusercontent.com';
  set $ngo_client_secret 'abcdefg-123-xyz';
  set $ngo_token_secret 'a very long randomish string';
  access_by_lua_file "/etc/nginx/nginx-google-oauth/access.lua";

  location / {
    header_filter_by_lua "ngx.header.content_length = nil";
    body_filter_by_lua_file "/etc/nginx/nginx-google-oauth/body_filter.lua";

    proxy_set_header Accept-Encoding "";
    proxy_pass http://supersecret-backend;
  }
}

```

The ``header_filter_by_lua`` directive is required so that the 
``content_length`` header returned by the backend is stripped and re-calculated
after the body filter has been applied.

The ``Accept-Encoding`` directive is recommended in cases where the backend
may be returning a gzipped document, in which case nginx will not decompress
the document before sending it to the body filter.

The ``body_filter_by_lua_file`` directive causes all responses from the backend
to be routed through a lua script that will inject a div just after the opening
``<body>`` element. The div will take the form of:

```html
<div class="ngo_auth">
  <img src="google-oauth-profile-pic" />
  <span class="ngo_user">google-oauth-user-name</span>
  <span class="ngo_email">google-oauth-email</span>
  <a href="signout_uri">Signout</a>
</div>
```

If ``$ngo_css`` is defined, the default stylesheet will be overridden,
otherwise the stylesheet will be:

```css
<style>
  div.ngo_auth { width: 100%; background-color: #6199DF; color: white; padding: 0.5em 0em 0. 5em 2em; vertical-align: middle; margin: 0; }
  div.ngo_auth > img { width: auto; height: 2em; margin: 0 1em 0 0; padding: 0; }
  div.ngo_auth > span { color: white; }
  div.ngo_auth > span.ngo_user { font-weight: bold; margin-right: 1em; }
  div.ngo_auth > a { color: white; margin-left: 3em; }
</style>

```

The filter operates by performing a regular expression match on ``<body>``,
and so should act as a no-op for non-HTML content types. It may be necessary
to use the body filter only on a subset of routes depending on your application.

## Username variable

If you wish to pass the username returned from Google to an external FastCGI/UWSGI script, consider using the ``$ngo_user`` variable:

```
server {
  server_name supersecret.net;
  listen 443;

  ssl on;
  ssl_certificate /etc/nginx/certs/supersecret.net.pem;
  ssl_certificate_key /etc/nginx/certs/supersecret.net.key;

  set $ngo_client_id "abc-def.apps.googleusercontent.com";
  set $ngo_client_secret "abcdefg-123-xyz";
  set $ngo_token_secret "a very long randomish string";
  set $ngo_secure_cookies "true";
  access_by_lua_file "/etc/nginx/nginx-google-oauth/access.lua";

  set $ngo_user "unknown@unknown.com";

  include uwsgi_params;
  uwsgi_param REMOTE_USER $ngo_user;
  uwsgi_param AUTH_TYPE Basic;
  uwsgi_pass 127.0.0.1:3031;
}
```

If you wish the full email address returned from Google to be set as the username, set the ``$ngo_email_as_user`` variable to any non-empty value.

## Development

See `test/README.md`.

Bug reports and pull requests are [welcome](https://github.com/agoragames/nginx-google-oauth).

It can be useful to turn off [lua_code_cache](http://wiki.nginx.org/HttpLuaModule#lua_code_cache)
while you're iterating.

## Roadmap

- Add support for non-blocking sockets in obtaining an auth token
- Support auth token refresh and timeouts
- Continue support for Ubuntu but make imports work on other platforms as well
- 401 page that allows signing out and back in with a different account
- whitelist and blacklist is checked on every request

## Copyright

Copyright 2014 Aaron Westendorf

## License

MIT

## Thanks

This project wouldn't have gone beyond the idea stage without the excellent
example provided by [SeatGeek](http://chairnerd.seatgeek.com/oauth-support-for-nginx-with-lua/).

Thank you to @eschwim for some much-needed usability and security [fixes](https://github.com/agoragames/nginx-google-oauth/pull/4).
