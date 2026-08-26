# ADR-0013: Default-deny NetworkPolicy for the Python ML enclave, narrow named exceptions only

**Status:** Accepted (Phase 0, audited Phase 7 / SEC-002)

## Context

§14.2 treats `vektor-ml`'s two Python services as untrusted third-party dependencies from the TS application layer's perspective: no direct storage access, isolated namespace, gRPC-only ingress. INFRA-011 needed to turn that into an enforceable Kubernetes control, not just a code-review convention.

## Decision

The `vektor-ml-enclave` namespace starts from a `default-deny-all` NetworkPolicy (empty pod selector, both Ingress and Egress) and adds only three narrow, named exceptions: gRPC ingress to `llm-cloud-svc` from `coa-svc` specifically, metrics-scrape ingress from the observability namespace, and DNS egress.

## Consequences

- `cv-train-svc` (the enclave's other named service) deliberately gets no ingress-allow rule of its own — audited during SEC-002 and confirmed correct, not a gap: it's a CLI-driven batch training job (`cli.py`/`train.py`/`export.py`) with no inbound listener at all. An ingress rule for a service nothing ever calls would be an unused hole.
- SEC-002's audit found and fixed one real over-broad rule: DNS egress originally matched `namespace_selector {}` with no `match_labels` — Kubernetes treats an empty `namespace_selector` block as "any namespace," not "no namespace," so the actual policy was "port 53 to anywhere in the cluster," not "to CoreDNS." Narrowed to a `var.dns_namespace` (default `kube-system`) match.
- Any future third service added to this enclave inherits default-deny automatically and needs its own explicit, reasoned ingress/egress rule the same way `llm-cloud-svc`'s was — there's no ambient "enclave services can reach each other" assumption baked in anywhere.
