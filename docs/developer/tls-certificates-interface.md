(tls-certificates-interface)=

# Implementing the `tls-certificates` integration

```{note}
The `tls-certificates` library has moved from Charmhub to [charmlibs](https://github.com/canonical/charmlibs/tree/main/interfaces/tls-certificates). All new development and documentation updates happen there.
```

The `tls-certificates` charm integration interface is used by charms that provide or require X.509 certificates. It automates certificate creation, renewal, and revocation within the Juju ecosystem.

If your application charm needs to terminate TLS at the application level (see [](understanding-tls)), you must implement the **requirer** side of the `tls-certificates` interface in your charm. You then integrate your charm with a TLS provider operator of your choice.

## Key concepts

### Requirer vs. Provider

- **Provider**: A charm that issues certificates (e.g., `self-signed-certificates`, `vault`, `lego`, `manual-tls-certificates`, `notary`).
- **Requirer**: A charm that requests and uses certificates for its workload.

### Certificate modes

The interface supports two modes for certificate issuance:

- **APP mode**: A single certificate is issued for the application as a whole. This is typically used for ingress controllers or services with a single endpoint.
- **UNIT mode**: Each unit of the application receives its own unique certificate. This is used for securing communication between individual units (e.g., database cluster replication).

## Getting started

The `tls-certificates` library handles most of the heavy lifting for both requirers and providers.

### Installation

The library is available from the [charmlibs repository](https://github.com/canonical/charmlibs/tree/main/interfaces/tls-certificates):

```bash
charmcraft fetch-lib charms.tls_certificates_interface.v4.tls_certificates
```

### Implementing a requirer

A minimal requirer implementation involves:

1. Adding the `tls-certificates` integration to your charm's `charmcraft.yaml`:

   ```yaml
   requires:
     certificates:
       interface: tls-certificates
   ```

2. Using the library in your charm code to generate a CSR (Certificate Signing Request) and handle the certificate lifecycle events.

### Implementing a provider

Provider charms are responsible for:

1. Receiving CSRs from requirers.
2. Signing them and returning the certificate, CA certificate, and chain.
3. Handling certificate renewal and revocation.

## Further resources

- [Library source code](https://github.com/canonical/charmlibs/tree/main/interfaces/tls-certificates)
- [TLS Certificates Interface on Charmhub](https://charmhub.io/tls-certificates-interface) (legacy documentation)
