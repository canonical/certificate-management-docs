# Important Update: New Labels for Juju Secrets

A change in private key labeling will allow a requirer with multiple instances of the tls-certificates relation interface to have one private key per relation.

## What’s Changing

The tls-certificates-interface is changing how it labels private keys for certificate request signing (Requirer side)

Previous pattern: `{LIBID}-private-key-{unit_number}`
New pattern: `{LIBID}-private-key-{unit_number}-{relation_name}`

## Impact on Your Deployment

When upgrading to V4.8:

- The library will generate new private keys that will be stored in Juju secrets with the new labels
- This will trigger a certificate rotation process, it removes the current certificate requests and certificates and requests new certificates from the provider.

## Why This Change Was Needed

Before V4.8, charms using multiple instances of the TLS relation interface would get one private key for all instances. This occurred because:

- The previous labeling pattern didn’t account for multiple interface instances
- Juju doesn’t create multiple secrets with the same label for the same owner

This affected charms requiring different certificates for different purposes (e.g., internal unit communication vs. external HTTPS) and instantiate multiple instances of the interface.
