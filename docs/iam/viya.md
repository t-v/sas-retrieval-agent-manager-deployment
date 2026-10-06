---
layout: default
title: SAS Viya SSO
parent: Identity and access
nav_order: 5
---

# Connecting Keycloak to SAS Viya

Sign in to SAS Retrieval Agent Manager with SAS Viya credentials, and let the
agent act as that user against Viya compute.

Unlike the [OpenID Connect](./openid.md) and [SAML](./saml.md) pages, which
describe manual Keycloak configuration, this integration is applied for you by
the Helm chart. It is idempotent and re-applies on every `helm upgrade`.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [What the chart configures](#what-the-chart-configures)
- [The SASLogon issuer URI](#the-saslogon-issuer-uri)
- [Group membership and admin access](#group-membership-and-admin-access)
- [Running without Viya administrator credentials](#running-without-viya-administrator-credentials)
- [Verifying](#verifying)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- SAS Retrieval Agent Manager installed, with ingress enabled and a resolvable
  `ingress.domain`.
- A reachable SAS Viya deployment.
- A SAS Viya administrator account (for example `sasboot`) able to create
  OAuth clients and write configuration. Needed only while the chart applies
  the configuration, not at runtime.

## Quick start

Create a secret holding the Viya administrator credentials:

```bash
kubectl create secret generic viya-admin \
  --namespace <ram-namespace> \
  --from-literal=username=sasboot \
  --from-literal=password='<password>'
```

Then enable the integration:

```yaml
integrations:
  viya:
    enabled: true
    url: https://viya.example.com
    namespace: viya                 # omit if Viya is in another cluster
    sso:
      enabled: true
      adminSecret: viya-admin
```

After `helm upgrade`, the RAM sign-in page offers a **SAS Viya** option.

## What the chart configures

On the SAS Viya side. Both steps authenticate with `sso.adminSecret`:

| Step | Value | Disable with |
|------|-------|--------------|
| Sets `issuer.uri` on the `sas.logon.jwt` configuration, then restarts `sas-logon-app` | `{viya.url}/SASLogon` | `sso.configureSASLogon: false` |
| Registers the OAuth client Keycloak federates to | `sso.clientId`, default `ram-client` | `sso.registerClient: false` |

On the Keycloak side. No additional credentials are required; Keycloak is
reached over its in-cluster service:

- Creates the `viya-oidc` identity provider. Every endpoint is taken from
  Viya's own discovery document, so the issuer Keycloak stores cannot drift
  from the one Viya stamps on its tokens.
- Adds the group mappers described [below](#group-membership-and-admin-access).
- Removes the `email verified` protocol mapper and hardcodes `emailVerified`,
  because Viya does not always send that claim and Keycloak otherwise refuses
  the login.
- Enables retrieval of the federated Viya token by the RAM client.
- Authorizes token exchange, so RAM mints a *fresh* Viya token on each call
  rather than reusing the copy captured at login, which Viya never refreshes.

On oauth2-proxy, adds the Viya issuer to `extra_jwt_issuers`, so a caller can
present a raw Viya bearer token instead of using a browser session. Disable
with `sso.acceptViyaTokens: false`.

## The SASLogon issuer URI

SASLogon ships with **no external issuer configured**. Until an administrator
sets one, every token it issues carries
`iss: http://localhost/SASLogon/oauth/token`, whatever the real hostname is.
Nothing outside that pod can dial `localhost`, so no service can fetch the
signing keys to verify a Viya token.

The resulting failures appear far from their cause: token exchange returns
`400`, and oauth2-proxy rejects raw tokens with `401`. Both look like a
misconfigured identity provider.

The chart therefore:

1. Sets the issuer when `sso.configureSASLogon` is true, which is the default.
2. Refuses to continue if the issuer still reads `localhost`, naming the fix,
   rather than building a federation that cannot work.
3. Warns if the issuer is valid but differs from `sso.issuerURI`, because
   `extra_jwt_issuers` is rendered from that value and would not match.

The change takes effect only when `sas-logon-app` restarts. Set
`integrations.viya.namespace` and the chart performs the restart; the Role it
creates in that namespace allows `get` and `patch` on the single
`sas-logon-app` Deployment, plus the read access needed to watch the rollout.
If the namespace is not set, the configuration is still written and the
install stops with instructions to restart and upgrade again.

## Group membership and admin access

Every user federated from Viya is placed in **both** the `Admin` and `User`
Keycloak groups, with no claim involved.

This is deliberate. Keycloak does not decide who is an administrator here.
The RAM API makes that decision at request time by asking Viya directly
whether the caller belongs to `SASAdministrators`, using the caller's own
token.

Earlier designs did gate the Keycloak group on a `SASAdministrators` signal
carried in the login token's `authorities` claim. That claim does not
reliably survive this login flow, and administrators were dropped from both
groups after an ordinary session refresh. Asking Viya directly is both
simpler and correct.

Change the group names with `sso.adminGroup` and `sso.userGroup` if your
realm uses different ones.

## Running without Viya administrator credentials

If the chart may not hold Viya administrator credentials, disable both
Viya-side steps:

```yaml
integrations:
  viya:
    sso:
      registerClient: false
      configureSASLogon: false
```

`helm install` then prints the OAuth client definition to register by hand,
including the exact redirect URI and the generated client secret. The chart
verifies the issuer rather than setting it, and still performs all Keycloak
configuration. Note that `autoapprove: true` on the client matters: without
it, SASLogon shows a consent screen on every login.

## Verifying

```bash
# What the chart did
kubectl logs -n <ram-namespace> job/<release>-viya-sso

# The issuer Viya actually serves. Must not be localhost
curl -sk https://viya.example.com/SASLogon/.well-known/openid-configuration \
  | jq -r .issuer

# oauth2-proxy trusts that issuer
kubectl get cm -n <ram-namespace> <release>-oauth2-proxy \
  -o jsonpath='{.data.oauth2-proxy\.cfg}' | grep extra_jwt_issuers
```

For an end-to-end check, have a real Viya process call RAM with a user token:
run a short `PROC PYTHON` block from SAS Studio that exchanges the injected
`SAS_SERVICES_TOKEN` for a user token and queries an agent. A `200` with an
answer confirms the whole path, including the Keycloak token exchange.

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Install stops, naming a `localhost` issuer | `configureSASLogon` is false and SASLogon was never configured |
| Install writes the issuer then stops | `integrations.viya.namespace` not set; restart `sas-logon-app`, then upgrade again |
| Consent screen on every login | OAuth client registered without `autoapprove: true` |
| `404` when RAM retrieves a Viya token, login fine | Keycloak lacks the `identity-brokering-api:v2` feature; RAM falls back to the older endpoint |
| No `token-exchange` permission available | Keycloak lacks the `token-exchange` or `admin-fine-grained-authz:v1` features |
| `401` from oauth2-proxy on a raw Viya token | Issuer mismatch between Viya and `sso.issuerURI` |
| Login works, but compute sessions hang | The user has no home directory on the launcher's NFS export |

The last item is outside the chart's control: creating those directories
requires root on the export, because each must be owned by the UID and GID
Viya assigned the user. SAS documents the procedure in
[KB 68620](http://support.sas.com/kb/68/620.html).
