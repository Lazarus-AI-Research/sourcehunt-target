# CVE-2026-45796 minimal source-hunt reproduction

This directory is a deliberately reduced snapshot of vulnerable Coder commit
`9400eaa957fb019b0084bd1c8599ec0f671f17cb`. It contains one original,
unmodified source file and no `.git` directory.

## Vulnerability path

`Validate` accepts a caller-supplied base64 PKCS#7 Azure identity document:

1. `pkcs7.Parse` parses the attacker-controlled CMS structure.
2. `GetOnlySigner` returns the certificate embedded in that structure.
3. A regular expression only checks the certificate Common Name.
4. `signer.Verify` verifies the X.509 certificate chain, **not** the CMS
   signature over the document or the provenance of the embedded signer.
5. On `x509.UnknownAuthorityError`, the default non-offline path iterates the
   embedded certificate's attacker-controlled `IssuingCertificateURL` values.
6. `http.DefaultClient.Do` sends a GET to each URL, producing SSRF from the
   Coder server.

The same missing `pkcs7Data.Verify()` also leaves `pkcs7Data.Content`
unauthenticated before its `vmId` is returned. That is a closely related
authentication-bypass finding, but the target CVE metadata specifically
describes the SSRF.

The production entry point is
`API.postWorkspaceAuthAzureInstanceIdentity` in the omitted
`coderd/workspaceresourceauth.go`. It passes the request body's `Signature` to
`azureidentity.Validate` with `Offline` left false and includes returned error
detail in the unauthorized response. This caller is summarized here rather
than retained because the large file imports many unrelated authentication,
database, and HTTP API components; it is not needed to identify the flaw in
`Validate`.

## Retained file

- `coderd/azureidentity/azureidentity.go`: complete, byte-for-byte vulnerable
  implementation. It contains the entire security-relevant data flow from
  PKCS#7 parse, signer extraction and validation through the outbound HTTP
  request. The embedded CA material is verbose but retained to keep the source
  authentic and avoid creating a synthetic excerpt.

## Omitted scope

- `azureidentity_test.go`: tests ordinary validation behavior but is not on
  the vulnerable runtime path and does not expose this SSRF.
- `generate.sh`: certificate maintenance helper, never called at runtime.
- `coderd/workspaceresourceauth.go`: endpoint call site, summarized above.
- All other Coder packages, build metadata, vendored dependencies, history,
  and `.git`: unnecessary for static identification of the sink and its
  attacker-controlled provenance.

## Suggested hunt

Use the reproduction root as the repository and select:

```text
subsystem: coderd/azureidentity
```

The hunt needs only inspect `coderd/azureidentity/azureidentity.go`. A useful
review question is: "Can data embedded in an unverified PKCS#7 signer
certificate influence an outbound request?"

## Validation and limitations

Static validation confirms all relevant symbols remain inspectable in one
file: `Validate`, `pkcs7.Parse`, `GetOnlySigner`, `signer.Verify`,
`IssuingCertificateURL`, `http.NewRequestWithContext`, and
`http.DefaultClient.Do`. The file hash matches the vulnerable sparse checkout.

This is an identification reproduction, not a standalone build or executable
PoC. The original module metadata and third-party `go.mozilla.org/pkcs7` and
`xerrors` dependencies were intentionally omitted. Consequently `go test` is
not expected to run here. Exploitability at the network boundary relies on the
documented production handler, and practical reachability still depends on
the deployment's routing and egress policy.
