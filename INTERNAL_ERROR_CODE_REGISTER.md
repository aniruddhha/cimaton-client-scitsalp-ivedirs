# INTERNAL — EP-LICENSE-01 diagnostic code register

**Developer/support only. Exclude this file from customer installers.** Mapping is not cryptographically secret; implementation must not depend on obscurity for security. This document belongs in the private engineering cumulative source ZIP.

Customer popup (identical for all failures):

> The current Windows system configuration could not be validated.

> Error Code: [SYS-XXX]

| Code | Internal classification | Support triage (never shown in popup) |
|---|---|---|
| SYS-101 | License file missing | Verify signed file exists next to plugin DLL and access path |
| SYS-102 | Malformed license or signature invalid | Check file SHA-256, signing key pairing, file integrity |
| SYS-103 | Time-bound license expired | Verify signed expiry and trusted HTTPS time |
| SYS-104 | Trusted HTTPS time unavailable | Check connectivity, proxy, firewall, approved time endpoints |
| SYS-105 | TLS certificate-time validation failure | Check Windows date/time, certificate validation evidence; do not bypass TLS |
| SYS-106 | Signed product identifier mismatch | Check product identifier in signed payload; regenerate with correct product |
| SYS-107 | Signed payload required fields invalid | Check required product field and expiry format for time-bound licenses |
| SYS-199 | Unexpected/unclassified gate failure | Investigate exception path through development diagnostics, never disclose signing secrets |

## Handling and limitations

- Same non-licensing UI text for every blocking reason, only stable code varies.
- These codes are compiled into plugin and are not a secure secret against inspection or reverse engineering.
- The signature of a valid license is always verified before signed payload-specific classifications (SYS-106/107) are possible.
- A missing file is SYS-101. Access denied or I/O issues are an unclassified SYS-199, not incorrectly asserted missing.
- Strict TLS validation remains mandatory; a network outage returns SYS-104 unless the trusted-time implementation observed a specific certificate-time failure (SYS-105).
- Do not ship this register, source code, fixture keys or test fixtures to clients.
- Historical v0.41.5 UAT results do not establish validation of v0.41.6. Run standalone suite and live Cimatron UAT.
