# CIMD Software Statements

This is the working area for a family of individual Internet-Drafts on software statements for OAuth clients identified by a [Client ID Metadata Document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/) (CIMD).

Client ID Metadata Documents let OAuth clients connect without prior registration, but some deployments still require an independent party's approval of the client configuration. These drafts make that approval portable across authorization servers that trust the reviewer. A reviewer, such as a platform marketplace or an enterprise security team, records its review of the exact bytes of the client's document in a signed software statement. An authorization server that trusts the reviewer admits the client on that review at request time, checks that the presenter is bound to the reviewed document, and applies the review to the rest of the grant, all without registering the client. Review policy and resource authorization remain separate decisions.

New to the drafts? Start with the [FAQ](https://github.com/mcguinness/draft-mcguinness-oauth-software-statement-issuance/wiki/FAQ).

## Why

### Discovery Is Not Approval

CIMD tells an authorization server where a client's metadata lives. When the reviewer and the authorization server are different organizations, or the reviewer should not take part in every grant, each server also needs an interoperable way to answer four questions:

* Who approved this client configuration?
* Is this the configuration they approved?
* Does that approval apply here, and is it still acceptable?
* Is the requester bound to that configuration?

The core draft connects those answers to OAuth processing, so a review made once is enforced the same way by every server that trusts its issuer. That saves each server a separate integration review and leaves less room for an implementation to attach an approval to the wrong metadata.

### What Already Exists

| Existing mechanism | What it provides | What these drafts add |
| --- | --- | --- |
| [RFC 7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2.3) software statements | Signed assertions about client metadata at registration | A statement bound by digest to a separately hosted CIMD, admission without registration, and lifecycle rules |
| [CIMD](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/) | URL-based client identity and metadata, with a software statement allowed inside the document | A statement outside the document, which can bind to its exact bytes, plus presenter checks and rules for continuing the grant |
| [OpenID Federation](https://openid.net/specs/openid-federation-1_0.html) | Third-party trust, metadata policy, and Trust Marks resolved through a federation | A CIMD-specific mechanism with directly configured issuer trust |
| Enterprise-managed authorization through [identity assertion grants](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/) | A policy decision by the enterprise's identity provider on each grant | A portable review for when the reviewer is not in the grant path |

These are the predecessors the drafts build on. Existing specifications provide the ingredients. The drafts specify one combination of them: a review bound to exact CIMD content, admission without registration, presenter binding, and enforcement of the review's lifecycle.

### Why Now

The [MCP authorization specification](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) recommends CIMD and deprecates Dynamic Client Registration, which it keeps only for backward compatibility. Where registration was the point at which an operator approved a client, that checkpoint goes away, but the need for approval does not. CIMD already allows a software statement inside the document, but a statement there cannot bind the review to the bytes that carry it. Specifying the binding now can help implementations converge before they build incompatible approval formats and enforcement rules.

## Scope and Limits

* A statement approves an OAuth client configuration: the redirection URIs, keys, grant types, and scope in the document. It does not attest the software's code, dependencies, or behavior. Software can change while its document stays the same, and a byte change with no effect on the metadata invalidates the statement. Assurance about the software itself comes from the reviewer's process and, where available, attestation of the running instance.
* The drafts do not address prompt injection, misuse of tools, or what an admitted client does with the access it is granted.
* Under the core draft, native and command-line clients that use loopback or private-use redirects get only a record of the review, because another application on the device could receive their authorization codes. The attested-keys draft addresses this under additional assumptions: an attester the authorization server trusts, typically on a managed device.
* The drafts do not make authorization servers agree on review quality, liability, or who may act as a reviewer. They let a server act on a reviewer it already trusts.

## When It Helps

Portability matters only if independent authorization servers want to rely on someone else's review. The strongest fit is a marketplace, enterprise reviewer, or ecosystem authority whose decisions several independently operated authorization servers consume. Other deployments may not need it:

* A single operator with its own list of approved clients can use local policy.
* An enterprise identity provider that decides on every grant already answers the question for each request.
* An established federation can carry the review in its own trust and metadata mechanisms (see the sketch below).

## The Drafts

| Draft | What it defines | Editor's copy |
| --- | --- | --- |
| **CIMD Software Statement** | The statement and its digest binding, validation, issuer trust, and runtime admission by a statement presented in a request or pulled from where the client's document points | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt.txt) |
| CIMD Software Statement Registration | Consuming a statement in an RFC 7591 registration: metadata taken from the reviewed document, validity bounded by the statement, and renewal by a replacement statement | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-registration.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-registration.txt) |
| CIMD Software Statement Issuance | How a client obtains a statement: token exchange, a redirect flow returning a `software_statement_code`, and asynchronous review under Deferred Token Response | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-issuance.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-issuance.txt) |
| CIMD Software Statement Status | Optional withdrawal mechanisms: Token Status List as the authority on whether a statement stands, and a Shared Signals event telling an authorization server to check at once rather than on its next scheduled check | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-status.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-status.txt) |
| CIMD Software Statement Attested Keys | An extension in which an attester the authorization server trusts attests an installation's key for the reviewed document, so a native application, typically on a managed device, can use loopback redirects through PAR, with the same binding enforced at code redemption and refresh | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-attested.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-attested.txt) |

The statement draft is the core. The other drafts build on it, and it depends on none of them.

The statement and registration drafts are planned for submission; their Datatracker pages will be available after first submission:
[statement](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt/),
[registration](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration/).
The issuance, status, and attested-keys drafts are working drafts kept here for discussion and are not planned for submission yet.

## What Would Move This Forward

The next evidence needed is an end-to-end interoperability test: one real reviewer issuing statements, two independently implemented authorization servers consuming them, and a demonstration of admission, metadata replacement, and withdrawal. If you operate a reviewer, a marketplace, or an authorization server and want to take part, please open an issue.

For the working drafts:

* Status is optional. Deployments that cannot accept a withdrawal taking effect only at expiry would establish the need for it.
* Issuance can wait for independent issuer implementations, since consuming statements interoperates even when reviewers use different issuance workflows.
* Attested keys needs practical evidence that attesters and protected key use deliver the assurance the draft assumes.

## MCP Profile

[Reviewed Client Software](mcp/reviewed-client-software.mdx) is a draft MCP authorization extension, written in the format of MCP's extensions repository for proposal there. MCP authorization servers pull statements from the `software_statements_uri` a client's metadata document names, treat desktop clients with loopback redirects as review-only, and compose the review with Enterprise-Managed Authorization.

## Background and Sketches

These documents explore deployments and edge cases. They are non-normative and not drafts.

| Document | What it covers |
| --- | --- |
| [Deployment Model](sketches/deployment-model.md) | A provider marketplace deciding which software may exist as a client, an enterprise deciding which of it may operate in its tenant, and the layers that keep those decisions separate |
| [A Mobile App, End to End](sketches/mobile-app-sketch.md) | An app store install through review, administrator approval, sign-in, and a third-party SaaS reached through an identity assertion, ending at what a per-install key still lacks |
| [A Customer Approving a Vendor Integration](sketches/saas-integration-sketch.md) | An exploratory vendor-to-vendor integration with no user present, comparing the customer's own statement, a customer-issued workload assertion, and an approval the platform pulls from the customer's root, with the prerequisites each needs |
| [Tenant Admin Consent](sketches/admin-consent-sketch.md) | Microsoft Entra ID's tenant-wide admin consent, compared part by part with open specifications, and what this family changes and leaves alone |
| [Composing with OpenID Federation](sketches/openid-federation-sketch.md) | Issuer trust from a federation, a review carried as a Trust Mark and the mapping that would need, and why Federation's resolved metadata cannot carry a digest |
| [Design Rationale: Pulled Statements and the MCP Profile](sketches/mcp-profile-sketch.md) | Why statements are pulled from where the client's document points, what that removes and what it keeps, and how the MCP profile composes with Enterprise-Managed Authorization |
| [Verification Scenarios for Attested Instance Keys](sketches/attested-verification-scenarios.md) | An implementation checklist for the attested draft, with prerequisites and expected results |

## Related Specifications

Dependencies:

* [OAuth Client ID Metadata Document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/), for every draft
* [Token Status List](https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list/), for the status draft only
* [OAuth 2.0 Attestation-Based Client Authentication](https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth/), for the attested-keys draft
* [Identity Assertion Authorization Grant](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/), for the `aud_tenant` claim
* [Deferred Token Response](https://datatracker.ietf.org/doc/draft-ietf-oauth-deferred-token-response/), for asynchronous issuance only
* [OpenID Shared Signals Framework](https://openid.net/specs/openid-sharedsignals-framework-1_0.html), for the status draft only

Related work:

* [Client Instance Identification for Attestation-Based Client Authentication](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-id/), which identifies instances within one client
* [OpenID Federation 1.0](https://openid.net/specs/openid-federation-1_0.html), sketched above
* [OAuth Identity Assertion Trust Framework](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework/), which generalizes issuer trust

## Contributing

See the [guidelines for contributions](CONTRIBUTING.md).

Contributions can be made by creating pull requests. The GitHub interface supports creating pull requests using the Edit (✏) button.

## Command Line Usage

Formatted text and HTML versions of the drafts can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed. See [the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).
