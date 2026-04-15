# Explanation: The TLS Certificates Interface

The whole idea behind the TLS Certificates interface is that charms can request TLS certificates to TLS providers without ever sharing their private key.

The TLS Certificates Requirer (through the use of the TLS Certificates Library) generates its private key and a Certificate Signing Request (CSR). This CSR is inserted into its unit (or application) relation data.

The TLS Certificates Provider reads this CSR, signs a certificate for it and inserts this certificate into its application relation data.

The TLS Certificates Requirer then reads the certificate, and typically stores it in a file on the workload.

```{mermaid}
flowchart LR
	subgraph Requirer
		A[Generate Private Key & CSR]
		B[Insert CSR into relation data]
		C[Read certificate from relation data]
		D[Store certificate in workload]
	end
	subgraph Provider
		E[Read CSR from relation data]
		F[Sign certificate]
		G[Insert certificate into relation data]
	end
	A --> B
	B --> E
	E --> F
	F --> G
	G --> C
	C --> D
```
