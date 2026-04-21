(understanding-tls)=

# Understanding TLS in Juju deployments

X.509 certificates are essential to the security of many Internet protocols, including TLS/SSL. If your charm workload communicates over HTTPS, you most likely need these certificates. Within the Juju ecosystem, the `tls-certificates` charm relation interface handles X.509 certificate creation, renewal, and revocation.

This guide walks you through two key decisions:

1. **Where** should TLS be terminated in your deployment?
2. **Which** TLS provider charm should you use?

## Where to terminate TLS

The first question to answer is where TLS connections should be terminated. This determines whether your application charm needs to manage certificates directly.

### At the ingress level

In deployments where X.509 certificates are needed to ensure HTTPS communication with users or systems **outside** of the Juju model, it is recommended to use an ingress controller like [Traefik](https://charmhub.io/traefik-k8s). Traefik handles TLS termination, sending decrypted traffic to your application over HTTP. Your application charm does not need to manage X.509 certificates — implementing the ingress integration is all that's required as Traefik already supports the `tls-certificates` interface.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart LR
    C["👤 Client"] -->|"HTTPS"| T["Traefik\n(Ingress)"]
    T -->|"HTTP"| A["Application"]
    P["TLS Provider"] -.->|"certificates\nintegration"| T

    classDef client fill:#f5f5f5,stroke:#333,stroke-width:2px,color:#333
    classDef ingress fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333
    classDef app fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333

    class C client
    class T ingress
    class A app
    class P provider
```

In this setup:

- The **client** communicates with the ingress over **HTTPS**.
- The **ingress** (Traefik) terminates TLS and forwards **HTTP** traffic to the application.
- The **TLS provider** supplies certificates to the ingress through the `tls-certificates` integration.

### At the application level

For use cases where HTTPS communication is needed **inside** the Juju model — such as between an ingress and the application, or between different units of the same application — TLS connections must be terminated at the application level. This requires implementing the requirer side of the `tls-certificates` interface in the application charm.

A typical example is enabling various units of a PostgreSQL cluster to communicate via TLS, or ensuring end-to-end encryption between an ingress and the backend application.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart LR
    C["👤 Client"] -->|"HTTPS"| T["Traefik\n(Ingress)"]
    T -->|"HTTPS"| A["Application"]
    P["TLS Provider"] -.->|"certificates\nintegration"| T
    P -.->|"certificates\nintegration"| A

    classDef client fill:#f5f5f5,stroke:#333,stroke-width:2px,color:#333
    classDef ingress fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333
    classDef app fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333

    class C client
    class T ingress
    class A app
    class P provider
```

In this setup:

- The **client** communicates with the ingress over **HTTPS**.
- The **ingress** (Traefik) forwards **HTTPS** traffic to the application (end-to-end encryption).
- The **TLS provider** supplies certificates to **both** the ingress and the application through the `tls-certificates` integration.

## Choosing a TLS provider

Once you've determined where TLS will be terminated, the next step is choosing the right TLS provider charm for your deployment. Use the following decision tree:

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '11px'}, 'flowchart': {'nodeSpacing': 25, 'rankSpacing': 25, 'curve': 'linear', 'padding': 10}}}%%
flowchart TD
    A{Do you have existing<br/>PKI infrastructure?}
    B{Does it expose an<br/>ACME interface?}
    C{Do you need a<br/>full-featured solution?}

    D["🔗 Lego"]
    E["🔗 Manual TLS certificates"]
    F{Do you need<br/>HSM support?}
    G["🔗 Self-signed certificates"]
    H["🔗 Notary"]
    I["🔗 Vault"]

    A -->|Yes| B
    A -->|No| C

    B -->|Yes| D
    B -->|No| E

    C -->|Yes| F
    C -->|No| G

    F -->|Yes| H
    F -->|No| I

    classDef charmLink fill:#FFF5F0,stroke:#E95420,stroke-width:2px,color:#333
    classDef plainNode fill:#fff,stroke:#333,stroke-width:2px,color:#333

    class D,E,G,H,I charmLink
    class A,B,C,F plainNode

    click D "https://charmhub.io/lego" "View Lego charm documentation"
    click E "https://charmhub.io/manual-tls-certificates" "View Manual TLS charm documentation"
    click G "https://charmhub.io/self-signed-certificates" "View Self-signed certificates charm documentation"
    click H "https://canonical-notary.readthedocs-hosted.com/en/latest/" "View Notary documentation"
    click I "https://charmhub.io/vault" "View Vault charm documentation"
```

### Self-signed certificates

The [self-signed-certificates](https://charmhub.io/self-signed-certificates) operator is ideal for:

- **Internal communication**: Securing traffic between units of the same application (e.g., PostgreSQL cluster replication) or between applications within a Juju model.
- **Development and non-production environments**: Quick setup without external dependencies.

Upon deployment, the self-signed-certificates operator generates a private key and a CA certificate (not signed by any external authority). It signs each certificate request it receives using this self-signed CA certificate. Because the CA is self-signed, clients and applications must explicitly trust it — see {ref}`ca-trust-best-practices` for details.

This charm implements the `tls-certificates` interface using the [tls_certificates library](https://documentation.ubuntu.com/charmlibs/reference/charmlibs/interfaces/tls-certificates), which provides automatic certificate renewal before expiry. Additionally supports CA private key rotation via the `rotate-private-key` action, which automatically revokes and regenerates all certificates for integrated applications.

### Lego (Let's Encrypt or another ACME server)

The [Lego](https://charmhub.io/lego) charm operator requests certificates using the ACME protocol from providers like Let's Encrypt. The ACME server validates domain ownership using the [DNS-01](https://letsencrypt.org/docs/challenge-types/) challenge.

This is a good option when you need publicly trusted certificates and your DNS provider is supported. If your DNS provider is not currently supported, reach out to the team.

This charm implements the `tls-certificates` interface using the [tls_certificates library](https://documentation.ubuntu.com/charmlibs/reference/charmlibs/interfaces/tls-certificates), which provides automatic certificate renewal before expiry and structured error reporting with standardized codes when certificate requests fail.

### Manual TLS certificates

The [manual-tls-certificates](https://charmhub.io/manual-tls-certificates) operator is designed for organisations that have an existing manual process for managing certificates. It supports actions to list certificate requests, retrieve CSRs, and supply manually obtained certificates.

For a more streamlined experience, consider [Notary](https://canonical-notary.readthedocs-hosted.com/en/latest/), which provides:

- A **web UI** for managing certificate requests and lifecycle.
- **User management** for controlling access to certificate operations.
- **HSM support** for hardware-backed key storage.

### Vault

[Vault](https://charmhub.io/vault-k8s) is a popular secret management platform. Both the Kubernetes and machine versions of the Vault charm can be used as an intermediate Certificate Authority to provide certificates in the Juju ecosystem.

Vault is well-suited for production deployments that require a full-featured PKI solution with secret management capabilities. See the [Vault charm documentation](https://canonical-vault-charms.readthedocs-hosted.com/en/latest/) for details.

This charm implements the `tls-certificates` interface using the [tls_certificates library](https://documentation.ubuntu.com/charmlibs/reference/charmlibs/interfaces/tls-certificates), which provides automatic certificate renewal before expiry and structured error reporting with standardized codes when certificate requests fail.
