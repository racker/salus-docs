## Problem Statements

The Envoys are currently configured with an Identity username, API key, and tenant ID. 

Since Identity only maintains a single API key per username, the Envoy's dependency on that API key is brittle. If a user decides to rotate the API key for any reason (periodic, compromised system, etc), then all Envoy configurations would be adjusted.

Storing the API key in each resource, even if the configuration file has limited access rights, may not be acceptable for some users. The API key is less crucial than a password, but is still a credential that has applicability beyond Envoy attachment and Salus access in general.

## Solution Overview

This proposal introduces an overall workflow similar to MaaS where Envoys (or Agents in the case of MaaS) are "pre authenticated" with a token, which is allocated by a user/system separate from the Envoy and configured for the Envoy. The gRPC attachment process of the Envoy remains mostly the same where the TLS client certificate is retrieved by the Envoy from the Auth Service, but the configured token is used instead of an Identity username and API key.

The concept of a pre-authenticated token addresses the problem statements by 
- ...shifting the lifecycle of the tokens into the realm of Salus, which improves usability with the possibility of longer lived tokens
- ...introducing authentication material stored on each resource which is completely decoupled from the Identity credentials. Furthermore, the scope of the token only applies to a single API endpoint, the gRPC client cert retrieval. Compromised tokens can be easily and precisely revoked and serve no benefit to a bad actor beyond the cert retrieval operation.

It's worth noting that a future enhancement of the Envoy could simplify the workflow by introducing a "setup" mechanism similar to the [MaaS `--setup` option](https://developer.rackspace.com/docs/rackspace-monitoring/v1/getting-started/install-configure/#run-agent-setup-program). This enhancement is not discussed further in this proposal since automated deployments/"kicks" of the Envoy may take on the burden of token allocation/retrieval.

## Work breakdown

- Introduce a dependency on MySQL for auth service
- Add API endpoints to auth service for allocating, retrieving, and revoking envoy-tokens (see [section below](#auth-service-api-endpoints) for more details). The API endpoints loosely follow [the MaaS Agent Token operations](https://developer.rackspace.com/docs/rackspace-monitoring/v1/api-reference/agent-token-operations/).
- Token value allocation is performed by the auth service using Java's `SecureRandom` and encoded as URL-safe values using `Base64`'s URL encoder
- Token values are persisted in MySQL with the entity fields:
  - Token : the generated token value; declared as primary key since token validation will only need to provide the token's value and retrieves the tenant ID as part of validation 
  - Tenant ID : the tenant that owns the token; indexed to facilitate query by tenant ID
  - Description : provided for the end user's informational purposes, for example "This is the primary token for any new device on the account"
  - Created timestamp : set on initial creation and provided for information purposes
  - Last access timestamp : updated each time auth service validates a token; provided to help end users audit usage and decide which tokens are no longer actively used; nullable and initialized to null
- Implement and add a security filter on auth service that validates envoy-token passed via `Authorization` header with authentication type `Bearer`. The security filter populates the authenticated security context with the principal set to the tenant ID and a role of "allocate cert".
- Add public API proxy endpoints for token allocation, retrieval, and revoke operations. Those operations are routed to auth service (see [section below](#public-api) for more details)
- Remove the Keystone/Identity logic from Envoy and simplify down to only the cert retrieval call using the configured envoy-token
- Simplify Repose configuration used in front of auth service to limit access to only the cert retrieval endpoint, but simply pass-through the request since auth service will perform the authentication via token validation

## System Changes

[This diagram](auth-service-components.puml) shows the resulting deployment structure and the three phases to enable authenticated Envoy attachment to Ambassadors:

1. Envoy-Tokens are created/retrieved by a human or a deployment-automation system. These API calls are authenticated by Repose (using Identity credentials), proxied by the public API service, and routed to the auth service.
2. The token retrieved in the previous phase is configured in the Envoys running on the tenant's resources.
3. The Envoy performs a two-step authentication
   1. Authenticate to the auth service using the token configured in previous phase and retrieve client certificate
   2. Authenticate gRPC attachment to an Ambassador using client certificate

## API Endpoints

### Auth Service API Endpoints

- `GET /cert` : retrieve client certificates for Envoy attachment
  - Requires `Authorization` header with type `Bearer` and Envoy Token
- `POST /api/tenant/{tenant}/envoy-tokens` : allocate an Envoy token
- `GET /api/tenant/{tenant}/envoy-tokens` : provides all Envoy tokens for the tenant
- `GET /api/tenant/{tenant}/envoy-tokens/{id}` : provides specific Envoy token for the tenant
- `PUT /api/tenant/{tenant}/envoy-tokens/{id}` : update the description field of a token
- `DELETE /api/tenant/{tenant}/envoy-tokens/{id}` : revoke an Envoy token

### Public API

The following are proxied to the respective endpoint listed above

- `POST /api/tenant/{tenant}/envoy-tokens`
- `GET /v1.0/tenant/{tenant}/envoy-tokens`
- `GET /v1.0/tenant/{tenant}/envoy-tokens/{id}`
- `PUT /api/tenant/{tenant}/envoy-tokens/{id}`
- `DELETE /v1.0/tenant/{tenant}/envoy-tokens/{id}`

### Token Response

- ID
- Token
- Description
- Created timestamp
- Last access timestamp

### Token Allocation/Update Request

- Description

## Thoughts on using Vault for token allocation and validation

Auth service currently doesn't access the MySQL database, so an initial thought was to manage agent-tokens via Vault. One of Vault's main features is to [manage a hierarchy of tokens](https://www.vaultproject.io/docs/concepts/tokens/); however, other than the root token, all tokens must have a TTL. This would not be feasible since an Envoy might only need to re-retrieve its client certificates once every few weeks or months if the overall system is stable. Vault provides the concept of [periodic tokens](https://www.vaultproject.io/docs/concepts/tokens/#periodic-tokens) for long running services that need ongoing use of a token; however, that would mean every Envoy would need periodically contact the auth service and it in turn contact Vault to refresh the token.

