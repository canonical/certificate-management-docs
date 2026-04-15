# Getting Started with the TLS Certificates Library (v4)

In this tutorial, we will take a working Nginx charm and add the TLS Certificates integration using the TLS certificates library v4.

## Pre-requisites

### Knowledge

- You have some experience writing charms and are comfortable with the charm development tooling

### Software

- Ubuntu 22.04
- A Juju controller with at least version 3.0 running on MicroK8s
- Charmcraft

## 1. Pack and deploy the nginx demo charm

Clone the TLS Certificates Interface Demo project:

```bash
git clone git@github.com:canonical/tls-certificates-interface-demo.git
cd tls-certificates-interface-demo
```

Pack the charm:

```bash
charmcraft pack
```

Create a Juju model:

```bash
juju add-model demo
```

Deploy the charm:

```bash
juju deploy ./tls-certificates-interface-demo_amd64.charm nginx-http --resource nginx-image=nginx:1.27.1
```

Wait for the charm to go to the Active/Idle status:

```bash
juju status
```

Using your browser, navigate to the application address on port 8080 using the HTTP scheme (e.g. `http://10.152.183.199:8080`).

You should see the Nginx welcome page. You now have a working Nginx charm.

## 2. Import and use the TLS Certificates Library

This section outlines the changes to make to the Nginx charm to support the TLS certificates integration. You can use [this pull request](https://github.com/canonical/tls-certificates-interface-demo/pull/5) as reference.

Add `charmlibs-interfaces-tls-certificates` to your Python dependencies. Then in your Python code, import as:

```python
from charmlibs.interfaces import tls_certificates
```

Import the following classes from the TLS Certificates Interface Library:

```python
from charms.tls_certificates_interface.v4.tls_certificates import (
    Certificate,
    CertificateRequestAttributes,
    Mode,
    PrivateKey,
    TLSCertificatesRequiresV4,
)
```

Add the following global variables:

```python
CERTS_DIR_PATH = "/etc/nginx"
PRIVATE_KEY_NAME = "nginx.key"
CERTIFICATE_NAME = "nginx.pem"
```

In the charm’s class constructor, instantiate a `TLSCertificatesRequiresV4` object and have the `certificate_available` event handled by your central `_configure` event handler:

```python
class TlsCertificatesInterfaceDemoCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)
        ...
        self.certificates = TLSCertificatesRequiresV4(
            charm=self,
            relationship_name="certificates",
            certificate_requests=[self._get_certificate_request_attributes()],
            mode=Mode.UNIT,
        )
        ...
        framework.observe(
            self.certificates.on.certificate_available, self._configure
        )
        ...
```

Add a `_get_certificate_request_attributes` method:

```python
    def _get_certificate_request_attributes(self) -> CertificateRequestAttributes:
        return CertificateRequestAttributes(common_name="example.com")
```

In the collect unit status event handler, have the charm go to `Blocked` status until it is integrated with a TLS Certificates Provider:

```python
    def _on_collect_status(self, event: ops.CollectStatusEvent):
        ...
        if not self._relation_created("certificates"):
            event.add_status(
                ops.BlockedStatus("certificates integration not created")
            )
            return
        ...
```

Update the `_configure` event handler to manage TLS Certificates and restart the Pebble container when certificates have changed:

```python
    def _configure(self, _: ops.EventBase):
        if not self.container.can_connect():
            return
        if not self._relation_created("certificates"):
            return
        if not self._certificate_is_available():
            return
        certificate_update_required = self._check_and_update_certificate()
        desired_config_file = self._generate_config_file()
        if config_update_required := self._is_config_update_required(desired_config_file):
            self._push_config_file(content=desired_config_file)
        should_restart = config_update_required or certificate_update_required
        self._configure_pebble(restart=should_restart)
```

You will need methods to handle pulling, pushing, and comparing certificates (see the original tutorial for full code examples).

## 3. Handle attribute changes

The `CertificateRequestAttributes` can change. To have the library pick up the changes and generate new CSRs, you can register `refresh_events` when instantiating `TLSCertificatesRequiresV4`:

```python
self.certificates = TLSCertificatesRequiresV4(
    charm=self,
    relationship_name="certificates",
    certificate_requests=[self._get_certificate_request_attributes()],
    mode=Mode.UNIT,
    refresh_events=[self.on.config_changed]
)
```

Or you can use the public `sync` function of the library to trigger the same refresh process:

```python
self.certificates.sync()
```

## 4. Deploy the nginx charm and integrate it with a TLS provider

Deploy the new nginx charm:

```bash
juju deploy ./tls-certificates-interface-demo_amd64.charm nginx-https --resource nginx-image=nginx:1.27.1
```

Wait for the charm to go to the Blocked/idle status:

```bash
juju status
```

Deploy [Self Signed Certificates](https://charmhub.io/self-signed-certificates) (a TLS Certificates provider), and integrate it with the nginx charm:

```bash
juju deploy self-signed-certificates --channel=1/stable
juju integrate self-signed-certificates:certificates nginx-https:certificates
```

Wait for the `nginx-https` charm to go to the Active/Idle status:

```bash
juju status
```

Using your browser, navigate to the application address on port 8080 using the HTTPS scheme (e.g. `https://10.152.183.188:8080`).

You should see a warning about the certificate not being valid (expected for self-signed). Proceed and you should see the Nginx welcome page. You can inspect the certificate and notice that you received a certificate for `example.com`.

Congratulations, you added the TLS integration to your charm!
