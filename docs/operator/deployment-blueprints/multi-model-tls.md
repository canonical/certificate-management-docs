(multi-model-tls)=

# Multi-model TLS reference architecture

This reference architecture shows how to secure a multi-model Juju deployment with TLS. It combines a central PKI model for public-facing certificates with per-model `self-signed-certificates` for internal communication.

## Architecture

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '10px'}, 'flowchart': {'nodeSpacing': 20, 'rankSpacing': 45, 'curve': 'linear', 'padding': 10}}}%%
flowchart TD
    Client["👤 Client"]

    subgraph PKIModel["PKI Model"]
        TLS["TLS Provider"]
    end

    subgraph ModelA["Model A"]
        SSC_A["self-signed-certificates"]
        Traefik["Traefik<br/>(Ingress)"]
        AppA["Application A"]
    end

    subgraph ModelB["Model B"]
        SSC_B["self-signed-certificates"]
        AppB["Application B"]
    end

    %% Central TLS provider issues public-facing certs
    TLS -.->|"tls-certificates (APP mode)"| Traefik
    TLS -.->|"tls-certificates (APP mode)"| AppB

    %% Central CA trust distribution
    TLS -.->|"certificate-transfer"| AppA
    TLS -.->|"certificate-transfer"| Traefik

    %% Per-model SSC for internal communication
    SSC_A -.->|"tls-certificates (UNIT mode)"| AppA
    SSC_B -.->|"tls-certificates (UNIT mode)"| AppB

    %% Ingress trusts internal CA
    SSC_A -.->|"certificate-transfer"| Traefik

    %% Traffic flows
    Client -->|"HTTPS"| Traefik
    Client -->|"HTTPS"| AppB
    Traefik -->|"HTTPS"| AppA

    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333
    classDef app fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef ingress fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333
    classDef client fill:#f5f5f5,stroke:#333,stroke-width:2px,color:#333
    classDef model fill:#fafafa,stroke:#666,stroke-width:1px,color:#333

    class TLS provider
    class SSC_A,SSC_B provider
    class AppA,AppB app
    class Traefik ingress
    class Client client
    class PKIModel,ModelA,ModelB model
```

## How it works

**Internal communication**: Each model deploys `self-signed-certificates` which issues per-unit certificates in UNIT mode. Units use these to encrypt peer traffic (replication, cluster membership).

**Public-facing communication**: A central PKI model hosts a TLS provider (Vault, Lego, or any other) that issues certificates in APP mode:

- **Application A** is behind an ingress. The TLS provider issues a certificate to Traefik. The ingress trusts the internal CA via `certificate-transfer` so it can validate Application A's backend certificate.
- **Application B** is accessed directly by clients. The TLS provider issues a certificate directly to Application B in APP mode. The application serves this certificate on its public endpoint.

**CA trust**: Applications that need to validate the central CA (e.g., to call another service that presents a certificate issued by it) receive it via `certificate-transfer` from the PKI model.

```{note}
The per-model `self-signed-certificates` for internal communication could also be centralised into the PKI model. This simplifies CA management at the cost of making all models dependent on the PKI model for internal cert renewal.
```
