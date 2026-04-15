# Certificate Renewal with the TLS Certificates Library v4

This feature is only available in v4.

The automated renewals of certificates is abstracted to charm authors and handled by the TLS Certificates Library. Here’s how it works:

1. When the TLS certificates requirer receives a certificate from the TLS certificates provider, it stores it in a Juju secret. The Juju secret is set to expire prior to the certificate expire time.
2. A Certificate Available event is emitted, prompting the requesting charm to store the certificate where it needs to be.
3. When the Juju secret expires, the TLS requirer removes the old CSR from the relation data, removes the Juju secret, generates a new CSR and places this CSR in its relation data.
4. The TLS provider will read this certificate request, generate a certificate and place this certificate in its relation data.
5. The TLS requirer will read this certificate and emit a certificate available event.
