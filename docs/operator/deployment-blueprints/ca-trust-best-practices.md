(ca-trust-best-practices)=

# CA trust best practices

In the Juju ecosystem, multiple TLS providers can be used to issue certificates for applications and units:

- [self-signed-certificates](https://charmhub.io/self-signed-certificates)
- [Vault](https://charmhub.io/vault-k8s)
- [Lego](https://charmhub.io/lego)
- [manual-tls-certificates](https://charmhub.io/manual-tls-certificates)
- [Notary](https://canonical-notary.readthedocs-hosted.com/en/latest/)

In many cases, the CA certificates used by these providers are self-signed or private. This means that the certificates they issue will **not** be trusted by default by other applications or clients unless the CA is explicitly trusted.

## Trust establishment between clients and applications

For TLS to work correctly:

- Clients must **trust the CA** that issued the application's leaf certificate.
- Applications must present a **valid chain** that leads to that trusted CA.

### Public CAs

When using a public CA through `lego` (e.g., Let's Encrypt), certificates are usually trusted by default by most clients and browsers.

### Private or self-signed CAs

When using `self-signed-certificates`, `vault`, `manual-tls-certificates`, or `notary`, clients (or client applications) must explicitly trust the CA.

For Juju-integrated client applications, this is achieved by integrating with the provider over the `certificates-transfer` interface.

## Best practices for production deployments

### Internal communication (unit-to-unit)

As described in the {ref}`securing internal communication <securing-internal-communication>` guide, `self-signed-certificates` can be used to secure intra-application traffic. In this case:

- Each unit of an application receives the CA certificate as part of the relation data from the `tls-certificates` integration.
- The CA is the **direct issuer** of the application's leaf certificate.
- **Trusting this CA is sufficient** to establish trust in the leaf certificates.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart TD
    SSC["Self-signed<br/>certificates<br/>(CA)"]

    subgraph Application
        U1["Unit 0"]
        U2["Unit 1"]
    end

    SSC -.->|"certificates integration<br/>(issues leaf cert + CA cert)"| Application
    U1 <-->|"HTTPS<br/>(trusts CA)"| U2

    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333
    classDef unit fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef appGroup fill:#F3E5F5,stroke:#6A1B9A,stroke-width:2px,color:#333

    class SSC provider
    class U1,U2 unit
    class Application appGroup
```

### API communication

For more complex deployments that {ref}`secure API communication <securing-api-communication>`, client applications should trust the CA **directly from the CA provider**, not from the application serving the certificate.

- The CA certificate can be obtained by integrating with the provider over the `certificates-transfer` interface.
- When the provider uses an intermediate CA, it is recommended to trust the **root CA** (or the highest CA in the hierarchy). However, this is not mandatory.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart TD
    P["TLS Provider<br/>(CA)"]
    P -.->|"certificates<br/>integration"| Server["Server App"]
    P -.->|"certificates-transfer<br/>integration<br/>(provides CA cert)"| ClientApp["Client App"]
    ClientApp -->|"HTTPS<br/>(validates chain → CA)"| Server

    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333
    classDef server fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef client fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333

    class P provider
    class Server server
    class ClientApp client
```

### Summary

| Scenario                | Trust source                                  | Interface               |
| ----------------------- | --------------------------------------------- | ----------------------- |
| Internal (unit-to-unit) | CA cert from `tls-certificates` relation data | `tls-certificates`      |
| API (client-to-server)  | CA cert directly from the provider            | `certificates-transfer` |
| Intermediate CA         | Root CA (recommended)                         | `certificates-transfer` |
