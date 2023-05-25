# Overview

* A terminal in the browser (yikes!),
* authenticated by OpenID Connect (ok, so hopefully MFA),
* provided with an ephemeral SSH certificate for onward connections (hmm, really?),
* that's issued from a private Certificate Authority using the same OIDC token (no key material lying around, alright!)

or

Use GitHub login for an emergency ~medical hologram~ jump shell

# Moving parts

* [WeTTY](https://github.com/butlerx/wetty) - "Terminal access in browser over http/https"
* [OAuth2Proxy](https://github.com/oauth2-proxy/oauth2-proxy) - "A reverse proxy and static file server that provides authentication using Providers"
* [Dex](https://github.com/dexidp/dex) - "A federated OpenID Connect provider"
* [GitHub](https://github.com/) - an OpenID provider with 2FA
* [Nginx](https://nginx.org) - a web server supporting Lua middleware
* [Step CA](https://smallstep.com/docs/step-ca/) - "an online Certificate Authority for secure, automated X.509 and SSH certificate management"

# Flow

* New user arrives at `/` and Nginx middleware calls out to OAuth2Proxy at
  `/oauth2/auth` to determine their OAuth2Proxy session state

* Without a valid session they are redirected to `/oauth2/sign_in`

* OAuth2Proxy offers to authenticate with its OpenID Connect provider, dex at
  `/dex`

* Dex authenticates the user using its OpenID provider, GitHub

* If successful Dex redirects back to the OAuth2Proxy at `/oauth2/callback`

* OAuth2Proxy completes the OAuth2 flow, receives the OIDC token from Dex,
  issues a session cookie and redirects back to `/`

* Nginx again calls out to OAuth2Proxy at `/oauth2/auth` to check the
  OAuth2Proxy session state

* A successful response from OAuth2Proxy allows Nginx to proxy the request to
  WeTTY setting `Remote-User` to the local user detemined from the OIDC

* WeTTY uses its private SSH key to log into the localhost as the authenticated
  user and starts the terminal session

* Meanwhile Nginx middleware passes the OIDC token provided by Dex as a
  `X-Auth-Request-Access-Token` header to a webhook that saves it in the user's
  `~/.ssh` directory

* A login script uses the stored OIDC token to obtain a SSH certificate from
  Step CA and loads it into the user's SSH agent 

* Onward SSH authenticates the user using the SSH certificate

# Deployment

Separate hosts for

* OAuth2Proxy and Dex
* Nginx, WeTTY and webhook
* Step CA

FreeBSD jails are a good fit.

# Notes

Step CA issues a SSH certificate for the principal in the OIDC token supplied
by a OIDC provider via OAuth2Proxy. In this setup that provider is Dex.
(Alternatives include Google Cloud Platform's authentication services tied to
a Google Workspace domain.)

Dex uses identity providers such as GitHub to authenticate the user. We use it
here since GitHub itself offers registered applications just OAuth2, not full
OIDC.

Step CA [discuss](https://smallstep.com/docs/step-ca/provisioners/#notes) using
its OIDC provisioner in situations where the redirect is not `localhost`, since
this implies the CA is network connected. The CA publishes its OAuth client
details at `/provisioners` which includes the client secret. Our CA *is*
network connected, though only on a private backend network between the Step CA
server and the WeTTY server.

This means that the OAuth client secret is visible to any local user on the
WeTTY server. A less trustful approach is to issue the SSH certificate on the
Step CA server itself and forward that SSH agent to the WeTTY server. Doable,
but fiddly.

OAuth2Proxy and Step CA use the same OAuth client details for Dex. This allows
Step CA to use the token issued to OAuth2Proxy.  Dex [provides a facility for
cross-client
tokens](https://dexidp.io/docs/custom-scopes-claims-clients/#cross-client-trust-and-authorized-party)
which in theory removes the need for reusing the client details, but these tokens
fail Step CA's audience checks.

Having WeTTY SSH into the user's account is a weird, but likely ok, solution to
not being able to instruct WeTTY to start a local session as a given user
without interactive authentication.
