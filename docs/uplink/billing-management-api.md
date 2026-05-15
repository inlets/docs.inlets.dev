# Billing Management API

The Billing Management API is available to approved customers who need to update the billable quantity for a subscription from their own automation.

Each billing API key is tied to a specific subscription. If you have more than one subscription, request a separate API key for each subscription that needs automated quantity updates.

This API is separate from:

* your inlets Pro or Inlets Uplink license key
* the Inlets Uplink Client API used to create and manage tunnels
* any Kubernetes Secret used by the Uplink operator

The billing API uses a separate bearer token issued by the inlets team.

## Request access

API access requires a separate agreement and vetting. Once approved, the inlets team will issue access manually.

### Generate an encryption keypair

To request an API key, generate an encryption keypair and send us your public key:

```sh
inlets-pro seal keygen
```

The command writes a private key to `~/.inlets/seal.key` and a public key to `~/.inlets/seal.pub`.

The private key stays on your machine. Do not send it to anyone.

The command also prints the public key with a message similar to:

```text
Email the below to the inlets team:

inlets-seal-pub-v1:...
```

Email the printed public key to the inlets team when requesting API access.

### Unseal the API key

The inlets team will create a billing API key linked to your subscription, seal it with your public key, and send the sealed value back to you.

```sh
inlets-pro unseal < encrypted_key_you_received.txt
```

Put the unsealed API key in permanent storage such as a KMS, HashiCorp Vault, or a password manager, then add it to your automation.

After that, you can delete the seal keypair:

```sh
rm ~/.inlets/seal.key ~/.inlets/seal.pub
```

The seal keypair is only used to deliver an API key to you. Generate a new seal keypair if you need a replacement API key, a rotated API key, or an API key for a different subscription.

## Authentication

Use the unsealed API key as a bearer token. Your product license key remains the credential used by the product itself. The billing API key only authorises billing-related API calls and uses this format:

```text
inlets_live_<key-id>.<secret>
```

The key is shown once when issued. If it is lost or exposed, ask the inlets team to rotate it.

## Fetch quantity

Fetch the current quantity and policy for the subscription linked to the billing API key:

```http
GET /v1/billing/quantity
Authorization: Bearer inlets_live_<key-id>.<secret>
```

Response:

```json
{
  "billable_quantity": 15,
  "licensed_quantity": 15,
  "policy": {
    "included_quantity": 10,
    "quantity_step": 5
  }
}
```

## Update quantity

Report the number of units you want your subscription to cover. The product and subscription are resolved from the billing API key issued by the inlets team. For Inlets Uplink and standalone inlets Pro, the quantity is the number of tunnels.

```http
PUT /v1/billing/quantity
Authorization: Bearer inlets_live_<key-id>.<secret>
Content-Type: application/json

{
  "quantity": 11
}
```

Response:

```json
{
  "requested_quantity": 11,
  "billable_quantity": 15,
  "licensed_quantity": 15,
  "policy": {
    "included_quantity": 10,
    "quantity_step": 5
  }
}
```

Inlets Uplink is sold as provider capacity. Each subscription includes a base tunnel allowance, and additional capacity is sold in bands.

For example, if your subscription includes 10 tunnels and the additional capacity step is 5:

| Inlets Uplink requested tunnels | Billable capacity |
| ---: | ---: |
| 1-10 | 10 |
| 11-15 | 15 |
| 16-20 | 20 |
| 21-25 | 25 |

Standalone inlets Pro subscriptions are billed with per-tunnel granularity after the minimum quantity.

For example, if your subscription has a minimum billable quantity of 5:

| inlets Pro requested tunnels | Billable capacity |
| ---: | ---: |
| 1-5 | 5 |
| 6 | 6 |
| 7 | 7 |
| 8 | 8 |
| 9 | 9 |
| 10 | 10 |
| 11 | 11 |

The API returns the billable quantity after applying the current subscription policy.

## Fetch license entitlement

Inlets Uplink can use the license entitlement endpoint to fetch the tunnel limit and subscription identity for a license key. This endpoint is authenticated by the inlets Pro or Inlets Uplink license key, not by the billing API key.

```http
GET /v1/license/inlets/entitlement
Authorization: Bearer <license-key>
```

Response:

```json
{
  "valid": true,
  "provider": "lemonsqueezy",
  "product": "inlets-uplink",
  "quantity": 15,
  "entitlements": {
    "tunnels": 15
  },
  "subscription": {
    "product_name": "Inlets Uplink",
    "variant_name": "Provider",
    "customer_name": "Example Ltd",
    "customer_email": "ops@example.com"
  }
}
```

## Errors

The API uses standard HTTP status codes:

| Status | Meaning |
| --- | --- |
| `400` | The request body is invalid or the quantity is missing. |
| `401` | The bearer token is missing or invalid. |
| `403` | The token is not authorised for quantity updates. |
| `409` | The subscription could not be resolved unambiguously. |
| `500` | The request could not be completed. |

Example error:

```json
{
  "error": "quantity must be greater than zero"
}
```
