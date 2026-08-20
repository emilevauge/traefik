---
title: "Traefik Security Documentation"
description: "Security is a key part of Traefik Proxy. Read the technical documentation to learn about security advisories, CVE, and how to report a vulnerability."
---

# Security

## Security Advisories

We strongly advise you to join our mailing list to be aware of the latest announcements from our security team.
You can subscribe by sending an email to security+subscribe@traefik.io or on [the online viewer](https://groups.google.com/a/traefik.io/forum/#!forum/security).

## CVE

Reported vulnerabilities can be found on
[cve.mitre.org](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=traefik).

CVEs are only created for vulnerabilities affecting **Generally Available (GA) versions** of Traefik.
Vulnerabilities discovered in non-GA versions (release candidates, betas, early access, or development branches)
will be fixed without creating a CVE.

## Threat Model

Traefik is an edge router. Its security boundary sits between **untrusted network clients** and the
services it routes to. Whether a report is a vulnerability depends on which side of that boundary the
attacker starts from, and the assumptions below are what we use to decide.

This threat model applies to reports submitted on or after **1 September 2026**. It describes positions
we already apply consistently; publishing them is meant to save reporters the work of rediscovering them.

### What Traefik Trusts by Design

- **The configuration, and every provider that supplies it.** Static configuration, dynamic
  configuration, Kubernetes CRDs and Ingress objects, container labels, files, and KV stores are trusted
  input. Anyone able to write configuration that Traefik reads already controls routing: they can
  redirect traffic, terminate TLS, or remove a middleware, by design. **A report whose precondition is
  the ability to write configuration is not a vulnerability.**
- **The internal configuration keyspace.** On a shared instance, the keyspace naming routers, services
  and middlewares is not a tenant isolation boundary. Collisions and cross-references inside it are
  hardening opportunities, treated as defence in depth, and do not receive an advisory.
- **Operators and cluster administrators.** A report requiring operator, cluster-admin, or equivalent
  privilege does not cross a boundary. In Kubernetes specifically, an actor who can create or edit the
  `Ingress`, `IngressRoute` or `TraefikService` objects involved already controls routing in that
  namespace, so a collision or shadowing they must author themselves is a bug, not a boundary crossing.

### Boundaries the Operator Controls

These are configurable, and where the configuration says "trust this", Traefik trusts it.

- **Client-supplied forwarding headers.** Traefik's trust boundary for `X-Forwarded-*` and `Forwarded`
  is **entrypoint-level**: `forwardedHeaders.trustedIPs` and `forwardedHeaders.insecure` decide once,
  at the entrypoint, before any middleware runs, whether a client's forwarding headers are trusted.
  We deliberately do not re-decide that per middleware. A middleware that passes through a header the
  entrypoint accepted is behaving as designed. The one shape that is in scope is a path that
  **reintroduces or reconstructs a value after entrypoint sanitisation**.
- **Explicitly opt-in permissive settings.** Behaviour reachable only after enabling a documented
  option that widens handling is the documented contract of that option, not a defect. This includes
  `allowEncodedSlash` and encoded-character handling, snippet annotations, cross-provider namespace
  references, and error request headers. Where a permissive default exists for compatibility, that is a
  deliberate decision, changed only at a major version.
- **Dashboard and API exposure.** Securing the dashboard and API is the operator's responsibility, as
  documented. Reachability in a deployment that has not applied that documentation is not a
  vulnerability in Traefik.
- **The Kubernetes Ingress-NGINX provider.** This provider's contract is annotation compatibility with
  ingress-nginx. Where it faithfully reproduces documented upstream semantics, we keep the compatible
  behaviour **even when the upstream outcome is insecure**, and we improve the documentation instead:
  changing it would silently break the migrations the provider exists to serve. Check the upstream
  behaviour first. This covers semantics the operator opted into, and it does **not** cover
  authentication or mTLS enforcement silently not happening: where the provider fails open, that is a
  vulnerability.

### What Is in Scope

Reports that cross the boundary from an unprivileged, untrusted client, with no configuration-write and
no operator privilege required:

- **Routing and matching bypass.** An untrusted request reaching a router, service or backend the
  configuration does not grant it. Path normalization and encoding differentials belong here when the
  transformed path actually reaches the backend, with stock entrypoint defaults.
- **Authentication or authorization middleware failing open**, or being bypassable, under a
  configuration that a reasonable reading of the documentation would call correct.
- **Loss of confidentiality or integrity of traffic Traefik terminates or proxies**, including
  credential and secret exposure, and mTLS or allow-list enforcement that does not hold.
- **Remote crash or unbounded resource consumption** reachable from unauthenticated requests.
- **Escalation across a boundary the configuration explicitly established.**

## Handled as a Bug, Without an Advisory

Some reports describe real defects that we fix, often at high priority, but that do not receive a
security advisory or a CVE, because they do not cross the boundary described above. We say so
explicitly rather than leaving it implicit, and we will point at this section when we close a report.

The recurring classes are below, and each one is developed, with the neighbouring variant we do treat as
a vulnerability, in [Security Decisions](./security-decisions.md).

- **By design, or the operator's responsibility.** The largest class by far. See the trust assumptions
  above.
- **A variant or duplicate of an issue already tracked or published.** The fix lands under the original
  advisory, and you are credited there if it results in a CVE.
- **Behaviour matching comparable projects.** Where Traefik behaves as ingress-nginx or HAProxy does,
  we do not treat it as a Traefik-specific vulnerability. We will show the comparison.
- **Non-GA code only.** Vulnerabilities in release candidates, betas, or development branches are fixed
  without a CVE, as stated above.
- **Dependency findings that are not reachable.** A dependency or Go standard library CVE whose
  vulnerable code path Traefik does not reach, or reaches only at build time, is not an exposure. We
  confirm reachability with `govulncheck`.
- **No working proof of concept, or no reply.** See the submission requirements below.

## Report a Vulnerability

We want to keep Traefik safe for everyone.
If you've discovered a security vulnerability in Traefik,
we appreciate your help in disclosing it to us in a responsible manner,
by creating a [security advisory](https://github.com/traefik/traefik/security/advisories).

## Code of Conduct for Vulnerability Submissions

We are committed to handling every legitimate report responsibly,
and we expect submitters to engage with our security team in a respectful and collaborative manner.

The following behaviors are **not acceptable** and will not be tolerated:

- **Threats** to publicly disclose the vulnerability if it is not fixed within a timeframe you set unilaterally.
- **Ultimatums** or pressure tactics intended to force a faster response than our normal triage and remediation process allows.
- **Demands** for payment, bug bounties, or any form of compensation in exchange for not disclosing the issue
  (Traefik does not operate a paid bug bounty program).
- **Aggressive, abusive, or disrespectful communication** with our security team.

Submitters who engage in any of the above may face the following consequences:

- The submitter **will not be credited** in the security advisory or any subsequent communication.
- The submitter's GitHub profile may be **reported to GitHub** for violation of platform terms of service.
- We may **decline to engage further** on the report, while still addressing the underlying issue if it is legitimate.

We take security seriously and act on legitimate reports as quickly as our resources allow.
Patience and constructive dialogue help us protect users effectively.

## Submission Quality Guidelines

We have been receiving an increasing number of low-quality vulnerability reports that are not actual security issues.
Many of these reports originate from AI/LLM tools and are submitted without any human validation or testing.
This wastes the time of our security team and delays the handling of legitimate vulnerabilities.

Before submitting a security advisory, you **must**:

- **Carefully test and validate** the vulnerability yourself before submitting.
  You must be able to demonstrate a working proof of concept with clear reproduction steps.
- **Understand the impact** of the vulnerability and explain how it can be exploited in a realistic scenario.
- **Verify that the issue is not a false positive**.
  Ensure the behavior you are reporting is actually a security concern and not expected behavior.
  Check it against the [Threat Model](#threat-model) above before submitting.
- **Submit one finding per report.**
  Bundling several unrelated issues into a single advisory makes each one slower to triage, and means a
  single closure decision has to cover claims that deserve different answers.
- **Disclose your use of AI tooling.**
  Using an AI assistant to find or write up an issue is acceptable, and saying so is required. State
  which tool you used and what you verified yourself. An undisclosed AI-generated report that turns out
  to be unvalidated is treated as a breach of these guidelines, not merely a low-quality report.

### Policy on AI-Generated Reports

Security reports that are **directly generated by AI/LLM tools without proper human validation** will be **closed immediately**.

Indicators of unvalidated AI-generated reports include (but are not limited to):

- No working proof of concept or reproduction steps.
- Generic or theoretical vulnerability descriptions with no evidence of actual testing.
- Misunderstanding of Traefik's architecture or threat model.
- Hallucinated code paths, configuration options, or behaviors that do not exist.

**Contributors who repeatedly submit low-quality or unvalidated reports may have their accounts blocked.**

We appreciate the work of security researchers who take the time to rigorously validate their findings.
Quality over quantity helps keep Traefik safe for everyone.
