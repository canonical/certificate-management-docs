# Security Explanation in TLS Certificates Interface

The TLS Certificates Interface and library are developed with Security as one of their core values. This document outlines the key security features of the interface and library.

If you discover a security issue, see the [TLS Certificates Interface security policy](https://github.com/canonical/tls-certificates-interface/blob/main/SECURITY.md) for information on how to report the issue.

## Private Key management

In X.509 certificate workflows, the private key is highly sensitive and must remain confidential. As outlined in the [TLS Certificates Interface reference](../tls-certificates-interface.md), the private key never leaves the charm that requires the TLS certificate.

## Encryption at Rest

The TLS Certificates library stores the private key in a Juju secret that can only be read by the charm requiring the TLS certificates or a Juju administrator.

For information on how you can manage Juju secret backends, see [How to manage secret backends](https://documentation.ubuntu.com/juju/3.6/howto/manage-secret-backends/#manage-secret-backends).

## Key Generation Algorithm

The TLS Certificates library generates private keys using the RSA algorithm.

## Key rotation

Charm authors can rotate the private key by calling the `regenerate_private_key` method which will generate a new private key, remove old certificate requests, and send new ones to the TLS provider.
