## Built-in HTTP Authentication

This page applies to inlets-pro 0.10.0 and onwards.

Services exposed over HTTP tunnels can have additional authentication added to them using a mechanism built-into inlets.

The `inlets-pro http` command provides three options:

1. Basic Authentication
2. Bearer Token Authentication
3. OAuth

## Basic Authentication

Basic authentication is a simple way to restrict access to your service by requiring a username and password.

When a user visits the URL of the service in a web-browser, they will be prompted to enter a username and password.

If they're using curl, then they can pass the username and password using the `--user` flag.

```bash
curl --user username:password https://example.com
```

Or simply pass the username and password as part of the URL:

```bash
curl https://username:password@example.com
```

The username has a default of `admin` for brevity, but can be overridden if you like:

```bash
inlets-pro http client \
    --basic-auth-username admin \
    --basic-auth-password password \
```

You can generate a string password using the `openssl` command:

```bash
openssl rand -base64 32
```

## Bearer Token Authentication

If you're exposing an endpoint that does not need to be accessed via a web-browser, then you can use Bearer Token Authentication.

This is useful for exposing endpoints that are used by mobile apps or other non-web clients.

To enable Bearer Token Authentication, you can use the `--bearer-token` flag when starting the `inlets-pro http` command.

```bash
inlets-pro http client \
    --bearer-token token \
```

Both Bearer Token and Basic Authentication can be used together by supplying both flags.

```bash
inlets-pro http client \
    --bearer-token token \
    --basic-auth-username admin \
    --basic-auth-password password \
```

## OAuth

With OAuth:

* You can define access for multiple users using usernames or email addresses
* Avoid managing credentials in your application
* Use an existing well-known provider for authentication such as GitHub

The OAuth 2.0 flow requires a web-browser, so if you anticipate mixed use, then you can combine it with Bearer Token Authentication, for headless clients.

### Example with GitHub.com

The example below will expose: `http://127.0.0.1:3000` using the domain name `tunnel.example.com`.

In order to use GitHub as the OAuth provider, you need to create a new OAuth application.

1. Go to [GitHub Developer Settings](https://github.com/settings/developers)
2. Click on "New OAuth App"
3. Enter a name for the application, for example `inlets-tunnel`
4. Enter the callback URL, for example `http://tunnel.example.com/_/oauth/callback`
5. Click on "Register application"
6. Click "Generate a new client secret"

You will be given a client ID and client secret, which you can use to authenticate with GitHub.

We suggest saving this in a convenient location, for example: `~/.inlets/oauth-client-id` and `~/.inlets/oauth-client-secret`.

```bash
inlets-pro http client \
    --oauth-client-id $(cat ~/.inlets/oauth-client-id) \
    --oauth-client-secret $(cat ~/.inlets/oauth-client-secret) \
    --upstream tunnel.example.com=http://127.0.0.1:3000 \
    --oauth-provider github \
    --oauth-acl alexellis \
    --oauth-acl name@example.com
```

Once authenticated, a cookie will be set on the domain i.e. `tunnel.example.com` and the user will be redirected back to the root URL of the service `/`.

The duration of the cookie defaults to 1 hour, but can be extended through the `--oauth-cookie-ttl` flag i.e.

```diff
inlets-pro http client \
+  --oauth-cookie-ttl 24h \
```

For the first version, GitHub is the only option available for the `--oauth-provider`. More options will be added over time, based upon requests from users, so if you want to use Google, Facebook, GitLab, etc, send us an email to help with prioritisation.


