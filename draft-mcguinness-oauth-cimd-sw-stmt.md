---
title: "Client ID Metadata Document Software Statement"
abbrev: oauth-cimd-sw-stmt
docname: draft-mcguinness-oauth-cimd-sw-stmt-latest
category: std
submissiontype: IETF

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Dynamic Client Registration
 - Client ID Metadata Document
 - Runtime Presentation
 - Sender Constraint

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC6234:
  RFC6585:
  RFC6838:
  RFC7515:
  RFC7519:
  RFC8725:
  RFC7591:
  RFC8414:
  RFC9126:
  RFC7636:
  RFC9449:
  RFC9110:
  RFC8252:
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  IDJAG: I-D.ietf-oauth-identity-assertion-authz-grant

informative:
  RFC7522:
  RFC7523:
  RFC9700:
  RFC8705:
  RFC8628:
  RFC8693:
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "Client ID Metadata Document Software Statement Registration"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-registration-00
  LIFECYCLE:
    target: https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt/draft-mcguinness-oauth-cimd-sw-stmt-lifecycle.html
    title: "Client ID Metadata Document Software Statement Lifecycle"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
  ABCA: I-D.ietf-oauth-attestation-based-client-auth
  OPENID-FED:
    target: https://openid.net/specs/openid-federation-1_0.html
    title: "OpenID Federation 1.0"
    author:
      - ins: R. Hedberg
        role: editor
      - ins: M.B. Jones
      - ins: A.Å. Solberg
      - ins: J. Bradley
      - ins: G. De Marco
      - ins: V. Dzhuvinov
    date: 2026-02-17

--- abstract

A Client ID Metadata Document (CIMD) carries a client's own claims, which can change at any time. This specification defines a software statement in which a trusted party records its review of the exact octets of such a document, and how an authorization server enforces that review when it admits the client at request time, without a registration. The client presents the statement in a token or pushed authorization request, or publishes it where the reviewed document points. Possession of the statement alone does not complete a grant: the presenter is also bound to the reviewed document. Because the document remains the metadata, one review is portable across the authorization servers that trust its issuer. A companion specification covers RFC 7591 registration. Withdrawal, supersession, and keeping grants in step with later reviews are left to a separate lifecycle extension.

--- middle

# Introduction

A Client ID Metadata Document (CIMD) {{CIMD}} identifies a client by a URL and supplies its metadata from a document the client controls. That metadata is the client's own claim and can change at any time. An organization that reviews client software has no interoperable way to record its review of a particular version of the document, or to have authorization servers that trust the organization, but have no relationship with the client, enforce that review.

This specification profiles the software statement (Section 2.3 of {{RFC7591}}) for that purpose. A statement names the client by its document URL and binds the review to the document's exact octets by a digest ({{metadata-digest}}). An authorization server that trusts the statement's issuer ({{issuer-trust}}) admits the client at request time on the reviewed document, and no other, without creating a registration ({{cimd-presentation}}), so one review is portable across every authorization server in the statement's audience.

This specification serves deployments in which a party other than the client decides which software can connect, such as a marketplace or registry that vouches for software to authorization servers it does not operate, or an enterprise that approves software for its users at a service run by someone else. It also serves server-hosted software, such as an agent, that uses the `client_credentials` grant. Admission grants no access to resources: the authorization server's policy, and the resource owner's authorization where a grant involves one, still govern what it issues.

A statement records approval of a client configuration, not of the software's code or behavior ({{what-statement-attests}}). Possession of a statement alone admits nothing: the presenter is bound to the reviewed document ({{sender-constraint}}), though for a public client that binding does not hold against a party that controls the user agent ({{public-client-security}}). Presentations by a native application whose redirection URIs another application on the device could claim are review-only ({{review-only}}). A statement's expiry limits new admissions but ends no open grant, and an older statement that has not expired remains admissible after a newer one is issued ({{enforcement-bounds}}). Withdrawal, supersession, and grants that follow later reviews are left to {{LIFECYCLE}}.

This specification does not define how a client obtains a statement. {{REGISTRATION}} builds on this specification, which does not depend on it, to define consumption of the same statement in an {{RFC7591}} registration request. {{relationship}} relates this specification to other work.

## Protocol Overview

The following non-normative sequence summarizes runtime admission ({{processing}}):

1. Using its CIMD URL as `client_id`, the client presents its statement in a token request or, for a redirect flow, a pushed authorization request, or publishes it at the `software_statements_uri` its document names ({{runtime-presentation}}, {{pulled-statements}}).
2. The authorization server validates the statement ({{validation}}), resolves the reviewed document ({{effective-metadata}}), and verifies the sender constraint against it ({{sender-constraint}}).
3. The request proceeds under that document's metadata, without a registration.
4. The state on which the rest of the grant depends persists as an establishment ({{grant-lifecycle}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology from {{RFC6749}} (OAuth), {{RFC7591}} (client metadata and software statements), and {{CIMD}} (Client ID Metadata Documents).

This specification defines the following terms.

Trusting Authorization Server:
: An authorization server that consumes a software statement, at runtime under this specification or at registration under {{REGISTRATION}}.

Runtime Presentation:
: The consumption of a validated software statement presented in a token request or pushed authorization request, or pulled from where the client's document points ({{pulled-statements}}), without creating a client registration.

Admission:
: A trusting authorization server's acceptance of a client for a request or a registration, either on a valid statement ({{processing}}, {{REGISTRATION}}), after which the reviewed metadata governs its requests, or under the server's policy for CIMD clients it has not reviewed.

Establishment:
: The state a successful runtime presentation creates for the grant it opens, enumerated in {{grant-lifecycle}}.

Proven Key:
: The key an establishment is bound to: the key whose possession the presenter proves during runtime presentation ({{sender-constraint}}), the key it binds under {{public-client-presentation}}, or, for a statement pulled at the authorization endpoint, the key proved at code redemption ({{pull-binding}}).

Effective Expiry:
: The earlier of a statement's `exp` and its `iat` plus the maximum statement lifetime the trusting authorization server honors for its issuer ({{issuer-trust}}). A statement is expired once its effective expiry has passed.

Required Issuer:
: An issuer whose current statement a trusting authorization server requires in one of its tenants for software in configured client identifier namespaces ({{issuer-trust}}).

Current Statement:
: A statement whose effective expiry has not passed and that no extension the server implements, such as {{LIFECYCLE}}, has ended.

Review-Only:
: Describes a presentation, or a pulled statement, that the authorization server evaluates but whose statement counts for nothing in admission ({{review-only}}).

Tenant:
: A partition of an authorization server's configuration and issued tokens, such as one customer's. {{validation}} says how the authorization server determines a request's tenant.

# The Software Statement {#profiles}

The software statement is a compact JWT {{RFC7519}} protected by JWS {{RFC7515}}. Although {{RFC7591}} permits a MAC, a statement issued under this specification MUST use an asymmetric digital signature, and the `none` algorithm and symmetric algorithms MUST NOT be used. Issuers and trusting authorization servers follow {{RFC8725}}.

The JOSE header MUST include the `kid` header parameter, identifying the signing key in the issuer's statement key set ({{issuer-trust}}), and the `typ` header parameter with the value `software-statement+jwt` (Section 3.11 of {{RFC8725}}). An incompatible revision would use a new `typ` value.

A statement asserts that its issuer evaluated the CIMD whose content the digest identifies, and vouches for that document. The JWT payload contains the following claims.

`iss`:
: REQUIRED. The issuer's identifier, a URL using the `https` scheme with no query or fragment component. For an issuer that is an authorization server, it is its issuer identifier as defined by {{RFC8414}}.

`sub`:
: REQUIRED. The exact client identifier URL of the CIMD the issuer reviewed.

`aud`:
: OPTIONAL. One or more authorization server issuer identifiers ({{RFC8414}}) restricting which authorization servers can accept the statement. If the claim is absent, any authorization server that trusts the issuer for the subject can accept it ({{issuer-trust}}).

`aud_tenant`:
: OPTIONAL. A tenant identifier at a trusting authorization server, as {{IDJAG}} defines the claim, naming the tenant in which the statement's decision applies. A statement whose decision is confined to one of a server's tenants MUST carry an `aud` claim naming exactly one authorization server. It MUST also carry an `aud_tenant` claim naming the tenant, unless the issuer and that server have agreed that the server's trust configuration confines the issuer to the tenant ({{issuer-trust}}).

  The absence of `aud_tenant` does not mean that the statement applies in every tenant ({{audience-practice}}).

`statement_uses`:
: OPTIONAL. A JSON array identifying the permitted uses of the statement: `registration` ({{REGISTRATION}}) and `presentation` ({{cimd-presentation}}). If the claim is absent, only `presentation` is permitted. A pulled statement ({{pulled-statements}}) is a `presentation` use.

`iat`:
: REQUIRED. A NumericDate value, which MUST be an integer, giving the time the statement was issued. An issuer MUST ensure that the `iat` claim values of its statements for a given `iss` and `sub` increase strictly, which the selection of a pulled statement ({{pull-selection}}) and extensions such as {{REGISTRATION}} and {{LIFECYCLE}} rely on.

`exp`:
: REQUIRED. A NumericDate value giving the expiration time.

`jti`:
: REQUIRED. A unique identifier for the statement within the issuer's namespace.

`cimd_digest`:
: REQUIRED. The metadata digest ({{metadata-digest}}) of the CIMD the issuer evaluated.

Profiles and extensions can define further claims, such as `status` ({{LIFECYCLE}}).

A statement issued under this specification MUST NOT contain a claim named in the IANA "OAuth Dynamic Client Registration Metadata" registry, and a trusting authorization server MUST NOT take client metadata from a statement's claims.

A CIMD can carry a `software_statement` member (Section 4.3 of {{CIMD}}). A trusting authorization server MUST NOT consume a statement of this profile from that member, and MUST NOT treat that member as satisfying any requirement of this specification. A statement inside the document could not name the document's digest, because it would be part of the octets it digests, so this specification carries statements in requests ({{runtime-presentation}}) or at a separate location ({{pulled-statements}}). Statements of other kinds in the member are out of scope for this specification.

An authorization server implementing this specification MUST reject a statement of this profile in an {{RFC7591}} registration request with the `unapproved_software_statement` error code (Section 3.2.2 of {{RFC7591}}), unless an extension it implements defines consumption of the statement at registration ({{extensions}}). {{legacy-servers}} explains why.

## Metadata Digest {#metadata-digest}

The metadata digest is the unpadded base64url-encoded SHA-256 hash {{RFC6234}} of the retrieved representation body after removal of content coding. The body is not transcoded, normalized, or re-serialized. Any byte order mark or trailing newline is included.

Retrieval for digest purposes follows {{CIMD}} and MUST NOT use content negotiation: the retrieving party sends the `Accept` header field that {{CIMD}} specifies for retrieving the document, or `Accept: application/json` if it specifies none, and no other variant selection. Equal digests show that the issuer and the trusting authorization server obtained the same octets, so any condition that makes retrieval depend on who is asking defeats the comparison.

A `304 Not Modified` response shows that stored octets, and their digest, are still current only if it carries a strong entity tag that matches the one stored with them under strong comparison (Section 8.8.3.2 of {{RFC9110}}). Otherwise, the server retrieves the body again.

## Validating a Statement {#validation}

Before accepting a statement, a trusting authorization server MUST ensure the following:

* The `typ` JOSE header parameter has the value `software-statement+jwt`, and the `kid` header parameter is present.
* The signature verifies under a key trusted for the exact `iss` value, obtained as {{issuer-trust}} requires.
* Every required claim of {{profiles}} is present and of the correct type.
* The statement is not expired, and its `iat` is not unreasonably far in the future according to the server's clock-skew policy.
* If the statement carries `aud`, a value in it exactly matches one of the server's audience identifiers that its trust configuration permits the statement's issuer to name ({{issuer-trust}}).
* If the statement carries `aud_tenant`, `aud` carries exactly one value and `aud_tenant` identifies the request's tenant.
* If the server requires a tenant-scoped decision for the request, the statement carries `aud_tenant`, or the server's trust configuration confines the statement's issuer to the request's tenant ({{issuer-trust}}).
* If the server's trust configuration confines the statement's issuer to a tenant, the request belongs to that tenant.
* The use being made of the statement is one its `statement_uses` claim permits ({{profiles}}).
* The `sub` claim is a client identifier URL conforming to {{CIMD}} and falls within the client identifier namespaces for which this server accepts the issuer ({{issuer-trust}}).
* Any claim the server does not recognize, and any client metadata claim ({{profiles}}), is ignored.
* The JWT validation guidance in {{RFC8725}} has been applied.

A trusting authorization server that hosts more than one tenant MUST evaluate every tenant-specific check, such as the `aud_tenant` claim and a required issuer ({{issuer-trust}}), against the request's tenant. The request's tenant is the tenant that scopes what the server issues on the statement. The server MUST determine it before evaluating the statement, by its own means or from a signed assertion it has validated, and never from a value the client chooses.

Rejections outside a registration endpoint use {{errors}}.

# Issuer Trust {#issuer-trust}

A trusting authorization server MUST derive issuer trust from its trust configuration, which is established out of band, and MUST NOT derive it from an `iss`, `jku`, `x5u`, or other key-location value carried in a statement. This specification defines no in-band issuer discovery or trust decision.

For each trusted issuer, the configuration records at least the following:

* The exact `iss` value the server accepts.
* The signing algorithms it accepts from the issuer.
* The client identifier namespaces the issuer can attest through `sub`.
* The audience identifiers the issuer can name.
* Any tenant to which the server confines the issuer's decisions.
* The maximum statement lifetime the server honors, which sets each statement's effective expiry.

The issuer's statement key set is the JWK Set at the `software_statement_jwks_uri` in its authorization server metadata ({{authorization-server-metadata}}), discovered from the configured `iss` ({{RFC8414}}). For an issuer that is not an authorization server, it is the JWK Set at a statement key set URL in the configuration. It is never the JWK Set at a `jwks_uri` metadata value. Its URL MUST use the `https` scheme, and a trusting authorization server MUST validate the certificate of the host serving it. The issuer MUST NOT publish a key of its statement key set in any other JWK Set, or sign anything other than statements under this specification with those keys. A trusting authorization server reuses a retrieved statement key set only as {{retrieval}} permits.

The configuration can also make an issuer a required issuer for client identifier namespaces in one of the server's tenants ({{trust-configuration}}). The server then MUST NOT admit software in a covered namespace in that tenant without a current statement from the required issuer, whatever its policy for CIMD clients it has not reviewed. Without this rule, a holder could evade the required issuer's decision by withholding that issuer's statement or by presenting or publishing only a broader one, such as a marketplace listing. A required issuer configured for software whose presentations are review-only ({{review-only}}) excludes that software from the tenant, and the server rejects a request for such software there.

## Multi-Tenant Issuers {#multi-tenant-issuers}

If an issuer reviews software for several customers whose decisions are each trusted independently, it MUST issue under a distinct `iss` value for each customer, so that each customer's decisions, and any state a server keeps per `iss` value, stay separate.

# Runtime Presentation {#cimd-presentation}

## Admission {#processing}

For a runtime presentation, the authorization server proceeds as follows, rejecting at the first failure ({{errors}}):

1. Obtain the statement, presented in the request ({{runtime-presentation}}) or pulled ({{pulled-statements}}).
2. Validate the statement ({{validation}}), completing every check that needs no client-controlled retrieval before any such retrieval ({{retrieval}}).
3. Verify the `client_id` rule of {{runtime-presentation}}.
4. Resolve the reviewed document and compare its digest ({{effective-metadata}}). A server that proceeds without the statement on a mismatch ends admission on the statement here.
5. Apply {{sender-constraint}} to that document, verifying the sender constraint unless the statement is review-only ({{review-only}}). If {{sender-constraint}} or {{pull-binding}} defers the proof to code redemption, the server verifies it there.
6. Evaluate the request against the reviewed metadata ({{effective-metadata}}).
7. Create the establishment ({{grant-lifecycle}}), unless the statement is review-only ({{review-only}}).

## Reviewed Metadata {#effective-metadata}

Having validated the statement, and before verifying the proof ({{processing}}), the authorization server MUST obtain the CIMD at the statement's `sub`, only by retrieval or from octets it has already digested for that identifier ({{retrieval}}), and compare its digest with the `cimd_digest` claim. If octets it already holds do not match, it retrieves the document once more before treating the mismatch as a change.

A match means the served document is the reviewed one, and its members are the client's metadata for the request. The server takes those members from the octets whose digest matched, not from another retrieval or a copy it has not digested. A mismatch means the document changed after review, and the authorization server MUST NOT treat the changed document as reviewed. It rejects the presentation or, by local policy, proceeds without the statement. If it proceeds, it applies its policy for CIMD clients it has not reviewed to the current document and creates no establishment for the statement.

A trusting authorization server MUST reject a reviewed document containing duplicate object member names, wherever it resolves one. Parsers interpret such names differently despite an identical digest.

The request is evaluated against the reviewed metadata as against any client's registered metadata ({{RFC6749}}, {{CIMD}}): any requested grant type, response type, or scope MUST fall within it. A document that omits `scope` sets no ceiling on scope beyond the server's own policy. Omitted `grant_types` and `response_types` members take their {{RFC7591}} defaults.

## Retrieval {#retrieval}

Every retrieval this specification causes, of the CIMD, a `jwks_uri` it names, or the statements its `software_statements_uri` names, inherits the server-side request forgery protections in Section 8.6 of {{CIMD}}. A trusted signature does not make a URL safe: the authorization server MUST apply its URL, redirect, address-range, transport, and content-type policy independently to every referenced location.

If a retrieval fails, the authorization server MUST reject the request and MUST NOT fall back to a weaker proof. The exception is a failed pull of statements, or a failed retrieval of a statement key set ({{issuer-trust}}) needed to verify a pulled statement, for a request opening a new grant: that failure does not by itself reject the request, which proceeds as {{pull-selection}} describes.

A retrieval does not complete if it times out, its connection fails, or the response has a 429 (Too Many Requests) status code (Section 4 of {{RFC6585}}) or a 5xx status code. Any other failure to obtain an acceptable representation is final, such as another 4xx status code, a body that is not a JSON object, or a response exceeding the server's configured size bound (Section 8.7 of {{CIMD}}), which the authorization server MUST treat as a failure rather than digesting what it read. After a final failure, a document is treated as failing validation, a key set as carrying no acceptable key, and a statements location as publishing no statement.

A trusting authorization server MAY reuse document octets it holds for digest comparison ({{effective-metadata}}), a key set retrieved from a document's `jwks_uri`, and a statement key set from either source ({{issuer-trust}}), only within ordinary HTTP caching bounds and a maximum reuse period of its own choosing. It MUST NOT let the publisher's cache directives alone (the client's for the document and its key set, the issuer's for a statement key set) determine how long held octets continue to match a statement or a removed key continues to verify ({{statement-validation}}). Without this rule, a key the document carries inline would continue to verify after the publisher removes it.

## Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for a presentation made in a request. The software statement ({{profiles}}). It is consumed as a runtime presentation or in a role defined by an extension the server implements. On refresh-token use under an establishment, a server ignores the parameter unless an extension it implements defines its role there. A request carrying the parameter in any other role is rejected.

A request making a runtime presentation MUST include the `client_id` parameter, even if the client authenticates, and its value, the client's CIMD URL, MUST exactly equal the statement's `sub` claim. The authorization server MUST reject a request that omits the parameter or carries another value. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

An authorization server that does not implement this specification ignores the parameter (Sections 3.1 and 3.2 of {{RFC6749}}).

### Token Endpoint {#token-requests}

At the token endpoint, a runtime presentation is valid on a request that opens a new grant in that request, such as one using the `client_credentials` grant, the JWT or SAML 2.0 assertion grant (Section 2.1 of {{RFC7523}} and Section 2.1 of {{RFC7522}}), or token exchange {{RFC8693}}.

It is not valid on a request that continues or completes a grant an earlier request began, such as authorization-code redemption, refresh-token use ({{grant-lifecycle}}, {{refresh}}), or redemption of a device code {{RFC8628}}. Admission belongs to the request that begins the grant, before the resource owner approves the client.

### Authorization Endpoint {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}. The subsequent authorization request MUST use a `client_id` parameter exactly equal to the establishment's `sub` and MUST NOT include the `software_statement` parameter. A pulled statement needs no pushed authorization request ({{pulled-statements}}).

## Pulled Statements {#pulled-statements}

A client can publish its statements at a URL its document names instead of presenting one. When a client opening a new grant presents no statement, at the token endpoint, the pushed authorization request endpoint, or the authorization endpoint, an authorization server that supports pulled statements MAY retrieve them. Admission then follows {{processing}}, except that:

* The server chooses the statement ({{pull-selection}}).
* If the pull yields no usable statement, the request proceeds as if the client had presented none ({{pull-selection}}).
* A pulled statement is review-only in the cases {{pull-binding}} identifies.

This specification defines the following client metadata member:

`software_statements_uri`:
: OPTIONAL. URL, in a CIMD, at which the client publishes software statements about itself. The member has no meaning in a registration request. The URL MUST use the `https` scheme. A request to it returns a JSON object whose `software_statements` member is an array of statements ({{profiles}}), each a string in JWS compact serialization. The metadata digest covers this member ({{metadata-digest}}).

An authorization server advertises support through `software_statement_pull_supported` ({{authorization-server-metadata}}).

### Retrieval and Selection {#pull-selection}

In retrieving the `software_statements_uri`, the authorization server:

* MUST send a `GET` request with `Accept: application/json`.
* MUST NOT automatically follow redirects.
* MUST accept a new representation only from a `200` response with a JSON media type, and can instead reuse a stored representation that a `304 Not Modified` response revalidates (Section 15.4.5 of {{RFC9110}}).

The document and this URL are necessarily retrieved before any statement is validated, so the ordering of step 2 of {{processing}} applies only to the retrievals that follow.

The server considers only statements that meet all of the following conditions, and ignores the rest:

* The `sub` claim equals the document's client identifier.
* The statement validates under {{validation}}.
* The `cimd_digest` claim equals the digest of the document the server resolved.

If the server's trust configuration requires an issuer for this client ({{issuer-trust}}), the server also ignores statements from any other issuer. If more than one statement remains, it applies the most recently issued statement from an issuer of its choice.

Having selected a statement, the server continues from step 5 of {{processing}}, with the document already resolved. The selection satisfies the `client_id` rule of step 3.

If retrieving the statements does not complete or no statement remains, the server applies its policy for CIMD clients it has not reviewed, subject to any required issuer ({{issuer-trust}}), and rejects the request if that policy or a required issuer requires reviewed software.

### Binding {#pull-binding}

Unless {{review-only}} makes it review-only for its redirection URIs, a pulled statement is bound to the presenter as a presented statement is ({{sender-constraint}}), at the point the following table shows:

| Where the statement is pulled | Confidential client | Public client |
| --- | --- | --- |
| Token endpoint, opening a new grant | The request's client authentication | Review-only |
| Pushed authorization request endpoint | The request's client authentication | PKCE with `S256` and a key bound through a DPoP proof or `dpop_jkt`, as {{public-client-presentation}} requires. Otherwise review-only |
| Authorization endpoint | Client authentication at code redemption, before any token is issued | PKCE with `S256` and a key bound through `dpop_jkt`, as {{public-client-presentation}} requires, which redemption then proves. Otherwise review-only |

A statement pulled for a document that omits `token_endpoint_auth_method` is review-only wherever it is pulled.

At code redemption, a confidential client whose statement was pulled at the authorization endpoint authenticates with a key the reviewed document carries, in its `jwks` or at its `jwks_uri` within the reuse bounds of {{retrieval}}, not one that only a later retrieval of the document carries, and that key becomes the establishment's proven key.

## Sender Constraint {#sender-constraint}

A runtime presentation that is not review-only ({{review-only}}) MUST be sender-constrained under one of two models, set by the `token_endpoint_auth_method` its reviewed document declares:

* Confidential client (a method other than `none`): by a key the reviewed document carries ({{confidential-client-presentation}}).
* Public client (`none`): by an installation key the presenter binds ({{public-client-presentation}}), while its reviewed redirection URIs, with PKCE `S256`, bind the presentation itself ({{public-client-security}}).

The authorization server MUST reject a presentation made in a request ({{runtime-presentation}}) under a reviewed document that omits the `token_endpoint_auth_method` member, even if the presentation is review-only. {{pull-binding}} covers a pulled statement.

For a presentation that is not review-only, under either model:

* The presenter proves possession of the key its model requires through the applicable client authentication method or a DPoP proof {{RFC9449}}. A public client that binds its key through `dpop_jkt` proves possession at code redemption ({{grant-lifecycle}}). If a request carries both client authentication and a DPoP proof, the key its client authentication proves is the proven key.
* The proof MUST be bound to the current request and validated with the replay protections of that mechanism.
* The presenter MUST use the client authentication method its reviewed document declares in `token_endpoint_auth_method` ({{CIMD}}).
* The authorization server MUST reject a presentation without such a proof or, for a confidential client, whose proven key the reviewed document does not carry.

### Confidential Clients {#confidential-client-presentation}

For a confidential client's presentation that is not review-only, the proven key MUST appear in the `jwks` or at the `jwks_uri` of the reviewed document ({{effective-metadata}}). Methods that prove possession of a key the document carries, such as `private_key_jwt` (Section 8.2 of {{CIMD}}) and `self_signed_tls_client_auth` (Section 2.2 of {{RFC8705}}), satisfy this directly.

### Public Clients {#public-client-presentation}

Software distributed to end users cannot hold a key its reviewed document carries: a key shipped in every copy identifies the software, not the installation. Such a document declares `token_endpoint_auth_method` of `none` and carries no key material.

An authorization server MUST reject a presentation made in a request, even a review-only one, that opens a new grant at the token endpoint ({{token-requests}}) from a public client. Such a presentation opens no redirect and has nothing to bind it. {{pull-binding}} covers a pulled statement.

For a public client's presentation that is not review-only, made or pulled at the pushed authorization request endpoint, or pulled at the authorization endpoint ({{pull-binding}}), the authorization server:

* MUST require PKCE {{RFC7636}} with the `S256` method.
* MUST require the presenter to bind a key it holds, through the `dpop_jkt` parameter or, at the pushed authorization request endpoint, a DPoP proof (Section 10.1 of {{RFC9449}}), and records that key as the proven key of the establishment ({{grant-lifecycle}}).
* MUST NOT require that key to appear in the reviewed document.

### Review-Only Presentations {#review-only}

The following are review-only:

1. Every presentation and pulled statement for a reviewed document that carries any redirection URI another application on the same device could claim, whatever authentication method the document declares. Such a URI is a non-`https` URI, such as a private-use scheme or an `http` loopback redirection URI (Sections 7.1 and 7.3 of {{RFC8252}}), or an `https` URI whose host is a loopback address or `localhost`. The application that claims such a URI receives the code, so neither the reviewed redirection URIs nor a key shipped in every copy of the software can bind the presenter.
2. A pulled statement that {{pull-binding}} makes review-only.

A review-only presentation or pulled statement creates no establishment, and its statement MUST NOT satisfy a policy requiring reviewed software: the authorization server proceeds as it would for the same CIMD client presenting no statement ({{ui-audit}}).

## Grant Binding {#grant-lifecycle}

A successful presentation creates an establishment: the state against which the subsequent authorization request, code redemption, and refresh under the grant it opens are checked. It comprises the following:

* The validated `sub`.
* The statement identity (its `iss` and `jti`), its `iat` and effective expiry, and any claim an extension uses to locate it, such as `status`.
* The authorization server's own tenant the grant was opened for, if it hosts more than one.
* The reviewed metadata and the digest it matched ({{effective-metadata}}).
* The issuer trust decision.
* The sender-constraint mechanism and proven key.

The authorization server MUST bind the resulting `request_uri`, authorization code, refresh token, and other grant continuation state to the establishment, as applicable. An establishment ends once no continuation state bound to it remains usable.

A token request that redeems an authorization code bound to an establishment:

* MUST have a `client_id` parameter exactly equal to the establishment's `sub`, even if the client authenticates.
* MUST demonstrate possession of the establishment's proven key under the same sender-constraint mechanism or, for a confidential client whose statement was pulled at the authorization endpoint ({{pull-binding}}), authenticate with the key that becomes the proven key.
* MUST NOT carry the `software_statement` parameter.

A redemption that carries a statement, has a wrong client identifier, or fails the key binding is rejected.

### Refresh {#refresh}

On refresh-token use under an establishment, the authorization server:

* MUST verify possession of the establishment's proven key under the same sender-constraint mechanism.
* MUST, except for an establishment created under {{public-client-presentation}}, verify that the document currently served at the establishment's `sub` still carries that key, in its `jwks` or at its `jwks_uri`, within the reuse bounds of {{retrieval}}.

If either check fails, the server rejects the refresh. The refreshed access MUST fall within the establishment's reviewed metadata.

Under this specification, refresh evaluates no statement ({{enforcement-bounds}}) and never changes the establishment's proven key or reviewed metadata. A client that needs a different key, or, without {{LIFECYCLE}}, a different review, opens a new grant by presentation.

# Error Responses {#errors}

A rejection uses the error response and HTTP status codes of its endpoint: Section 5.2 of {{RFC6749}} at the token endpoint, Section 2.3 of {{RFC9126}} at the pushed authorization request endpoint, and Section 4.1.2.1 of {{RFC6749}} at the authorization endpoint. {{REGISTRATION}} defines rejections at a registration endpoint.

A rejection for a reason this specification defines uses the first of the following error codes that applies:

1. `invalid_request`, if the request carries the `software_statement` parameter in a role {{runtime-presentation}} does not permit.
2. `temporarily_unavailable`, if a retrieval the request needs did not complete ({{retrieval}}), including a pull of statements ({{pulled-statements}}) or the retrieval of a statement key set needed to verify a pulled statement.
3. The code that {{presentation-errors}}, {{continuation-errors}}, or {{authorization-errors}} assigns.

At the token and pushed authorization request endpoints, an authorization server SHOULD respond with HTTP status code 503 to a request it rejects with the `temporarily_unavailable` error code.

A rejection for another reason, such as an assertion grant whose assertion is invalid (Section 3.1 of {{RFC7523}}) or an expired or revoked refresh token, uses the error code its grant or endpoint assigns. An extension such as {{LIFECYCLE}} assigns codes to the rejections it defines.

## Statement Required Error {#statement-required}

This specification defines the following error code:

`statement_required`:
: A current software statement is required and none was presented or pulled, or the one presented or pulled is expired and otherwise valid. An extension such as {{REGISTRATION}} or {{LIFECYCLE}} can assign it to further cases. The client recovers by obtaining a statement issued more recently.

## Presentation Requests {#presentation-errors}

A presentation at the token or pushed authorization request endpoint is rejected with the following error codes:

`invalid_client`:
: Any of the following:

  * The statement is malformed or fails validation ({{validation}}) for a reason other than expiry.
  * The request's `client_id` differs from the statement's `sub` ({{runtime-presentation}}).
  * The request fails the sender constraint ({{sender-constraint}}), including a presentation made in a request under a document that omits `token_endpoint_auth_method`, and one opening a new grant at the token endpoint for a public client ({{public-client-presentation}}).
  * The reviewed document cannot be obtained, including over the size bound, fails validation as {{CIMD}} requires, contains duplicate member names ({{effective-metadata}}), or does not match and the server does not proceed without the statement ({{effective-metadata}}).
  * The bound on establishments is reached ({{establishment-bounds}}).

`statement_required`:
: Expiry is the statement's only validation failure ({{statement-required}}), or a required statement is absent, including one from an issuer required for the tenant ({{issuer-trust}}).

`unauthorized_client`:
: Any of the following, in place of `statement_required` if both apply:

  * The request exceeds the reviewed document's grant types or response types ({{effective-metadata}}).
  * The document carries metadata this server's policy refuses.
  * The client is review-only where policy or a required issuer requires reviewed software ({{review-only}}).
  * The request is for software a required issuer excludes from the tenant ({{issuer-trust}}).

`invalid_scope`:
: The request exceeds the reviewed document's scope ({{effective-metadata}}), or the document carries a scope this server's policy refuses.

`invalid_request`:
: The request omits `client_id` ({{runtime-presentation}}).

## Code Redemption and Refresh {#continuation-errors}

Under an establishment, code redemption ({{grant-lifecycle}}) is rejected with the following error codes:

`invalid_grant`:
: The request has a wrong client identifier or fails the key binding.

`invalid_client`:
: The bound on establishments is reached ({{establishment-bounds}}).

Under an establishment, a refresh-token request is rejected with the following error codes:

`invalid_grant`:
: A check of {{refresh}} fails.

`unauthorized_client` or `invalid_scope`:
: The refreshed access exceeds the reviewed metadata or this server's policy, as for a presentation.

None of these rejections indicates refresh-token replay ({{RFC9700}}), whose detection this specification does not change.

## Authorization Requests {#authorization-errors}

If the authorization server's policy or a required issuer ({{issuer-trust}}) requires reviewed software, an authorization request from a client the server has not admitted on a statement is rejected in the authorization error response (Section 4.1.2.1 of {{RFC6749}}) with the following error codes:

`unauthorized_client`:
: The client is review-only ({{review-only}}), or a required issuer excludes the software from the tenant ({{issuer-trust}}).

`statement_required`:
: In any other case, subject to the order of {{errors}}, the authorization server MUST return this code.

Before redirecting with an error, the authorization server validates the `redirect_uri` against the client's CIMD, which it resolves first, or, for a registered client, against its registered redirection URIs (Section 4.1.2.1 of {{RFC6749}}). If it cannot resolve a CIMD client's document, it MUST NOT redirect.

## Proof Mechanism Errors {#proof-errors}

If the proof mechanism defines an error of its own for a proof it does not accept, such as `invalid_dpop_proof` for a DPoP proof that fails the checks of Section 4.3 of {{RFC9449}} or `use_dpop_nonce` for a nonce challenge (Section 8 of {{RFC9449}}), the request is rejected with that error in place of the code this specification assigns to a failed sender constraint.

A proof that its mechanism accepts but that proves a key other than the one this specification requires, such as a key the reviewed document does not carry or one other than the establishment's proven key, is not an invalid proof but a failed sender constraint.

## Error Descriptions {#error-descriptions}

The authorization server can return non-sensitive diagnostics in the `error_description` parameter to a client it has authenticated, but MUST NOT reveal details of its issuer trust configuration ({{issuer-trust}}), such as the client identifier namespaces an issuer can attest, to an unauthenticated requester.

# Examples {#example}

The examples in this section are non-normative.

## Presenting at the Token Endpoint

The following is an example of a client presenting a statement at the token endpoint of a server holding no record for the client (line breaks for display purposes only). The reviewed document, which is not shown, names a `jwks_uri`, the `private_key_jwt` authentication method, the `client_credentials` grant type, and a `scope` that includes `tools.read`, so the client authenticates with a key that document carries. The `client_id` is the statement's `sub`.

~~~ http
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&scope=tools.read
&client_id=https%3A%2F%2Fclient.example%2Fapp
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjYtMDgifQ...
&software_statement=eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1l...
~~~

The authorization server validates the statement, retrieves the document at the statement's `sub`, checks its digest, verifies the client assertion against a key from that document's `jwks_uri`, and checks that the requested `scope` falls within the document's `scope`. It keeps no registration.

## Publishing a Statement for Servers to Pull

The following is an example of a CIMD that names where its statements are published:

~~~ json
{
  "client_id": "https://client.example/app",
  "redirect_uris": ["https://client.example/cb"],
  "token_endpoint_auth_method": "private_key_jwt",
  "jwks_uri": "https://client.example/jwks.json",
  "software_statements_uri":
    "https://client.example/statements.json"
}
~~~

A request to that URL returns the statements, here one:

~~~ json
{
  "software_statements": [
    "eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1lbnQrand0Iiwi..."
  ]
}
~~~

An authorization server receiving an ordinary authorization request with this `client_id` resolves the document, retrieves the statements, and selects the one from a trusted issuer whose `cimd_digest` matches the document it retrieved. It completes the binding when the client authenticates with `private_key_jwt` at code redemption ({{pull-binding}}).

## Statement Header and Payload

The following is an example of the JOSE header and payload of a statement over the document in the previous example, served exactly as shown there and ending in a single newline:

~~~ json
{
  "typ": "software-statement+jwt",
  "alg": "ES256",
  "kid": "2026-10"
}
~~~

~~~ json
{
  "iss": "https://issuer.example",
  "sub": "https://client.example/app",
  "aud": ["https://as.example"],
  "statement_uses": ["presentation"],
  "iat": 1790812800,
  "exp": 1793404800,
  "jti": "c2a907b34d28",
  "cimd_digest": "vBIuTBeHfYmEC8CPwbfa2a9XciuKfGTEGf-qGij4N_Q"
}
~~~

The statement names one audience and permits only runtime presentation, as omitting `statement_uses` would. It carries no client metadata: the reviewed document supplies it.

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} values:

`software_statement_jwks_uri`:
: REQUIRED for an authorization server that issues statements under this specification. URL of its statement key set ({{issuer-trust}}), a JWK Set containing only the keys with which it signs them.

`software_statement_signing_alg_values_supported`:
: REQUIRED for an authorization server that issues statements under this specification. A JSON array containing the asymmetric JWS `alg` values that the authorization server can use to sign them. The array MUST NOT contain `none` or a symmetric algorithm.

`software_statement_pull_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server retrieves statements from a client's `software_statements_uri` member when admitting a client that presented none ({{pulled-statements}}). If omitted, the default value is `false`.

# Extension Points {#extensions}

The following extensions are left to separate specifications:

* Attested keys: a client attestation {{ABCA}} vouching for a key the reviewed document does not carry. Without one, software whose instances authenticate with keys of their own can neither prove nor register those keys ({{sender-constraint}}, {{REGISTRATION}}).
* Statement conveyance within a client attestation, rather than as a request parameter.
* Runtime presentation in the request that begins a grant at an endpoint this specification does not cover, such as the device authorization endpoint of {{RFC8628}}.
* Partial review, by which an issuer vouches for particular members rather than a whole document ({{metadata-digest}}).
* Statement lifecycle ({{LIFECYCLE}}): withdrawal, supersession, requiring a current statement on refresh, and refresh replacement.

# Relationship to Other Work {#relationship}

This section is non-normative.

## Client Attestation {#relationship-attestation}

A software statement and a client attestation {{ABCA}} are both signed third-party assertions. A statement attests client software and the metadata its issuer evaluated. A client attestation attests a running instance and the key it holds, and is presented with a proof bound to the request.

Neither assertion grants access nor substitutes for the other.

## Identity Assertions {#identity-assertions}

A statement records that a named party reviewed software, not that a particular customer currently permits that software to act in its tenant. A tenant-scoped decision (`aud_tenant`) authorizes no particular user or transaction. When a customer's identity provider mediates a grant, as with a cross-domain identity assertion {{IDJAG}}, it issues assertions per grant only for clients the customer permits. A statement serves when the issuer is not in the request path, or when a customer wants a portable record of what its review covered.

## OpenID Federation {#relationship-federation}

A trust mark {{OPENID-FED}} is also a signed third-party assertion about an entity, but it is resolved through a federation in which both parties enroll. A statement needs no federation: a trusting authorization server configures its issuer directly ({{issuer-trust}}), and the statement binds the exact octets of the client's own document rather than metadata the federation resolves.

# Implementation Considerations {#implementation-considerations}

This section gives guidance to implementers. It adds no requirement.

## Caching and Reuse of Retrieved Content {#caching}

A response from a `software_statements_uri` ought to be cached no longer than the document is. Because {{CIMD}} forbids caching an error response (Section 5.2 of {{CIMD}}), a failed retrieval ({{retrieval}}) is followed by back-off.

## Retrieval and Processing Bounds {#processing-bounds}

An authorization server can bound the retrieval and processing work of {{external-retrieval}} as follows:

* Rate-limit presentations per statement identity, per subject, and per source, and pulls ({{pulled-statements}}) per client identifier and per source, before spending retrieval or storage on a new presentation.
* Refuse a presentation that would create a new establishment, before spending retrieval or storage on it, once a bound on establishments ({{establishment-bounds}}) is reached.
* Bound JWT size and parsing work, concurrent retrievals, response size, and response time.
* Bound the size of a `software_statements_uri` response and the number of statements examined separately from the document's. Several issuers' statements outgrow a document-sized bound, and a host can pad the array.
* Discard pulled statements from unconfigured issuers before verifying any signature.

## Bounding Establishments {#establishment-bounds}

A statement attests client software, identified by `sub`, not its runtime instances, and this specification defines no instance identifier. One unexpired statement can therefore open establishments at each trusting authorization server in its audience, and more than one at each.

A trusting authorization server can bound the number of establishments, and any registrations ({{REGISTRATION}}), derived from one issuer's statements about one piece of software, identified by `iss` and `sub` claims. How it counts toward a bound is local policy, and a multi-tenant server keeps any bound per tenant. A bound stays effective only if what anyone holding a copy of a statement can create does not count toward it: a pushed authorization request and its unredeemed code, which anyone holding a copy of a public client's statement can approve with their own account, or a registration under which no access token has been issued. Otherwise a holder of a copy could exhaust the bound for the software's other instances.

For software admitted under {{public-client-presentation}}, a server can also bound establishments per resource owner, so that one user cannot exhaust a bound the software's other users share.

When a request is refused at a bound ({{presentation-errors}}), the authorization server ought to state that reason in `error_description`, and the client can retry once earlier establishments end ({{grant-lifecycle}}).

## User Interface, Audit, and Logging {#ui-audit}

An authorization server ought not to present the review in a review-only presentation to the user as an assurance about the presenter ({{review-only}}). It can record the statement's issuer for audit and inventory.

Authorization servers ought to avoid logging software statements, which are sensitive in transit and at rest. A statement names reviewed software and, if it carries an audience, that software's intended relationships.

# Deployment Considerations {#deployment-considerations}

This section gives guidance to operators and issuers. It adds no requirement.

## Configuring Issuer Trust {#trust-configuration}

Trust configuration ought to constrain each issuer to the client identifier namespaces it is expected to attest ({{validation}}), for example URLs under the domains of the software publishers it serves. Without such a scope, a compromised or over-broad issuer can mint acceptable statements about any client software.

When a trusting authorization server configures an issuer whose decisions are confined to one of its tenants, as a customer's own issuer is, it ought to record with that configuration the client identifier namespaces for which the issuer's decision is required in that tenant ({{issuer-trust}}).

## Audience and Tenant Claims {#audience-practice}

An issuer ought to name an audience.

An issuer ought to omit the `aud_tenant` claim from a statement whose review applies wherever its `aud` reaches, such as a marketplace listing. The claim is also unnecessary if the authorization server has one tenant, or if the issuer and the server have agreed that the server's trust configuration confines the issuer to a tenant, as they can for an issuer that does not know the tenant's identifier.

## Statement Lifetime and Renewal {#lifetime-practice}

Short lifetimes tighten an issuer's control and increase renewal traffic. A fleet of statements issued together expires together, so issuers ought to stagger expiries or renew ahead of the boundary.

A changed document is always served before a statement over it exists, so presentations under an earlier statement fail the digest comparison ({{effective-metadata}}) until a statement over the changed document exists. A statement's lifetime bounds how long a review stands without being renewed.

Instances need the most recently issued statement from a shared source rather than a copy fixed at build or install time, because the document can change ({{effective-metadata}}).

# Security Considerations {#security-considerations}

The security considerations of {{CIMD}} and Section 5 of {{RFC7591}} apply.

## Assurance Provided by a Statement {#what-statement-attests}

A statement binds an issuer's review to the octets of a CIMD, which describes an OAuth client configuration. It does not bind the software's code, dependencies, or behavior. Software can change while its document stays the same, and the statement remains acceptable until it expires ({{enforcement-bounds}}) or, under {{LIFECYCLE}}, is withdrawn. Assurance about the software itself rests on the issuer's review process. Any change to the document's octets, including one that changes no metadata value, makes statements over the earlier octets fail the digest comparison ({{effective-metadata}}). The digest covers a URL the document carries, such as `logo_uri`, but not the content served at it, which can change after review.

The following table states what admission on a statement does and does not establish:

| Property | Established |
| --- | --- |
| A configured issuer signed the statement for this client identifier and, if the statement names an audience, one that includes this server | Yes |
| The served document is the one the issuer reviewed, octet for octet | Yes |
| The presenter controls a key the reviewed document carries | Yes, before any token is issued, except under {{public-client-presentation}}. A `jwks_uri` attests only the keys' location ({{statement-validation}}) |
| Authorization codes reach only a reviewed redirection URI | Yes, in a redirect flow. For a public client under {{public-client-presentation}} it is the only binding, and it does not hold against a party that controls the user agent ({{public-client-security}}) |
| The running software is the software the issuer reviewed | No; an attested-key extension can add evidence for it ({{extensions}}) |
| The client may access a particular resource | No; the authorization server's policy, and the resource owner's authorization where a grant involves one, decide |

## Statement Theft and Replay {#statement-validation}

A copied statement admits nothing without the sender constraint ({{sender-constraint}}). Except under {{public-client-presentation}} ({{public-client-security}}), admitting only keys the reviewed document carries also prevents a downgrade to a key the issuer did not review. An extension admitting attested keys ({{extensions}}) has to address downgrade itself.

Attesting a `jwks_uri` attests the location, not its contents: a compromised key host can add keys that satisfy the proof without a digest change. If that matters, an issuer attests `jwks` inline and accepts that rotation changes the digest. A server reusing a cached key set ({{retrieval}}) also accepts that a just-removed key can briefly continue to verify.

## Servers That Do Not Implement This Specification {#legacy-servers}

A statement carries no client metadata, so an {{RFC7591}} server that verified one without implementing this specification would take every metadata value from the registration request, attaching the issuer's approval to an attacker's redirection URIs and keys. Keeping statement signing keys in a dedicated statement key set, out of any `jwks_uri`, and away from statements of other profiles ({{issuer-trust}}, {{authorization-server-metadata}}) means such a server finds no key that verifies a statement through the `jwks_uri` in an issuer's {{RFC8414}} metadata. It is exposed only if its operator configures the statement key set in it. For an issuer that is not an authorization server, that configuration is the only route to its keys, so the protection rests on operators recording the statement key set URL only in servers that implement this specification.

## Copied Statements and Public Clients {#public-client-security}

A party holding a copied statement for a public client can open a pushed authorization request in the reviewed software's name. It cannot receive an authorization code that another resource owner approves in a user agent the party does not control ({{public-client-presentation}}), so other users are exposed only to a consent prompt carrying the reviewed software's name and branding. Rate limits ({{processing-bounds}}) and the issuer's choice of audience and lifetime bound that exposure.

A party that controls the user agent receives the code: an application embedding a user agent (Section 8.12 of {{RFC8252}}), or a holder authorizing with its own account. It redeems the code with a PKCE verifier and key of its own choosing. An admission under {{public-client-presentation}} therefore shows where codes are delivered, not which software holds the grant, and a tenant's required issuer ({{issuer-trust}}) cannot keep unreviewed software out for a user who chooses to run it. This is the client impersonation that Section 8.6 of {{RFC8252}} describes for public clients. An attested-key extension ({{extensions}}) would strengthen the binding by giving the instance a key a third party vouches for.

## Tenant Confusion at a Multi-Tenant Authorization Server {#tenant-confusion}

A multi-tenant server that derives the tenant one way to check a decision and another way to scope a token lets a client reach one tenant's resources on another tenant's decision. {{validation}} requires one tenant for both, selected by no value the client chooses.

This specification defines no tenant parameter, and a CIMD URL is the same `client_id` in every tenant, so neither a client nor an issuer can verify that a server honored the tenant rule of {{validation}}.

The `aud_tenant` claim names a tenant at the trusting authorization server, not which of the client's own customers is asking. Software that acts for many customers under one client identifier and key presents the same statement and key for each, so a statement naming a tenant admits the software there on behalf of every customer it serves, and any of those customers can direct the software at another customer's tenant. A statement attests software and cannot prevent this. A deployment that needs a customer-to-tenant binding records it at the trusting authorization server.

## External Retrieval and Resource Exhaustion {#external-retrieval}

The retrievals of {{retrieval}} happen before any client is registered or any user has interacted, so an unauthenticated requester can cause them with one acceptable statement or, at a server that pulls statements, with none. The bounds of {{processing-bounds}} limit this work but do not prevent it.

## Expiry and Open Grants {#enforcement-bounds}

Under this specification, expiry limits new admissions and ends nothing already opened: a grant lasts for the life of its refresh tokens, whatever the statement's lifetime, unless the server finds the proven key gone from the client's document ({{refresh}}). An older statement that has not expired remains admissible after a newer one narrows the review. A deployment that needs withdrawal, supersession, or grants that follow later reviews implements {{LIFECYCLE}}. Removing an issuer from trust configuration, or narrowing what it can attest, does not by itself end a grant or a registration ({{REGISTRATION}}) already admitted on one of its statements. A server ends them by its own means, such as revoking refresh tokens or deleting registrations.

## Observable State {#oracle-considerations}

A `statement_required` refusal at the authorization endpoint ({{authorization-errors}}) travels through the user agent, so anyone who can see the redirection learns that this client has no established standing at this server. The disclosure is accepted because a client that cannot learn why it was refused cannot act. The disclosure carries no detail of the server's issuer trust configuration, which {{error-descriptions}} keeps from unauthenticated requesters.

# Privacy Considerations

The privacy considerations of {{CIMD}} and Section 6 of {{RFC7591}} apply.

A presentation reveals to the authorization server the client's issuer relationship and, if the statement carries an `aud` claim, the other authorization servers the client intends to use. Omitting that claim discloses nothing beyond the review. Pushed authorization requests ({{authorization-requests}}) keep statements out of browser history, referrers, and front-channel logs.

Retrieving the document tells its host that the server is resolving the client. Pulling statements ({{pulled-statements}}) tells the host of `software_statements_uri` the same. That host is an additional party only if the URL is on another origin. Published statements are readable by anyone, so their `aud` and `aud_tenant` claims disclose which servers and tenants a review names. A statement whose audience is sensitive is presented rather than published.

# IANA Considerations {#iana}

## OAuth Parameters Registry

This specification requests registration of the `software_statement` parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}:

Parameter Name:
: `software_statement`

Parameter Usage Location:
: authorization request, token request

Change Controller:
: IETF

Specification Document(s):
: This specification, {{runtime-presentation}} and {{authorization-requests}}

The "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}} already contains a `software_statement` member, which this registration does not affect. The parameter reuses the name because it carries the same {{RFC7591}} artifact.

## Media Type Registration {#media-type}

This specification requests registration of the `application/software-statement+jwt` media type in the IANA "Media Types" registry {{RFC6838}}. {{RFC7591}} defines no media type for software statements. This registration covers only statements under this specification, as its "Applications that use this media type" entry states, and a statement of another profile carries a `typ` value of its own.

Type name:
: application

Subtype name:
: software-statement+jwt

Required parameters:
: n/a

Optional parameters:
: n/a

Encoding considerations:
: 8bit. A software statement is a JWT. JWT values are encoded as a series of base64url-encoded values separated by period ('.') characters, as registered for `application/jwt` in Section 10.3.1 of {{RFC7519}}.

Security considerations:
: See {{security-considerations}} of this specification and Section 11 of {{RFC7519}}.

Interoperability considerations:
: n/a

Published specification:
: This specification

Applications that use this media type:
: Authorization servers and clients that issue, request, or accept software statements under this specification; other profiles are distinguished by their own `typ` values

Fragment identifier considerations:
: n/a

Additional information:
: Magic number(s): n/a. File extension(s): n/a. Macintosh file type code(s): n/a.

Person & email address to contact for further information:
: Karl McGuinness, public@karlmcguinness.com

Intended usage:
: COMMON

Restrictions on usage:
: none

Author:
: Karl McGuinness, public@karlmcguinness.com

Change controller:
: IETF

## JSON Web Token Claims Registry

This specification requests registration of the following values in the IANA "JSON Web Token Claims" registry established by {{RFC7519}}:

Claim Name:
: `cimd_digest`

Claim Description:
: Unpadded base64url-encoded SHA-256 digest of the retrieved octets of the Client ID Metadata Document evaluated during software statement issuance

Change Controller:
: IETF

Specification Document(s):
: This specification, {{profiles}}

Claim Name:
: `statement_uses`

Claim Description:
: Permitted uses of a software statement

Change Controller:
: IETF

Specification Document(s):
: This specification, {{profiles}}

## OAuth Extensions Error Registry

This specification requests registration of the following error code in the IANA "OAuth Extensions Error Registry" established by {{RFC6749}}:

Error Name:
: `statement_required`

Error Usage Location:
: authorization error response, token error response, pushed authorization request error response

Related Protocol Extension:
: Client ID Metadata Document Software Statement

Change Controller:
: IETF

Specification Document(s):
: This specification, {{errors}}

This specification also requests that IANA update the existing `temporarily_unavailable` entry, adding the token error response to its usage locations and this specification as a reference:

Error Name:
: `temporarily_unavailable`

Error Usage Location:
: token error response, in addition to the locations already registered

Related Protocol Extension:
: OAuth 2.0 Authorization Framework

Change Controller:
: IETF

Specification Document(s):
: {{RFC6749}}, this specification ({{errors}})

## OAuth Dynamic Client Registration Metadata Registry

This specification requests registration of the following value in the IANA "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}. A client publishes this member in its CIMD:

Client Metadata Name:
: `software_statements_uri`

Client Metadata Description:
: URL, in a Client ID Metadata Document, at which the client publishes software statements about itself for an authorization server to retrieve. It is not meaningful in a registration request.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{pulled-statements}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following values in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_jwks_uri`

Metadata Description:
: URL of the JWK Set containing only the keys with which the authorization server signs software statements.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_signing_alg_values_supported`

Metadata Description:
: JSON array containing the asymmetric JWS algorithms supported by the authorization server for signing software statements.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_pull_supported`

Metadata Description:
: Boolean value indicating whether the authorization server retrieves software statements from a client's `software_statements_uri`.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-server-metadata}}

--- back

# Acknowledgments
{:numbered="false"}

This mechanism began as the consumption part of a draft on software statement issuance and was split out so that consumption can evolve separately from issuance. It draws on working-group discussion of CIMDs and attestation-based client authentication.
