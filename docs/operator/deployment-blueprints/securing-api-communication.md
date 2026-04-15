(securing-api-communication)=

# Securing API communication

To secure communication between clients and your applications, you should deploy a TLS provider to issue certificates for API-facing traffic.

Supported TLS providers include:

- [self-signed-certificates](https://charmhub.io/self-signed-certificates)
- [Vault](https://charmhub.io/vault-k8s)
- [Lego](https://charmhub.io/lego)
- [manual-tls-certificates](https://charmhub.io/manual-tls-certificates)
- [Notary](https://canonical-notary.readthedocs-hosted.com/en/latest/)

```{note}
Deploying the TLS provider in a separate model is recommended for production but not mandatory. In smaller or simpler setups, the provider and requirer applications can be in the same model.
```

## 1. Using an ingress

In most cases, applications are accessed behind an ingress.

The ingress will:

1. Integrate with the TLS provider in **APP mode** to secure external (client-to-ingress) communication.
2. Terminate TLS traffic before passing requests to backend applications.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart LR
    C["👤 Client"] -->|"HTTPS"| T["Traefik\n(Ingress)"]
    T -->|"HTTP"| A["Application"]
    P["TLS Provider"] -.->|"certificates\nintegration\n(APP mode)"| T

    classDef client fill:#f5f5f5,stroke:#333,stroke-width:2px,color:#333
    classDef ingress fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333
    classDef app fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333

    class C client
    class T ingress
    class A app
    class P provider
```

For internal traffic between the ingress and backend applications, you can add end-to-end encryption:

- Use a provider like `self-signed-certificates` for backend certificates.
- Integrate each backend application with the provider over the `tls-certificates` interface in **UNIT mode**.
- Integrate the ingress with the provider over the `certificates-transfer` interface so that the ingress trusts the CA used by the backend units.
- This ensures the ingress can validate backend-provided certificate chains that lead to the trusted CA.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart LR
    C["👤 Client"] -->|"HTTPS"| T["Traefik\n(Ingress)"]
    T -->|"HTTPS"| A["Application"]
    APIP["TLS Provider\n(API certs)"] -.->|"certificates\nintegration\n(APP mode)"| T
    INTP["Self-signed\ncertificates"] -.->|"certificates\nintegration\n(UNIT mode)"| A
    INTP -.->|"certificates-transfer\nintegration"| T

    classDef client fill:#f5f5f5,stroke:#333,stroke-width:2px,color:#333
    classDef ingress fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333
    classDef app fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333

    class C client
    class T ingress
    class A app
    class APIP,INTP provider
```

## 2. Trust establishment between clients and applications

For TLS to work correctly:

- Clients must **trust the CA** that issued the application's leaf certificate.
- Applications must present a **valid chain** that leads to that trusted CA.

### Public CAs

When using a public CA through `lego` (e.g., Let's Encrypt), certificates are usually trusted by default by most clients and browsers.

### Private or self-signed CAs

When using `self-signed-certificates`, `vault`, `manual-tls-certificates`, or `notary`, clients (or client applications) must explicitly trust the CA.

For Juju-integrated client applications, this is achieved by integrating with the provider over the `certificates-transfer` interface.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart TD
    P["TLS Provider"] -.->|"certificates\nintegration"| Server["Server\nApplication"]
    P -.->|"certificates-transfer\nintegration"| ClientApp["Client\nApplication"]
    ClientApp -->|"HTTPS\n(trusts CA)"| Server

    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333
    classDef app fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef client fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333

    class P provider
    class Server app
    class ClientApp client
```

See {ref}`ca-trust-best-practices` for detailed guidance on CA trust patterns.
