---
id: authentication
title: Authentication
---

# Authentication

## Modes

Frigate supports two modes for authentication

| Mode     | Description                                                                                                                                                                     |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `native` | (default) Use this mode if you don't implement authentication with a proxy in front of Frigate.                                                                                 |
| `proxy`  | Use this mode if you have an existing proxy for authentication. Supports passing authenticated user downstream to Frigate for role-based authorization (future implementation). |

### Native mode

Frigate stores user information in its database. Password hashes are generated using industry standard PBKDF2-SHA256 with 600,000 iterations. Upon successful login, a JWT token is issued with an expiration date and set as a cookie. The cookie is refreshed as needed automatically. This JWT token can also be passed in the Authorization header as a bearer token.

Users are managed in the UI under Settings > Users.

#### Onboarding

On startup, an admin user and password are generated and printed in the logs. It is recommended to set a new password for the admin account after logging in for the first time under Settings > Users.

#### Resetting admin password

In the event that you are locked out of your instance, you can tell Frigate to reset the admin password and print it in the logs on next startup using the `reset_admin_password` setting in your config file.

#### Login failure rate limiting

In order to limit the risk of brute force attacks, rate limiting is available for login failures. This is implemented with Flask-Limiter, and the string notation for valid values is available in [the documentation](https://flask-limiter.readthedocs.io/en/stable/configuration.html#rate-limit-string-notation).

For example, `1/second;5/minute;20/hour` will rate limit the login endpoint when failures occur more than:

- 1 time per second
- 5 times per minute
- 20 times per hour

Restarting Frigate will reset the rate limits.

If you are running Frigate behind a proxy, you will want to set `trusted_proxies` or these rate limits will apply to the upstream proxy IP address. This means that a brute force attack will rate limit login attempts from other devices and could temporarily lock you out of your instance. In order to ensure rate limits only apply to the actual IP address where the requests are coming from, you will need to list the upstream networks that you want to trust. These trusted proxies are checked against the `X-Forwarded-For` header when looking for the IP address where the request originated.

If you are running a reverse proxy in the same docker compose file as Frigate, here is an example of how your auth config might look:

```yaml
auth:
  mode: native
  failed_login_rate_limit: "1/second;5/minute;20/hour"
  trusted_proxies:
    - 172.18.0.0/16 # <---- this is the subnet for the internal docker compose network
```

### Proxy mode

Proxy mode is designed to complement common upstream authentication proxies such as Authelia, Authentik, oauth2_proxy, or traefik-forward-auth.

#### Header mapping

If your proxy supports passing a header with the authenticated username, you can use the `header_map` config to specify the header name so it is passed to Frigate. For example, the following will map the `X-Forwarded-User` value. Header names are not case sensitive.

```yaml
auth:
  ...
  header_map:
    user: x-forwarded-user
```

Note that only the following list of headers are permitted by default:

```
Remote-User
Remote-Groups
Remote-Email
Remote-Name
X-Forwarded-User
X-Forwarded-Groups
X-Forwarded-Email
X-Forwarded-Preferred-Username
X-authentik-username
X-authentik-groups
X-authentik-email
X-authentik-name
X-authentik-uid
```

If you would like to add more options, you can overwrite the default file with a docker bind mount at `/usr/local/nginx/conf/proxy_trusted_headers.conf`. Reference the source code for the default file formatting.

Future versions of Frigate may leverage group and role headers for authorization in Frigate as well.

#### Login page redirection

Frigate gracefully performs login page redirection that should work with most authentication proxies. If your reverse proxy returns a `Location` header on `401`, `302`, or `307` unauthorized responses, Frigate's frontend will automatically detect it and redirect to that URL.

#### Custom logout url

If your reverse proxy has a dedicated logout url, you can specify using the `logout_url` config option. This will update the link for the `Logout` link in the UI.
