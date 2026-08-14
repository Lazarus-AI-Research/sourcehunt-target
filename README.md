# Clearwing minimal vulnerability demos

Minimal, Git-history-free static-analysis corpora for fast Clearwing SourceHunt demos.

Each `CVE-*` directory retains only the vulnerable source and the smallest useful
entry/propagation/sink or value-provenance evidence. These corpora are intended for
vulnerability identification, not standalone builds or exploit reproduction.

Included candidates:

- `CVE-2026-44420` — FreeRDP clipboard capability heap overflow
- `CVE-2026-6040` — LibreOffice number-format import out-of-bounds operation
- `CVE-2026-45796` — Coder Azure identity SSRF
- `CVE-2026-40034` — Gitoxide `.gitmodules` command authorization bypass
- `CVE-2026-28208` — Junrar archive path traversal

Every directory includes a README describing its retained evidence, suggested hunt
scope, omissions, and limitations. Source provenance and checksums are recorded in
the accompanying manifests.
