# CIMD Software Statements

This is the working area for a family of individual Internet-Drafts on software statements for OAuth clients identified by a [Client ID Metadata Document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/) (CIMD).

Client ID Metadata Documents let OAuth clients connect without prior registration, but some deployments still require an independent party's approval of the client configuration. These drafts let a reviewer, such as a platform marketplace or an enterprise security team, record that approval in a signed software statement bound to the exact bytes of the client's document, and let any authorization server that trusts the reviewer enforce it at request time, without registering the client.

New to the drafts? The [FAQ](https://github.com/mcguinness/draft-mcguinness-oauth-software-statement-issuance/wiki/FAQ) covers why they exist, how they relate to existing work, what they do not cover, and where they fit.

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
* [OAuth 2.0 Client Attester Endorsement](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-attesters/), which lets a client's metadata document name the attesters it endorses, and which the issuance draft treats as calling for particular scrutiny at review
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
