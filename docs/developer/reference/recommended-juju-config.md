# Recommended Juju Configuration Options for TLS Requirers

For charms that expose public HTTPS endpoints, certificate attributes should be decided at deployment time by users. Therefore, we recommend to have the following Juju configuration options:

- common_name
- sans_dns
- organization
- organizational_unit
- email_address
- country_name
- state_or_province_name
- locality_name

The charm should use those values when instantiating and create the appropriate CertificateRequest object from those.

```python
class TLSRequirerExample(ops.CharmBase):

	def __init__(self, framework: ops.Framework):
		super().__init__(framework)
		self.certificates = TLSCertificatesRequiresV4(
			charm=self,
			relationship_name="certificates",
			certificate_requests=[self._get_certificate_request()],
			mode=Mode.UNIT,
		)

	def _get_certificate_request(self) -> CertificateRequest:
		return CertificateRequest(
			common_name=self._get_config_common_name(),
			sans_dns=self._get_sans_dns(),
			organization=self._get_config_organization(),
			organizational_unit=self._get_config_organizational_unit(),
			email_address=self._get_config_email_address(),
			country_name=self._get_config_country_name(),
			state_or_province_name=self._get_config_state_or_province_name(),
			locality_name=self._get_config_locality_name(),
		)

	def _get_config_common_name(self) -> str:
		return self.model.config["common-name"]

	def _get_sans_dns(self) -> Optional[list[str]]:
		return self.model.config.get("sans-dns")

	def _get_config_organization(self) -> str:
		return self.model.config["organization"]

	def _get_config_organizational_unit(self) -> str:
		return self.model.config["organizational-unit"]

	def _get_config_email_address(self) -> str:
		return self.model.config["email-address"]

	def _get_config_country_name(self) -> str:
		return self.model.config["country-name"]

	def _get_config_state_or_province_name(self) -> str:
		return self.model.config["state-or-province-name"]

	def _get_config_locality_name(self) -> str:
		return self.model.config["locality-name"]
```
