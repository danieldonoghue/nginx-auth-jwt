nginx-auth-jwt
==============

[JSON Web Token]: https://datatracker.ietf.org/doc/html/rfc7519
[JSON Web Signature]: https://datatracker.ietf.org/doc/html/rfc7515
[JSON Web Key Set]: https://datatracker.ietf.org/doc/html/rfc7517#section-5
[JOSE header]: https://datatracker.ietf.org/doc/html/rfc7515#section-4
[JWT claim]: https://datatracker.ietf.org/doc/html/rfc7519#section-4

This nginx module implements client authorization by validating the provided
[JSON Web Token][] (JWT) using the specified keys.

The module supports [JSON Web Signature][] (JWS).

The module can be used for
[OpenID Connect](http://openid.net/specs/openid-connect-core-1_0.html)
authentication.

The module supports using either [OpenSSL](http://www.openssl.org/) or [GnuTLS](https://www.gnutls.org) for its cryptographic needs. 

> This modules is heavenly inspired by the nginx original
> [http_auth_jwt_module](http://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html).

Dependency
----------

- [jansson](http://www.digip.org/jansson/) header and library.

- [OpenSSL](http://www.openssl.org/) header and library _(when building for OpenSSL)_.

- [GnuTLS](https://www.gnutls.org) v3.7.10 (or newer) header and library _(when building for a GunTLS)_.


Installation
------------

### Build install (OpenSSL)

``` sh
$ : "clone repository"
$ git clone https://github.com/kjdev/nginx-auth-jwt
$ cd nginx-auth-jwt
$ : "get nginx source"
$ NGINX_VERSION=1.x.x # specify nginx version
$ wget http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz
$ tar -zxf nginx-${NGINX_VERSION}.tar.gz
$ cd nginx-${NGINX_VERSION}
$ : "build module"
$ ./configure --add-dynamic-module=../
$ make && make install
```

### Build install (GnuTLS)

A default build for nginx-auth-jwt will build against OpenSSL. To build for GnuTLS, instead, you should follow these instructions...

``` sh
$ : "clone repository"
$ git clone https://github.com/kjdev/nginx-auth-jwt
$ : "configure build for GnuTLS"
$ sed -i 's/ngx_module_srcs="$openssl_srcs/ngx_module_srcs="$gnutls_srcs/' nginx-auth-jwt/config
$ sed -i 's/ngx_module_libs=$openssl_libs/ngx_module_libs=$gnutls_libs/' nginx-auth-jwt/config
$ : "get nginx source"
$ NGINX_VERSION=1.x.x # specify nginx version
$ wget http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz
$ tar -zxf nginx-${NGINX_VERSION}.tar.gz
$ cd nginx-${NGINX_VERSION}
$ : "build module"
$ ./configure --add-dynamic-module=../
$ make && make install
```

### Docker

``` sh
$ docker build -f Dockerfile.openssl -t nginx-auth-jwt .
$ : "app.conf: Create nginx configuration"
$ docker run -p 80:80 -v $PWD/app.conf:/etc/nginx/http.d/default.conf nginx-auth-jwt
```

> Github package: ghcr.io/kjdev/nginx-auth-jwt

To build a GnuTLS version of nginx-auth-jwt, replace `Dockerfile.openssl` with `Dockerfile.gnutls` above.


Supported Algorithms
--------------------

The module supports the following JSON Web
[Algorithms](https://www.iana.org/assignments/jose/jose.xhtml#web-signature-encryption-algorithms).

JWS algorithms:

- HS256, HS384, HS512
- RS256, RS384, RS512
- ES256, ES384, ES512
- ~~EdDSA (Ed25519 and Ed448 signatures)~~

Configuration
-------------

### Example

```
location / {
  auth_jwt          "closed site";
  auth_jwt_key_file conf/keys.json;
}
```

### Directives

```
Syntax: auth_jwt string [token=$variable] | off;
Default: auth_jwt off;
Context: server
Context: http, server, location
```

Enables validation of JSON Web Token. The specified string is used as a realm.
Parameter value can contain variables.

The optional `token` parameter specifies a variable that contains
JSON Web Token.
By default, JWT is passed in the `Authorization` header as a
[Bearer Token](https://datatracker.ietf.org/doc/html/rfc6750).
JWT may be also passed as a cookie or a part of a query string:

> ```
> auth_jwt "closed site" token=$cookie_auth_token;
> ```

The special value off cancels the effect of the auth_jwt directive inherited
from the previous configuration level.

```
Syntax: auth_jwt_claim_set $variable name ...;
Default: -
Context: http
```

Sets the `variable` to a JWT claim parameter identified by key names.
For arrays, the variable keeps a list of array elements separated by commas.

> ```
> auth_jwt_claim_set $jwt_audience aud;
> ```

```
Syntax: auth_jwt_key_file file [jwks | keyval];
Default: -
Context: http, server, location
```

Specifies a file for validating JWT signature.
Parameter value can contain variables.

Specify `jwks` (default) or `keyval` as the file format.

- jwks: [JSON Web Key Set][] format

- keyval: JSON in key-value format

  > Example: `{"kid": "-----BEGIN PUBLIC KEY-----\nxx.."}`

Several `auth_jwt_key_file` directives can be specified on the same level.

> ```
> auth_jwt_key_file conf/key.jwks;
> auth_jwt_key_file conf/keys.json keyval;
> ```

```
Syntax: auth_jwt_key_request uri [jwks | keyval];
Default: -
Context: http, server, location
```

Allows retrieving a key from a subrequest for validating JWT signature and
sets the URI where the subrequest will be sent to.
Parameter value can contain variables.

Specify `jwks` (default) or `keyval` as the key format.

- jwks: [JSON Web Key Set][] format

- keyval: JSON in key-value format

To avoid validation overhead, it is recommended to cache the key file:

> ```
> proxy_cache_path /data/nginx/cache levels=1 keys_zone=foo:10m;
>
> server {
>   ...
>   location / {
>     auth_jwt "closed site";
>     auth_jwt_key_request /jwks_uri;
>   }
>
>   location = /jwks_uri {
>     internal;
>     proxy_cache foo;
>     proxy_pass  http://idp.example.com/keys;
>   }
> }
> ```

Several `auth_jwt_key_request` directives can be specified on the same level.

> ```
> auth_jwt_key_request /jwks_uri;
> auth_jwt_key_request /public_key keyval;
> ```

```
Syntax: auth_jwt_validate_alg HS256 | HS384 | HS512 | RS256 | RS384 | RS512 | ES256 | ES384 | ES512;
Default: -
Context: http, server, location
```

Determines whether to validating JWT algorithm.

```
Syntax: auth_jwt_validate_exp on | off;
Default: auth_jwt_validate_exp on;
Context: http, server, location
```

Determines whether to validating the exp JWT claim.

```
Syntax: auth_jwt_validate_sig on | off;
Default: auth_jwt_validate_sig on;
Context: http, server, location
```

Determines whether to validating JWT signature.

```
Syntax: auth_jwt_leeway time;
Default: auth_jwt_leeway 0s;
Context: http, server, location
```

Sets the maximum allowable leeway to compensate clock skew
when verifying the exp JWT claims.

```
Syntax: auth_jwt_phase preaccess | access;
Default: auth_jwt_phase access;
Context: http, server, location
```

Specifies the phase to be processed.

> ACCESS phase is not executed when a call is made from a subrequest.
>
> In the case of a call from a subrequest, `auth_jwt_key_request` cannot
> be processed. (nested in-memory subrequest)

### Embedded Variables

The module supports embedded variables:

```
$jwt_header_<name>
```

returns the value of a specified [JOSE header][].

```
$jwt_claim_<name>
```

Returns the value of a specified [JWT claim][].
For arrays, the variable keeps a list of array elements separated by commas.

```
$jwt_claims
```

Returns the value of [JWT claim][] (JSON).

Example
-------

- [OpenID Connect Authentication](example/README.md)


TODO
----

- [ ] `auth_jwt_key_request` in subrequests (nested in-memory subrequest)
- Support algorithms
  - [ ] EdDSA (JWK key type: OKP)
