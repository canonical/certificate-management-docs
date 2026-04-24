# Do I need to implement the TLS library?

Most charm applications need TLS certificate management for secure communication. The `tls-certificates` library handles certificate requests, renewals, and revocations automatically, making it essential for production-grade charms that require encrypted communication.

Implementing the TLS library in your charm gives operators the flexibility to choose their deployment model—whether they need application-level TLS termination, ingress-level termination, or both.

## When to implement the TLS library

Your charm needs to implement the `tls-certificates` library if your application requires any form of encrypted communication. Here are the most common scenarios:

### Securing internal communication among units

When your application runs as a cluster or distributed system, units need to communicate securely with each other. This is one of the most common use cases for TLS in Juju deployments.
For example database cluster nodes communicating internally (PostgreSQL, MySQL, MongoDB).

**Implementation**: Each unit requests its own certificate using `Mode.UNIT` to establish encrypted peer-to-peer communication.

### Securing API communication at the application level

When your application exposes APIs or services to other applications within the model, or to users calling the API, implementing TLS ensures end-to-end encryption and can enable mutual TLS (mTLS) authentication.
For example a user connecting to a database API.

## Next steps

To implement the TLS library in your charm:

1. Review the [Getting Started tutorial](../tutorials/getting-started-tls-certificates-library-v4.md)
2. Understand the [TLS certificates library reference](../tls-certificates-interface.md) to choose the right certificate mode
3. Choose your [TLS provider](../../operator/understanding-tls.md) based on your deployment needs
