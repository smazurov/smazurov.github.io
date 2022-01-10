---
title: "Server to Server Authentication with Google Provider in oauth2-proxy"
date: 2022-01-12T11:51:00-07:00
draft: false
tags: 
- jwt
- kubernetes
- auth
- google iam
---

At [Quantum Metric](https://quantummetric.com), we use very popular, and fantastic
[oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy). Since QM is a Google shop,
using Google OAuth2 Clients for authentication of our internal services makes a whole lot of sense.

One challenge with that, though, is server to server communication. There isn't a simple way to authenticate APIs
when using Google provider in `oauth2-proxy`. 

## Overview

In order to do server to server auth through oauth2-proxy when using Google Provider, you have to do the following:

1. Setup a new Google Service Account
1. Update OAuth2 Proxy settings
1. Exchange SA credentials for JWT token

## Google Service Account

[Creating a Service Account](https://cloud.google.com/iam/docs/creating-managing-service-accounts) is pretty standard fare, and no special permissions are needed for it. Grab the `service-account.json` file as you will need it to exchange for JWT Token

## Update OAuth2 Proxy settings

When deploying oauth2 proxy, in addition to [Google provider](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#google-auth-provider) settings you need to set the following options:

- `--oidc-issuer-url=https://accounts.google.com`, without this setting, as of v7.2.1, `oauth2-proxy` cannot validate jwt tokens when using
the Google provider.
- `--skip-jwt-bearer-tokens=true` - this tells oauth2 proxy not to do the exchange if `Authorization: Bearer ey....` header is set.

We also set the `--authenticated-emails-file=/path/to/file` setting, where you will need to add the SA email address that you created in the first step.

## Exchange SA credentials for JWT token

Last thing you will need to do is have a valid JWT token, which requires using the SA to get it from Google. Here's some code that gives you the token:

```bash
  #!/usr/bin/env bash
  set -euo pipefail

  get_token() {
    # Get the bearer token in exchange for the service account credentials.
    local service_account_key_file_path="${1}"
    local client_id="${2}"

    local iam_scope="https://www.googleapis.com/auth/iam"
    local oauth_token_uri="https://www.googleapis.com/oauth2/v4/token"

    local private_key_id="$(cat "${service_account_key_file_path}" | jq -r '.private_key_id')"
    local client_email="$(cat "${service_account_key_file_path}" | jq -r '.client_email')"
    local private_key="$(cat "${service_account_key_file_path}" | jq -r '.private_key')"
    local issued_at="$(date +%s)"
    local expires_at="$((issued_at + 3600))"
    local header="{'alg':'RS256','typ':'JWT','kid':'${private_key_id}'}"
    local header_base64="$(echo "${header}" | base64)"
    local payload="{'iss':'${client_email}','aud':'${oauth_token_uri}','exp':${expires_at},'iat':${issued_at},'sub':'${client_email}','target_audience':'${client_id}'}"
    local payload_base64="$(echo "${payload}" | base64)"
    local signature_base64="$(printf %s "${header_base64}.${payload_base64}" | openssl dgst -binary -sha256 -sign <(printf '%s\n' "${private_key}")  | base64)"
    local assertion="${header_base64}.${payload_base64}.${signature_base64}"
    local token_payload="$(curl -s \
      --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer" \
      --data-urlencode "assertion=${assertion}" \
      ${oauth_token_uri})"
    local bearer_id_token="$(echo "${token_payload}" | jq -r '.id_token')"
    echo "${bearer_id_token}"
  }

  main(){
    # TODO: Replace the following variables: 
    SERVICE_ACCOUNT_KEY="path-to-service-account.json"
    IAM_CLIENT_ID="<your client id>.apps.googleusercontent.com"

    # Obtain the Bearer ID token.
    ID_TOKEN=$(get_token "${SERVICE_ACCOUNT_KEY}" "${IAM_CLIENT_ID}")
    # Access the application with the Bearer ID token.
    echo ${ID_TOKEN}
  }

  main "$@"
```
While `get_token` looks complicated, its not. You can read more about it [here](https://developers.google.com/identity/protocols/oauth2/service-account#authorizingrequests). Key details are the `client_email` which must match allowed email address and `target_audience` which must match the client id in OAuth2 Proxy.

And thats it! Now, in addition to secure, short lived access for your users, you can have service accounts exchange their credentials for a valid JWT token. 