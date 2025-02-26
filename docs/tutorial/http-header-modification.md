# HTTP Header Modification

This page applies to inlets-pro 0.10.0 and onwards.

When you expose a local service over a HTTP tunnel, it may need the headers of the request or those of the response to be modified.

The `inlets-pro http` command provides a mechanism to modify the headers of the request and response, without having to write an additional proxy to do so.

1. Add a header to the request
2. Add a header to the response
3. Remove a header from the request
4. Remove a header from the response

Each flag is provided in the format of either: `Header` or `Header: Value`, and can be provided multiple times to work on multiple headers.

## Add a header to the request

To add a header to the request, you can use the `--request-header-add` flag.

For instance, if you have a client that calls the exposed service, but needs to add a special header like `X-Special-Header` to the request, you can do the following:

```bash
inlets-pro http client \
    --request-header-add "X-Special-Header: 1234567890"
```

Multiple headers can be added at once:

```bash
inlets-pro http client \
    --request-header-add "X-Special-Header: 1234567890" \
    --request-header-add "X-Another-Header: 0987654321"
```

## Add a header to the response

To add a header to the HTTP response from the exposed service, you can use the `--response-header-add` flag.

For example, to add a header to the response from the server to enable CORS:

```bash
inlets-pro http client \
    --response-header-add "Access-Control-Allow-Origin: *"
```

## Remove a header from the request

If you do not have control over the request headers because they are being sent by a third-party service or application, you can remove the ones you do not want to send.

Here is how you could remove the User-Agent header for privacy or security reasons:

```bash
inlets-pro http client \
    --request-header-remove User-Agent
```

## Remove a header from the response

Let's say that you're using Caddy, and want to remove the `X-Served-By` header from the request for security or privacy reasons.

To remove a header from the request, you can use the `--request-header-remove` flag.

```bash
inlets-pro http client \
  --request-header-remove X-Served-By
```
