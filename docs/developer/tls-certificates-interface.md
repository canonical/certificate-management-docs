(tls-certificates-interface)=

# The `tls-certificates` library

The `tls-certificates` library helps charm authors automate certificate requests, renewal, and revocation in Juju. It is used by charms that provide or require X.509 certificates, and supports both requirer and provider roles.

If your application charm needs to terminate TLS at the application level (see [Understanding TLS](../operator/understanding-tls.md)), you must implement the **requirer** side of the `tls-certificates` interface in your charm and integrate with a TLS provider charm.

## How the interface works

The whole idea behind the TLS Certificates interface is that charms can request TLS certificates from TLS providers without ever sharing their private key.

The TLS Certificates Requirer (through the use of the TLS Certificates Library) generates its private key and a Certificate Signing Request (CSR). This CSR is inserted into its unit (or application) relation data.

The TLS Certificates Provider reads this CSR, signs a certificate for it and inserts this certificate into its application relation data.

The TLS Certificates Requirer then reads the certificate, and typically stores it in a file on the workload.

```{mermaid}
flowchart LR
    subgraph App["Your App"]
        subgraph Unit0["unit-0"]
            PKEY["pkey"]
        end
    end
    Provider["TLS Certificates Provider"]

    App -- "tls-certificates" --> Provider
    App -.->|"CSR"| Provider
    Provider -.->|"CA Cert,<br/>CA Chain,<br/>Cert"| App
```

## Key concepts

### Requirer vs. Provider

- **Provider**: A charm that issues certificates (e.g., `self-signed-certificates`, `vault`, `lego`, `manual-tls-certificates`, `notary`).
- **Requirer**: A charm that requests and uses certificates for its workload.

### Certificate modes

The interface supports three modes for certificate issuance:

- **APP mode**: A single certificate is issued for the application as a whole. This is typically used for ingress controllers or services with a single endpoint.
- **UNIT mode**: Each unit of the application receives its own unique certificate. This is used for securing communication between individual units (e.g., database cluster replication).
- **APP_AND_UNIT mode**: Combines both APP and UNIT modes, one certificate for the application and individual certificates for each unit. This is useful when you need both application-level and unit-level certificates simultaneously.

## Getting started

The `tls-certificates` library handles most of the heavy lifting for both requirers and providers.

### Installation

Add `charmlibs-interfaces-tls-certificates` to your Python dependencies (e.g. in requirements.txt or pyproject.toml). Then in your Python code, import as:

```python
from charmlibs.interfaces.tls_certificates import (
  Certificate,
  CertificateRequestAttributes,
  Mode,
  PrivateKey,
  TLSCertificatesRequiresV4,
)
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

For more details, see the [library source code](https://github.com/canonical/charmlibs/tree/main/interfaces/tls-certificates).
