---
title: "Traefik Security Decisions"
description: "Positions the Traefik security team has already settled, grouped by the surface a report touches, so recurring reports can be answered with a reference."
---

# Security Decisions

Some classes of report reach us repeatedly. This page records the positions we have already settled on,
so that you can check your finding against them before submitting, and so that we can answer with a
reference instead of a fresh analysis.

It is organised by the surface a report touches, not by our internal taxonomy. Each entry states the
position, then **where the line is**: the neighbouring variant that we do treat as a vulnerability.
That line is the useful part. Most of these surfaces have produced both closures and CVEs.

!!! note "How to read the references"

    We reference published advisories only. Reports closed as not-a-vulnerability stay private to the
    reporter and the security team, so we describe those classes as rules rather than linking cases.
    Where a class has produced a CVE, that CVE is cited: it is the proof that the line is real and not
    a way of declining work.

Read this together with the [threat model](./submitting-security-issues.md#threat-model).

## Path Handling, Normalization and Encoding

**What is usually reported.** A crafted path (`%2e%2e`, `%2f`, double encoding, a literal `;` or
backslash) traverses or escapes a prefix, reaching a route or backend that a middleware was expected to
protect.

**Our position.** This surface is genuinely productive and we treat it seriously. A normalization or
encoding differential that reaches the backend **with stock entrypoint defaults** is a vulnerability.

**Where the line is.** We decline the variants whose escape depends on the operator having widened
handling, or on a specific backend's own re-interpretation of a legal character:

- The escape only works with `allowEncodedSlash` or encoded-character handling left permissive. That
  default was a deliberate, documented compatibility decision, revisited only at a major version.
- The vector is a character that [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986) permits in a path
  segment, such as a literal `;` or backslash, and the escape exists only because one particular
  backend re-interprets it. We do not extend CVE treatment to every literal-byte sibling of a published
  encoded-separator issue.

Before submitting, confirm the bypass works against a normalizing backend with stock entrypoint
configuration.

**Decided in this class.** [CVE-2026-40912](https://www.cve.org/CVERecord?id=CVE-2026-40912) and
[CVE-2026-33186](https://www.cve.org/CVERecord?id=CVE-2026-33186), both dot-segment bypasses in
strip-prefix-class middleware.

## Forwarded Headers and Client Identity

**What is usually reported.** A middleware passes through, or fails to strip, an `X-Forwarded-*` or
`Forwarded` header, letting a client influence what a backend believes about its identity, protocol or
address. Frequently framed as an incomplete fix for a previously published issue, because a sibling
code path does not repeat a strip that was added elsewhere.

**Our position.** Traefik's trust boundary for forwarded headers is **entrypoint-level**.
`forwardedHeaders.trustedIPs` and `forwardedHeaders.insecure` decide once, at the entrypoint, before
any middleware runs, whether a client's forwarding headers are trusted. We deliberately do not re-decide
that per middleware: re-deciding in each middleware is the design mistake we are avoiding, not an
omission. A middleware that passes through a header the entrypoint already accepted is behaving as
designed, and a sibling path that does not repeat a strip is not automatically an incomplete fix.

**Where the line is.** In scope is a path that **reintroduces or reconstructs a trusted value after
entrypoint sanitisation**, so that the entrypoint's decision no longer holds. Also in scope is header
**name** handling that lets an untrusted client reach a header the entrypoint believes it sanitised, for
example through underscore or alias normalization differences between Traefik and a backend.

## Kubernetes References Across Namespaces and Providers

**What is usually reported.** A resource in one namespace, or under one provider, reaches a service,
middleware or transport it should not, through a generated key collision, a `backendRef`, a
`ServersTransport`, or a cross-provider reference.

**Our position.** Two different things get reported here and they get opposite answers.

A **collision or shadowing that the reporter must author themselves** is not a boundary crossing. An
actor who can create or edit the `Ingress`, `IngressRoute`, `TraefikService` or Gateway API objects
involved already controls routing in that namespace. The internal keyspace naming routers, services and
middlewares is not a tenant isolation boundary on a shared instance: hardening it is defence in depth
and does not get an advisory.

A reference that **actually escapes a boundary the operator established**, reaching a namespace or
provider that the configuration did not grant, is a vulnerability.

**Where the line is.** Check whether the allow-list trust anchor is genuinely bypassable for the route
in question. If the traversal is gated by an opt-in that the operator enabled, such as
`crossProviderNamespaces`, that is the documented contract of the option. If it works without that
opt-in, it is in scope.

**Decided in this class.** [CVE-2026-71325](https://www.cve.org/CVERecord?id=CVE-2026-71325),
[CVE-2026-65602](https://www.cve.org/CVERecord?id=CVE-2026-65602),
[CVE-2026-44774](https://www.cve.org/CVERecord?id=CVE-2026-44774) and
[CVE-2026-54761](https://www.cve.org/CVERecord?id=CVE-2026-54761).

## Ingress-NGINX Provider Compatibility

**What is usually reported.** A default or annotation behaviour of the Kubernetes Ingress-NGINX
provider is insecure, demonstrated with a working reproduction.

**Our position.** This provider's contract is annotation compatibility with ingress-nginx. Where it
faithfully reproduces documented upstream semantics, we keep the compatible behaviour even when the
upstream outcome is insecure, and we improve the documentation instead. Changing it would silently break
the migrations the provider exists to serve. Check the upstream behaviour first: if Traefik matches it,
expect a documentation change.

**Where the line is.** Compatibility covers semantics the operator opted into. It does not cover
**authentication or mTLS enforcement silently not happening**. Where the provider fails open, rather
than faithfully reproducing an insecure upstream default, that is a vulnerability and we have treated it
as one.

**Decided in this class.** [CVE-2026-54762](https://www.cve.org/CVERecord?id=CVE-2026-54762),
auth-secret fail-open, and [CVE-2026-53616](https://www.cve.org/CVERecord?id=CVE-2026-53616), client
certificate enforcement not applied.

## Configuration the Operator Controls

**What is usually reported.** A behaviour reachable after enabling a documented option, or in a default
deployment that the documentation tells operators to harden. Dashboard and API reachability, snippet
annotations, error request headers, and variable interpolation are the recurring examples.

**Our position.** Configuration is trusted input, and every provider that supplies it is trusted:
Kubernetes objects, container labels, files, and KV stores. Anyone able to write configuration Traefik
reads already controls routing. A report whose precondition is the ability to write configuration, or
operator or cluster-admin privilege, is not a vulnerability. Where an option is explicitly opt-in, its
documented effect is its contract.

**Where the line is.** In scope is an unprivileged, untrusted client achieving the same effect without
that configuration access, and a documented option that does not actually do what it says.

## Availability and Resource Consumption

**What is usually reported.** A request pattern that exhausts memory, connections or CPU.

**Our position.** Unbounded consumption reachable from unauthenticated requests is in scope. Buffering
and queuing behaviour that stays within documented, configurable limits is not.

**Where the line is.** We require a reproduction rather than a reading of the code, and so should you:
this class has repeatedly turned on whether a per-stream limit actually bounds the per-connection
aggregate. A resource claim without a reproduction against a capped environment cannot be assessed.

## Dependency and Standard Library Findings

**What is usually reported.** A scanner reports a CVE in a Go module or in the standard library that
Traefik builds against.

**Our position.** A dependency finding is an exposure only if Traefik reaches the vulnerable code path
at runtime. Build-time-only and unreachable paths are not vulnerabilities in Traefik, and we confirm
reachability with `govulncheck` before answering.

**Where the line is.** If you can show a reachable call path from a Traefik entrypoint, say so
explicitly in the report: that is the part that determines the answer.

## Non-GA Code

Vulnerabilities that only affect release candidates, betas, or development branches are fixed without a
CVE, as stated in the [CVE policy](./submitting-security-issues.md#cve). Confirm your finding against a
GA release before submitting.
