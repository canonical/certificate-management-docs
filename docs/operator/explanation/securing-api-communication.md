(securing-api-communication)=

# Securing API communication

This page explains how TLS secures client-facing traffic, where external clients or other services connect to your applications over HTTPS directly or through an ingress.

## With an ingress

When clients reach the application through an ingress (e.g., Traefik), TLS is terminated at the ingress with a publicly trusted certificate. The ingress then connects to the backend over a separate internal TLS channel.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart LR
    C["👤 Client"] -->|"HTTPS"| T["Traefik<br/>(Ingress)"]
    T -->|"HTTPS"| A["Application"]
    APIP["TLS Provider<br/>(API certs)"] -.->|"tls-certificates<br/>(APP mode)"| T
    INTP["Self-signed<br/>certificates<br/>(internal certs)"] -.->|"tls-certificates<br/>(UNIT mode)"| A
    INTP -.->|"certificate-transfer"| T

    classDef client fill:#f5f5f5,stroke:#333,stroke-width:2px,color:#333
    classDef ingress fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333
    classDef app fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333

    class C client
    class T ingress
    class A app
    class APIP,INTP provider
```

- **API TLS provider** (Lego, Vault, or another provider) issues a certificate for the ingress in **APP mode**. This is the publicly trusted certificate presented to clients.
- **Traefik** terminates client TLS and forwards requests to the backend application.
- **`self-signed-certificates`** issues internal certificates to backend applications in **UNIT mode** for end-to-end encryption between the ingress and the application.
- The ingress integrates with `self-signed-certificates` over the `certificate-transfer` interface to trust the internal CA, so it can validate the backend application's certificate.

## Without an ingress

When clients connect directly to the application, the application itself serves the API certificate. The application integrates with two TLS providers on two separate relations—one for client-facing certificates and one for internal communication.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart LR
    C["👤 Client"] -->|"HTTPS"| A["Application"]
    APIP["TLS Provider<br/>(API certs)"] -.->|"tls-certificates<br/>(APP mode)"| A
    INTP["Self-signed<br/>certificates<br/>(internal certs)"] -.->|"tls-certificates<br/>(UNIT mode)"| A

    classDef client fill:#f5f5f5,stroke:#333,stroke-width:2px,color:#333
    classDef app fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333

    class C client
    class A app
    class APIP,INTP provider
```

- **API TLS provider** issues a certificate to the application in **APP mode**. The application presents this certificate to clients on its external-facing endpoint.
- **`self-signed-certificates`** issues a separate certificate in **UNIT mode** for internal peer communication (e.g., replication between units).
- The application binds each certificate to the appropriate network interface or port—API cert on the external address, internal cert on the cluster/peer address.

## When to use

- Any deployment where external clients require HTTPS access to the application—with or without an ingress.
- When the API-facing certificate must be publicly trusted (e.g., issued by Let's Encrypt via Lego).
- When end-to-end encryption is needed between the ingress and backend applications—not just TLS termination at the edge.
- When the application is directly exposed and needs to serve a trusted certificate while still using a private CA for internal peer traffic.

## Related topics

- {ref}`Securing internal communication <securing-internal-communication>`: for unit-to-unit TLS without an ingress.
- {ref}`Understanding TLS <understanding-tls>`: why API and internal certs require separate CAs.
- {ref}`Multi-model TLS reference architectures <multi-model-tls>`: when the TLS provider or the ingress is in a different model.
