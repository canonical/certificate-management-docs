(securing-internal-communication)=

# Securing internal communication

This page explains how TLS secures communication between units of the same application or between applications within a single model.

## Architecture

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart TD
    SSC["Self-signed<br/>certificates"]

    subgraph AppA["Application A"]
        A1["Unit 0"]
        A2["Unit 1"]
        A3["Unit 2"]
    end

    subgraph AppB["Application B"]
        B1["Unit 0"]
        B2["Unit 1"]
    end

    SSC -.->|"tls-certificates<br/>(UNIT mode)"| AppA
    SSC -.->|"tls-certificates<br/>(UNIT mode)"| AppB
    A1 <-->|"HTTPS"| A2
    A1 <-->|"HTTPS"| A3
    A2 <-->|"HTTPS"| A3
    AppA <-->|"HTTPS"| AppB

    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333
    classDef unit fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef app fill:#f8f9fa,stroke:#1565C0,stroke-width:2px,color:#333

    class SSC provider
    class A1,A2,A3,B1,B2 unit
    class AppA,AppB app
```

## How it works

- **`self-signed-certificates`** is deployed in the same model as the applications.
- Each application integrates with it over the `tls-certificates` interface in **UNIT mode**, so every unit receives its own unique leaf certificate.
- Each unit also receives the **CA certificate** that signed it. Units trust this CA to validate each other's certificates.
- The certificate chain is: `[Leaf Certificate, CA Certificate]`.

## When to use

- Internal traffic between units of a replicated application (e.g., database peer replication).
- Inter-application traffic within the same model where both sides can trust the same CA.

## Related topics

- {ref}`Securing API communication <securing-api-communication>`: for client-facing traffic that requires a publicly trusted CA or an ingress.
- {ref}`CA trust best practices <ca-trust-best-practices>`: how trust is established and why internal certs require a private CA.
- {ref}`Multi-model TLS reference architectures <multi-model-tls>`: when the deployment spans multiple Juju models.
