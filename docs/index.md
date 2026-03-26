---
myst:
  html_meta:
    description: "Canonical Certificate Management provides charms for managing TLS certificates in Juju-deployed applications with automated certificate lifecycle management."
---

(home)=

# Certificate Management documentation

Canonical Certificate Management is a collection of charms for TLS certificate management in Juju deployments. Choose the right charm(s) for your needs:

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
    H["Notary (coming soon)"]
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

    class D,E,G,I charmLink
    class A,B,C,F,H plainNode

    click D "https://charmhub.io/lego" "View Lego charm documentation"
    click E "https://charmhub.io/manual-tls-certificates" "View Manual TLS charm documentation"
    click G "https://charmhub.io/self-signed-certificates" "View Self-signed certificates charm documentation"
    click I "https://charmhub.io/vault" "View Vault charm documentation"
```

(project-and-community)=

## Project and community

Canonical Certificate Management is an open source project that warmly welcomes community projects, contributions, suggestions, fixes and constructive feedback.

* [Code of Conduct ](https://ubuntu.com/community/code-of-conduct)
* [Join our chat](https://matrix.to/#/#tls:ubuntu.com)
* [Join our forum ](https://discourse.charmhub.io/)
* [Report a bug](https://github.com/canonical/certificate-management-docs/issues)
* [Contribute](https://github.com/canonical/certificate-management-docs/issues)
* [Visit our careers page](https://canonical.com/careers)

Thinking about using Juju for your next project? [Get in touch](https://canonical.com/contact-us)!
