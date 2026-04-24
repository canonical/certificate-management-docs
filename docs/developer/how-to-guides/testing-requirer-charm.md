# Testing your requirer charm

This guide covers the integration test scenarios you should write to validate
the TLS certificate lifecycle in a requirer charm. By the end you will have
tests that cover:

1. Your charm receives a certificate after integrating with a TLS provider.
2. A certificate is re-requested when your charm's configuration changes.
3. An expiring certificate is automatically renewed.
4. A CA configuration change causes the next certificate to come from the new CA.
5. Rotating the requirer's private key triggers re-issuance.
6. Rotating the CA's private key causes all certificates to be re-issued.

## Prerequisites

- You have a working requirer charm that uses the `tls-certificates` library
  (see [Getting started with the TLS Certificates library](../tutorials/getting-started-tls-certificates-library-v4.md)).
- Configurable CSR attributes using **configuration options** (e.g. `sans_dns`).
- [jubilant](https://pypi.org/project/jubilant/) added to your test dependencies.

## Why use `self-signed-certificates` for testing?

[`self-signed-certificates`](https://charmhub.io/self-signed-certificates) is
the recommended TLS provider for integration testing. It offers several features
that make every lifecycle scenario testable:

| Feature                  | Config / action             | What it enables                      |
| ------------------------ | --------------------------- | ------------------------------------ |
| Short-lived certificates | `certificate-validity`      | Trigger expiry without waiting days  |
| Short-lived CA           | `root-ca-validity`          | Test CA expiry independently         |
| CA identity change       | `ca-common-name`            | Test re-issuance from a new CA       |
| CA key rotation          | `rotate-private-key` action | Test full revocation and re-issuance |

## Expose testable state in your charm

Your charm needs to expose its certificate state and key-rotation capability so
the tests can inspect them. In this guide we do that through actions, but you
could also read the certificate from a file or a Juju secret depending on how
your charm stores them.

Add a **`get-certificate` action** that returns the current certificate, CA
certificate, and chain, and a **`rotate-private-key` action** that calls
`self.certificates.regenerate_private_key()`.

```{note}
`regenerate_private_key()` raises `TLSCertificatesError` if the charm
initialized `TLSCertificatesRequiresV4` with an externally managed
`private_key` argument. Use `import_private_key()` instead in that case.
```

## Set up the test module

The tests all share a single Juju model. Use `jubilant.temp_model()` to create
it automatically and tear it down when the session ends:

```python
import json
import time
import jubilant
import pytest

APP_NAME = "my-charm"  # Replace with your charm's application name.
SSC_APP_NAME = "self-signed-certificates"


@pytest.fixture(scope="module")
def juju(request: pytest.FixtureRequest):
    charm_path = ...  # path to your packed charm

    # The library calls sync() on update-status to detect expiring certificates.
    # A short interval reduces wait time in renewal tests.
    with jubilant.temp_model(config={"update-status-hook-interval": "10s"}) as juju:
        juju.deploy(charm_path, APP_NAME)
        juju.deploy(SSC_APP_NAME, channel="1/stable")
        juju.integrate(f"{SSC_APP_NAME}:certificates", f"{APP_NAME}:certificates")
        juju.wait(
            lambda status: jubilant.all_active(status, APP_NAME, SSC_APP_NAME),
            error=jubilant.any_error,
        )
        yield juju

        if request.session.testsfailed:
            print(juju.debug_log(limit=1000), end="")
```

Add polling helpers that check the `get-certificate` action output:

```python
def _get_certificate(juju: jubilant.Juju) -> dict | None:
    try:
        task = juju.run(f"{APP_NAME}/0", "get-certificate")
        certs = json.loads(task.results.get("certificates", "[]"))
        return certs[0] if certs else None
    except (jubilant.TaskError, json.JSONDecodeError, KeyError):
        return None


def wait_for_certificate(juju, timeout=300) -> dict:
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        cert = _get_certificate(juju)
        if cert and cert.get("certificate"):
            return cert
        time.sleep(5)
    raise TimeoutError("Timed out waiting for a certificate")


def wait_for_new_certificate(juju, previous_certificate: str, timeout=300) -> dict:
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        cert = _get_certificate(juju)
        if cert and cert.get("certificate") != previous_certificate:
            return cert
        time.sleep(5)
    raise TimeoutError("Timed out waiting for a new leaf certificate")


def wait_for_new_ca(juju, previous_ca: str, timeout=300) -> dict:
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        cert = _get_certificate(juju)
        if cert and cert.get("ca-certificate") != previous_ca:
            return cert
        time.sleep(5)
    raise TimeoutError("Timed out waiting for a new CA certificate")
```

## Test: Certificate is received after integration

The most basic test verifies that once the integration is created and both
charms are active, the requirer holds a valid certificate signed by the
provider's CA.

```python
def test_given_charms_are_integrated_then_certificate_is_received(juju):
    cert = wait_for_certificate(juju)

    assert "-----BEGIN CERTIFICATE-----" in cert["certificate"]
    assert "-----BEGIN CERTIFICATE-----" in cert["ca-certificate"]
```

## Test: Certificate is re-requested when config changes

When a CSR attribute changes — for example `sans_dns` — the charm must send a
new CSR and receive a new certificate that reflects the updated value.

```python
def test_given_config_changed_then_new_certificate_is_requested(juju):
    initial = wait_for_certificate(juju)

    juju.config(APP_NAME, {"sans_dns": "new.example.com"})
    juju.wait(lambda status: jubilant.all_active(status, APP_NAME), error=jubilant.any_error)

    new = wait_for_new_certificate(juju, initial["certificate"])

    assert new["certificate"] != initial["certificate"]
```

How quickly this happens depends on how your charm is wired. If you register
`config_changed` as a `refresh_events` when creating `TLSCertificatesRequiresV4`:

```python
self.certificates = TLSCertificatesRequiresV4(
    ...,
    refresh_events=[self.on.config_changed],
)
```

then the new CSR is re-requested on that event, and the new certificate arrives
as soon as the provider processes it. Without a refresh event, the re-request
happens either when `sync()` is called explicitly or when the current
certificate nears expiry.

## Test: An expiring certificate is renewed

Configure `self-signed-certificates` to issue very short-lived certificates,
then wait for the certificate to expire and verify the requirer receives a fresh
one **from the same CA**.

```python
def test_given_certificate_expires_then_it_is_renewed(juju):
    # Issue 1-minute certificates; CA stays valid for 3 minutes.
    juju.config(SSC_APP_NAME, {"certificate-validity": "1m", "root-ca-validity": "3m"})
    juju.wait(
        lambda status: jubilant.all_active(status, APP_NAME, SSC_APP_NAME),
        error=jubilant.any_error,
    )

    initial = wait_for_certificate(juju)

    # Wait just past the 1-minute certificate expiry.
    time.sleep(70)

    renewed = wait_for_new_certificate(juju, initial["certificate"])

    assert renewed["certificate"] != initial["certificate"]
    # The CA must not have changed — only the leaf certificate was renewed.
    assert renewed["ca-certificate"] == initial["ca-certificate"]
```

The library triggers renewal at 90 % of the certificate's lifetime by default
(configurable with `renewal_relative_time`). The `update-status` hook drives
the check, so the short hook interval set at fixture time is important here.

## Test: A CA config change causes a new CA to be used

Changing `ca-common-name` on `self-signed-certificates` generates a new
Certificate Authority and revokes all previously issued certificates. The
requirer must request and receive a certificate signed by the new CA.

```python
def test_given_ca_config_changed_then_new_ca_issues_certificate(juju):
    initial = wait_for_certificate(juju)

    juju.config(SSC_APP_NAME, {"ca-common-name": "new-test-ca.example.com"})
    juju.wait(
        lambda status: jubilant.all_active(status, APP_NAME, SSC_APP_NAME),
        error=jubilant.any_error,
    )

    new = wait_for_new_ca(juju, initial["ca-certificate"])

    assert new["ca-certificate"] != initial["ca-certificate"]
    assert new["certificate"] != initial["certificate"]
```

Any `self-signed-certificates` config option whose description says _"Changing
this value will trigger generation of a new CA certificate"_ (e.g.
`ca-organization`, `ca-country-name`) triggers the same behaviour.

## Test: Rotating the requirer's private key re-issues the certificate

When the requirer regenerates its private key, it sends a new CSR and must
receive a certificate whose public key matches the new private key.

```python
def test_given_requirer_private_key_rotated_then_certificate_is_reissued(juju):
    initial = wait_for_certificate(juju)

    juju.run(f"{APP_NAME}/0", "rotate-private-key")
    juju.wait(lambda status: jubilant.all_active(status, APP_NAME), error=jubilant.any_error)

    new = wait_for_new_certificate(juju, initial["certificate"])

    assert new["certificate"] != initial["certificate"]
```

## Test: Rotating the CA's private key re-issues all certificates

The `rotate-private-key` action on `self-signed-certificates` generates a new
private key and CA, and revokes every previously issued certificate. The
requirer must request and receive a new certificate from the new CA.

```python
def test_given_ca_private_key_rotated_then_certificates_are_reissued(juju):
    initial = wait_for_certificate(juju)

    juju.run(f"{SSC_APP_NAME}/0", "rotate-private-key")
    juju.wait(
        lambda status: jubilant.all_active(status, APP_NAME, SSC_APP_NAME),
        error=jubilant.any_error,
    )

    new = wait_for_new_ca(juju, initial["ca-certificate"])

    assert new["ca-certificate"] != initial["ca-certificate"]
    assert new["certificate"] != initial["certificate"]
```

## Running the suite

The tests share state and should run in the order they appear in the file:

```bash
pytest tests/integration/test_tls_lifecycle.py -v
```
