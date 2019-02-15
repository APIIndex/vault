---
layout: "docs"
page_title: "JWT - Auth Methods"
sidebar_title: "JWT"
sidebar_current: "docs-auth-jwt"
description: |-
  The JWT auth method allows authentication using JWTs, with support for OIDC Discovery for key fetching
---

# JWT Auth Backend

The `jwt` auth backend can be used to authenticate with Vault using
[OIDC](https://en.wikipedia.org/wiki/OpenID_Connect) or by providing a [JWT](https://en .wikipedia.org/wiki/JSON_Web_Token).
The OIDC method will allow authentication via an configured OIDC provider using the user web browser.
This method may be initiated from the Vault UI or the command line. If a JWT is provided 
direction, it can be cryptographically verified using locally-provided keys, or, if
configured, an OIDC Discovery service can be used to fetch the appropriate keys.

## OIDC Authentication

### Redirect URIs
An important part of OIDC role configuration is properly configuring redirect URIs. This must be 
done both in Vault and with the OIDC provider, and these configurations must align. The 
redirect URIs are specified for a role via the `allowed_redirect_uris` parameter. There are 
different redirect URIs to configure for the Vault UI and `vault login -method=oidc` cases, so 
one or both will need to be set up depending on the installation.

#### CLI
If you plan to support authentication via `vault login -method=oidc`, a localhost redirect URI 
must be set. This can usually be: `http://localhost:8300/oidc/callback`. Logins via the CLI may 
specify a different listening port, and URI with this port must match one of the configured 
redirected URIs. These same "localhost" URIs must be added to the provider as well.

#### Vault UI
Logging in via the Vault UI requires a redirect URI of the form:
`https://{host:port}/ui/vault/auth/{path}/oidc/callback`

The "host:port must be correct for the Vault server, and "path" must match the path the JWT 
backend is mounted at (e.g. "oidc" or "jwt").
If [namespaces](https://www.vaultproject.io/docs/enterprise/namespaces/index.html) are being used, 
they must be added as query parameters, for example: 

`https://vault.example.com:8200/ui/vault/auth/oidc/oidc/callback?namespace=my_ns`
### OIDC Provider Configuration

The OIDC authentication flow has been successfully tested with a number of providers. A full 
guide to configuring OAuth/OIDC applications (in particular setting up claims) is beyond the scope
of this document, but brief overviews of how to set up an application is covered for some common 
providers.

#### Auth0
1. Select Create Application (Regular Web App).
1. Configure Allowed Callback URLs.
1. Copy client ID and secret.
1. If you see Vault errors involving signature, check the application's Advanced > OAuth settings
 and verify that that signing algorithm is "RS256".

#### Gitlab
1. Visit Settings > Applications.
1. Fill out name and Redirect URIs.
1. Making sure to select the "openid" scope.
1. Copy client ID and secret.

#### Google
Main reference: https://developers.google.com/identity/protocols/OAuth2

##### Setup

1. Visit the [Google API Console](https://console.developers.google.com).
1. Create or a select Project.
1. Create a new credential via Credentials > Create Credentials > OAuth Client ID
1. Configure the OAuth Consent Screen. Application Name is required. Save.
1. Select application type: "Web Application".
1. Configured Authorized Redirect URIs.
1. Save client ID and secret.

#### Okta

1. Make sure an Authorization Server has been created
1. Visit Applications > Add Application (Web)
1. Configure Login redirect URIs. Save.
1. Save client ID and secret.


## API

The JWT Auth Plugin has a full HTTP API. Please see the
[API docs](/api/auth/jwt/index.html) for more details.




## JWT Authentication

### Via the CLI

The default path is `/jwt`. If this auth method was enabled at a
different path, specify `-path=/my-path` in the CLI.

```text
$ vault write auth/jwt/login role=demo jwt=...
```

### Via the API

The default endpoint is `auth/jwt/login`. If this auth method was enabled
at a different path, use that value instead of `jwt`.

```shell
$ curl \
    --request POST \
    --data '{"jwt": "your_jwt", "role": "demo"}' \
    http://127.0.0.1:8200/v1/auth/jwt/login
```

The response will contain a token at `auth.client_token`:

```json
{
  "auth": {
    "client_token": "38fe9691-e623-7238-f618-c94d4e7bc674",
    "accessor": "78e87a38-84ed-2692-538f-ca8b9f400ab3",
    "policies": [
      "default"
    ],
    "metadata": {
      "role": "demo"
    },
    "lease_duration": 2764800,
    "renewable": true
  }
}
```

## Configuration

Auth methods must be configured in advance before users or machines can
authenticate. These steps are usually completed by an operator or configuration
management tool.


1. Enable the JWT auth method:

    ```text
    $ vault auth enable jwt
    ```

1. Use the `/config` endpoint to configure Vault with local keys or an OIDC Discovery URL. For the
list of available configuration options, please see the [API documentation](/api/auth/jwt/index.html).

    ```text
    $ vault write auth/jwt/config \
        oidc_discovery_url="https://myco.auth0.com/"
    ```

1. Create a named role:

    ```text
    vault write auth/jwt/role/demo \
        bound_subject="r3qX9DljwFIWhsiqwFiu38209F10atW6@clients" \
        bound_audiences="https://vault.plugin.auth.jwt.test" \
        user_claim="https://vault/user" \
        groups_claim="https://vault/groups" \
        policies=webapps \
        ttl=1h
    ```

    This role authorizes JWTs with the given subject and audience claims, gives
    it the `webapps` policy, and uses the given user/groups claims to set up
    Identity aliases.

    For the complete list of configuration options, please see the API
    documentation.

