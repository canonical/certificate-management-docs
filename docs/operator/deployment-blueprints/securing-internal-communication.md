(securing-internal-communication)=

# Securing internal communication

When deploying multiple applications in a Juju model that require TLS for internal communication (e.g., between units of the same application), we recommend using the [self-signed-certificates](https://charmhub.io/self-signed-certificates) charm.

## Architecture

Integrate each application with `self-signed-certificates` over the `tls-certificates` interface in **UNIT mode**. This ensures that each unit receives its own unique certificate.

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '12px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 40, 'curve': 'linear', 'padding': 10}}}%%
flowchart TD
    SSC["Self-signed\ncertificates"]

    subgraph Application
        U1["Unit 0"]
        U2["Unit 1"]
        U3["Unit 2"]
    end

    SSC -.->|"certificates\nintegration\n(UNIT mode)"| Application
    U1 <-->|"HTTPS"| U2
    U1 <-->|"HTTPS"| U3
    U2 <-->|"HTTPS"| U3

    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333
    classDef unit fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef app fill:#f8f9fa,stroke:#1565C0,stroke-width:2px,color:#333

    class SSC provider
    class U1,U2,U3 unit
    class Application app
```

The `self-signed-certificates` charm issues a Certificate object for each unit, containing:

- A **signed leaf certificate** for that specific unit.
- The **CA certificate** that signed the leaf certificate.
- A **certificate chain** in the form: `[Signed Leaf Certificate, CA Certificate]`.

## Trust establishment

Because the CA is self-signed, it is not trusted by default by most systems.

To establish trust:

- Explicitly configure the application to trust the CA certificate provided by `self-signed-certificates`.
- Since the CA directly issues the unit certificates, trusting this CA is sufficient to validate the leaf certificates.

Each application can then configure TLS according to its own requirements, using the provided certificate, CA, and chain.

See {ref}`ca-trust-best-practices` for more details on trust establishment patterns.

## Example: Production deployment with multiple applications

In a realistic production deployment, multiple applications may need both internal TLS (unit-to-unit) and external TLS (API communication). Here is an example of what such a deployment can look like:

```{mermaid}
%%{init: {'theme': 'default', 'themeVariables': {'fontSize': '11px'}, 'flowchart': {'nodeSpacing': 25, 'rankSpacing': 35, 'curve': 'linear', 'padding': 8}}}%%
flowchart TD
    Client["👤 Client"]
    Traefik["Traefik\n(Ingress)"]
    APIProvider["TLS Provider\n(API certs)"]
    InternalProvider["Self-signed\ncertificates\n(Internal certs)"]

    subgraph AppModel["Application Model"]
        subgraph AppA["Application A"]
            A1["Unit 0"]
            A2["Unit 1"]
        end
        subgraph AppB["Application B"]
            B1["Unit 0"]
            B2["Unit 1"]
        end
    end

    Client -->|"HTTPS"| Traefik
    Traefik -->|"HTTPS"| AppA
    AppA <-->|"HTTPS"| AppB
    A1 <-->|"HTTPS"| A2
    B1 <-->|"HTTPS"| B2

    APIProvider -.->|"certificates\nintegration\n(APP mode)"| Traefik
    InternalProvider -.->|"certificates\nintegration\n(UNIT mode)"| AppA
    InternalProvider -.->|"certificates\nintegration\n(UNIT mode)"| AppB

    classDef client fill:#f5f5f5,stroke:#333,stroke-width:2px,color:#333
    classDef ingress fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#333
    classDef provider fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#333
    classDef unit fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#333
    classDef appGroup fill:#f8f9fa,stroke:#1565C0,stroke-width:1px,color:#333
    classDef model fill:#fafafa,stroke:#666,stroke-width:1px,color:#333

    class Client client
    class Traefik ingress
    class APIProvider,InternalProvider provider
    class A1,A2,B1,B2 unit
    class AppA,AppB appGroup
    class AppModel model
```

In this deployment:

- An **API TLS provider** (e.g., Vault, Lego) provides certificates in APP mode for the ingress to secure client-facing traffic.
- A **self-signed-certificates** provider issues certificates in UNIT mode for each application unit to secure internal communication.
- **Application A** and **Application B** communicate over HTTPS, with each unit having its own certificate.
