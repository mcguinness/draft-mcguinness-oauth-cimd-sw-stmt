---
title: "Client ID Metadata Document Software Statement"
abbrev: oauth-cimd-sw-stmt
docname: draft-mcguinness-oauth-cimd-sw-stmt-latest
category: std

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
  RFC7522:
  RFC7523:
  RFC7591:
  RFC8414:
  RFC9126:
  RFC7636:
  RFC9449:
  RFC9700:
  RFC9110:
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  IDJAG: I-D.ietf-oauth-identity-assertion-authz-grant

informative:
  RFC7009:
  RFC8705:
  RFC8252:
  STATUSLIST: I-D.ietf-oauth-status-list
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "Client ID Metadata Document Software Statement Registration"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-registration-00
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

A Client ID Metadata Document (CIMD) carries a client's own claims, which can change at any time. This specification defines a software statement in which a trusted party records its review of the exact bytes of such a document, and how an authorization server enforces that review when it admits the client at request time, without a registration. The client presents the statement in a token or pushed authorization request, or publishes it where the reviewed document points. Possession of the statement alone does not complete a grant: the presenter is also bound to the reviewed document. Because the document remains the metadata, one review is portable across the authorization servers that trust its issuer. A companion specification covers RFC 7591 registration.

--- middle

# Introduction

A Client ID Metadata Document (CIMD) {{CIMD}} identifies a client by a URL and supplies its metadata from a document the client controls. That metadata is the client's own claim and can change at any time. An organization that reviews client software has no interoperable way to record its review of a particular version of the document, or to have authorization servers that trust the organization, but have no relationship with the client, enforce that review.

This specification profiles the software statement (Section 2.3 of {{RFC7591}}) for that purpose. A statement names the client by its document URL and binds the review to the document's exact octets by a digest ({{metadata-digest}}). An authorization server that trusts the statement's issuer ({{issuer-trust}}) admits the client at request time on the reviewed document, and no other, without creating a registration ({{cimd-presentation}}), so one review is portable across every authorization server in the statement's audience.

This specification serves deployments in which a party other than the client decides which software can connect, such as a marketplace or registry that vouches for software to authorization servers it does not operate, or an enterprise that approves software for its users at a service run by someone else. It also serves server-hosted software, such as an agent, that uses the `client_credentials` grant. Admission grants no access to resources: the authorization server's policy, and the resource owner's authorization where a grant involves one, still govern what it issues.

Presentations by a native application whose redirection URIs another application on the device could claim are review-only ({{review-only}}) until an extension binds an installation key that a third party vouches for ({{extensions}}).

A statement records approval of a client configuration, not of the software's code or behavior ({{what-statement-attests}}). Possession of a statement alone admits nothing: the presenter is bound to the reviewed document ({{sender-constraint}}), though for a public client that binding does not hold against a party that controls the user agent ({{public-client-security}}).

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
: A trusting authorization server's acceptance of a client on a valid statement, at request time ({{processing}}) or, under {{REGISTRATION}}, at registration, after which the client's requests are evaluated against the reviewed metadata.

Establishment:
: The durable state a successful runtime presentation creates for the grant it opens, or that a replacement accepted on refresh gives that grant, enumerated in {{grant-lifecycle}}.

Recorded Statement:
: The statement an establishment records: the one presented or pulled to create it or, after a refresh accepts a replacement, the replacement most recently accepted for it ({{refresh}}).

Proven Key:
: The key an establishment is bound to: the key whose possession the presenter proves during runtime presentation ({{sender-constraint}}), the key it binds under {{public-client-presentation}}, or, for a statement pulled at the authorization endpoint, the key proved at code redemption ({{pulled-statements}}).

Effective Expiry:
: The earlier of a statement's `exp` and its `iat` plus the maximum statement lifetime the trusting authorization server honors for its issuer ({{issuer-trust}}). A statement is expired once its effective expiry has passed.

Refusal Record:
: State an authorization server holds when it has learned that a statement ceased to be acceptable before its expiry, whether from a withdrawal mechanism it uses for the issuer ({{withdrawal}}) or from its own operator.

Required Issuer:
: An issuer whose current statement a trusting authorization server requires in one of its tenants for software in configured client identifier namespaces ({{issuer-trust}}).

Current Statement:
: A statement whose effective expiry has not passed and that matches no refusal record. The `iat` floor ({{multi-instance}}) separately limits which current statements a server accepts.

Review-Only:
: Describes a presentation, or a pulled statement, that the authorization server evaluates but that admits nothing ({{review-only}}).

Tenant:
: A partition of an authorization server's configuration and issued tokens, such as one customer's, that the server identifies by its own means and never by a value the client chooses ({{validation}}).

`iat` Floor:
: For a trusting authorization server, the greatest `iat` claim value among the statements it has accepted for a given `iss` and `sub`, per tenant on a multi-tenant server ({{multi-instance}}).

# The Software Statement {#profiles}

The software statement is a compact JWT {{RFC7519}} protected by JWS {{RFC7515}}. Although {{RFC7591}} permits a MAC, a statement issued under this specification MUST use an asymmetric digital signature, and the `none` algorithm and symmetric algorithms MUST NOT be used. Issuers and trusting authorization servers follow {{RFC8725}}.

The JOSE header MUST include the `kid` header parameter, identifying the signing key in the issuer's statement key set ({{issuer-trust}}), and the `typ` header parameter with the value `software-statement+jwt` (Section 3.11 of {{RFC8725}}). Extensions can add claims. An incompatible revision would use a new `typ` value.

A statement asserts that its issuer evaluated the CIMD whose content the digest identifies, and vouches for that document. The JWT payload contains the following claims.

`iss`:
: REQUIRED. The issuer's identifier, a URL using the `https` scheme with no query or fragment component. For an issuer that is an authorization server, it is its issuer identifier as defined by {{RFC8414}}.

`sub`:
: REQUIRED. The exact client identifier URL of the CIMD the issuer reviewed.

`aud`:
: OPTIONAL. One or more authorization server issuer identifiers ({{RFC8414}}) restricting which authorization servers can accept the statement. If the claim is absent, any authorization server that trusts the issuer for the subject can accept it ({{issuer-trust}}).

`aud_tenant`:
: OPTIONAL. A tenant identifier at a trusting authorization server, as {{IDJAG}} defines the claim, naming the tenant in which the statement's decision applies. A statement whose decision is confined to one of a server's tenants MUST carry an `aud` claim naming exactly one authorization server and, unless the issuer and that server have agreed that the server confines the issuer to the tenant through its own trust configuration ({{issuer-trust}}), an `aud_tenant` claim naming the tenant.

  The absence of `aud_tenant` does not mean that the statement applies in every tenant ({{audience-practice}}).

`statement_uses`:
: OPTIONAL. A JSON array identifying the permitted uses of the statement: `registration` ({{REGISTRATION}}) and `presentation` ({{cimd-presentation}}). If the claim is absent, only `presentation` is permitted. A delivery renewing a registration ({{REGISTRATION}}) is a `registration` use. A refresh replacement ({{refresh}}) and a pulled statement ({{pulled-statements}}) are `presentation` uses.

`iat`:
: REQUIRED. A NumericDate value, which MUST be an integer, giving the time the statement was issued. An issuer MUST ensure that the `iat` claim values of its statements for a given `iss` and `sub` increase strictly ({{multi-instance}}, {{refresh}}).

`exp`:
: REQUIRED. A NumericDate value giving the expiration time.

`jti`:
: REQUIRED. A unique identifier for the statement within the issuer's namespace.

`cimd_digest`:
: REQUIRED. The metadata digest ({{metadata-digest}}) of the CIMD the issuer evaluated.

`status`:
: OPTIONAL. The `status` claim as {{STATUSLIST}} defines it, by which a statement locates itself in a Status List Token. Only a withdrawal mechanism that uses the claim processes it. This specification does not ({{withdrawal}}).

A statement issued under this specification MUST NOT contain a claim named in the IANA "OAuth Dynamic Client Registration Metadata" registry, and a trusting authorization server MUST NOT take client metadata from a statement's claims. A trusting authorization server ignores such a claim rather than rejecting the statement ({{validation}}).

A CIMD can carry a `software_statement` member (Section 4.3 of {{CIMD}}). A trusting authorization server MUST NOT consume a statement of this profile from that member, and MUST NOT treat that member as satisfying any requirement of this specification. A statement inside the document could not name the document's digest, because it would be part of the octets it digests, so this specification carries statements in requests ({{runtime-presentation}}) or at a separate location ({{pulled-statements}}). Statements of other kinds in the member are out of scope for this specification.

An authorization server implementing this specification MUST reject a statement of this profile in an {{RFC7591}} registration request with the `unapproved_software_statement` error code (Section 3.2.2 of {{RFC7591}}), unless it consumes the statement as {{REGISTRATION}} specifies. The server's configured statement key set ({{issuer-trust}}) verifies the statement, so a generic registration path would otherwise attach the issuer's review to metadata the request supplies ({{legacy-servers}}).

## Metadata Digest {#metadata-digest}

The metadata digest is the unpadded base64url-encoded SHA-256 hash {{RFC6234}} of the retrieved representation body after removal of content coding. The body is not transcoded, normalized, or re-serialized. Any byte order mark or trailing newline is included.

Retrieval for digest purposes follows {{CIMD}} and MUST NOT use content negotiation: the retrieving party sends the `Accept` header field that {{CIMD}} specifies for retrieving the document, or `Accept: application/json` if it specifies none, and no other variant selection. Equal digests show that the issuer and the trusting authorization server obtained the same octets, so any condition that makes retrieval depend on who is asking defeats the comparison.

A trusting authorization server MUST treat a response exceeding its configured size bound (Section 8.7 of {{CIMD}}) as a retrieval failure rather than digesting what it read.

A server MUST NOT compare a digest against any representation other than one it retrieved itself or stored under this section. A `304 Not Modified` response shows that stored octets, and their digest, are still current only if it carries a strong entity tag that matches the one stored with them under strong comparison (Section 8.8.3.2 of {{RFC9110}}). Otherwise, the server retrieves the body again.

## Validating a Statement {#validation}

Before accepting a statement, a trusting authorization server MUST ensure the following:

* The `typ` JOSE header parameter has the value `software-statement+jwt`.
* The signature verifies under a key trusted for the exact `iss` value, obtained as {{issuer-trust}} requires.
* Every required claim of {{profiles}} is present and of the correct type.
* The statement is not expired, and its `iat` is not unreasonably far in the future according to the server's clock-skew policy.
* If the statement carries `aud`, a value in it exactly matches one of the server's audience identifiers that its trust configuration permits the statement's issuer to name ({{issuer-trust}}).
* If the statement carries `aud_tenant`, `aud` carries exactly one value and `aud_tenant` identifies the tenant this request belongs to, resolved before the statement is evaluated, by the server's own means or from a signed assertion it has validated, and never from a value the client chooses.
* If the server requires a tenant-scoped decision for the request, the statement carries `aud_tenant`, or the server's trust configuration confines the statement's issuer to the request's tenant ({{issuer-trust}}).
* If the server's trust configuration confines the statement's issuer to a tenant, the request belongs to that tenant.
* The use being made of the statement is one it permits: a use its `statement_uses` claim names or, if the claim is absent, presentation ({{profiles}}). A value the server does not recognize names no use it serves.
* The `sub` claim is a client identifier URL conforming to {{CIMD}} and falls within the client identifier namespaces for which this server accepts the issuer ({{issuer-trust}}).
* Any claim the server does not recognize, and any client metadata claim ({{profiles}}), is ignored.
* The JWT validation guidance in {{RFC8725}} has been applied.

A trusting authorization server that hosts more than one tenant MUST evaluate every tenant-specific check, such as the `aud_tenant` claim and a required issuer ({{issuer-trust}}), against the tenant that scopes what it issues on the statement, never selected by a value the client chooses.

A trusting authorization server MUST reject a statement for which it holds a refusal record ({{withdrawal}}). A server that uses a withdrawal mechanism for the statement's issuer SHOULD consult it before accepting the statement for a new admission, within the caching and freshness rules that mechanism defines.

A trusting authorization server resolving the reviewed document MUST reject a document containing duplicate object member names. Parsers interpret such names differently despite an identical digest.

Rejections outside a registration endpoint use {{errors}}.

## Withdrawal {#withdrawal}

An issuer can end a decision before the statement's `exp` by withdrawing the statement. Unlike OAuth token revocation ({{RFC7009}}), a withdrawal revokes no issued token: a server holding a refusal record refuses new admissions and refresh under the statement ({{refresh}}, {{enforcement-bounds}}).

How a trusting authorization server learns of a withdrawal is a withdrawal mechanism. This specification requires no withdrawal mechanism and defines none. Extensions define how a mechanism applies to these statements ({{extensions}}).

A refusal derived from a withdrawal mechanism lasts until a later authoritative result from that mechanism supersedes it, or until an expiry the mechanism itself defines for the withdrawal. A server MUST NOT retain such a refusal once it is superseded or expired in that way. A failure to consult the mechanism, or cached evidence that has lapsed, neither supersedes nor expires a refusal.

A withdrawal constrains and never relaxes. A statement for which no withdrawal mechanism reports a withdrawal, or for which the server cannot consult the mechanism it uses ({{withdrawal-practice}}), is bounded by its effective expiry as it would be otherwise, and a server MUST NOT treat any withdrawal mechanism as grounds to accept a statement past its effective expiry.

An issuer that uses a withdrawal mechanism and issues a replacement narrower than the statement it replaces SHOULD withdraw the earlier statement. A replacement is narrower if the document it names no longer carries a key, redirection URI, or scope the earlier document carried, if its `aud`, `aud_tenant`, or `statement_uses` is narrower, or if its remaining lifetime is shorter.

Without that withdrawal, a trusting authorization server that has not yet seen the replacement, or at which the replacement does not validate, accepts the earlier statement until it expires.

## Repeated Consumption of One Statement {#multi-instance}

A statement attests client software, identified by `sub`, not its runtime instances. This specification defines no instance identifier. One unexpired statement is therefore consumable more than once: at each trusting authorization server in its audience, in more than one establishment at that server ({{establishment-bounds}}), and, if the server registers clients ({{REGISTRATION}}) and its policy permits, in more than one registration there.

A trusting authorization server MUST retain, per `iss` and `sub` claims and, on a multi-tenant server, per tenant, the greatest `iat` claim value among the statements it has accepted, as its `iat` floor. A statement is accepted for the floor once the server admits a client on it, applies it as a replacement ({{refresh}}), or registers a client with it. The server MUST reject a statement whose `iat` claim is earlier than the retained `iat`, whether it arrives as a replacement, a new registration, or a new presentation. A client holding a superseded, broader statement therefore cannot defer a narrowing by opening a fresh registration or establishment with it. The server retains the `iat` floor at least as long as the maximum statement lifetime it honors for that issuer.

A multi-tenant server also keeps any bound on establishments ({{establishment-bounds}}) per tenant.

The first presentation of a replacement that establishes the client supersedes its predecessor for new admissions by every instance of the software at that server ({{lifetime-practice}}). Grants the predecessor already opened continue under {{refresh}}.

Software whose instances authenticate as the client with keys of their own, which the reviewed document does not carry, can neither prove nor register those keys ({{sender-constraint}}, {{REGISTRATION}}) and depends on the attested-key extension of {{extensions}}.

# Issuer Trust Establishment {#issuer-trust}

A trusting authorization server accepts statements only from issuers in its trust configuration, which is established out of band. This specification defines no in-band issuer discovery or trust decision. The configuration, not the signature alone, defines acceptance: a trusting authorization server MUST derive trust from it and MUST NOT derive it from an `iss`, `jku`, `x5u`, or other key-location value carried in a presented statement.

For each trusted issuer, the configuration records at least the exact `iss` value the server accepts, the signing algorithms it accepts from the issuer, the client identifier namespaces the issuer can attest through `sub`, the audience identifiers the issuer can name, any tenant to which the server confines the issuer's decisions, and the maximum statement lifetime the server honors, which sets each statement's effective expiry.

The issuer's statement signing keys come from the `software_statement_jwks_uri` in its authorization server metadata ({{authorization-server-metadata}}), discovered from the configured `iss` ({{RFC8414}}), or, for an issuer that is not an authorization server, from a statement key set URL in the configuration, never from a `jwks_uri` metadata value. A configured statement key set URL MUST use the `https` scheme, a trusting authorization server MUST validate the certificate of the host serving it, and the issuer MUST NOT place in that key set a key from any other JWK Set it publishes, or sign anything other than statements under this specification with its keys. A trusting authorization server reuses a retrieved statement key set only as {{retrieval}} permits.

The configuration can also make an issuer a required issuer for client identifier namespaces in one of the server's tenants ({{trust-configuration}}). The server then MUST NOT admit software in a covered namespace in that tenant without a current statement from the required issuer, whatever its policy for clients it has not reviewed. The requirement also applies on refresh ({{refresh}}). Without this rule, a holder could evade the required issuer's withdrawal by withholding that issuer's statement or by presenting or publishing only a broader one, such as a marketplace listing. A required issuer configured for software whose presentations are review-only ({{review-only}}) excludes that software from the tenant, and the server rejects a request for such software there.

## Multi-Tenant Issuers {#multi-tenant-issuers}

If an issuer reviews software for several customers whose decisions are each trusted independently, it MUST issue under a distinct `iss` value for each customer, so that each customer's decisions and `iat` floor ({{multi-instance}}) stay separate.

# Runtime Presentation {#cimd-presentation}

A client presents its statement in a request ({{runtime-presentation}}) or publishes it where its document points ({{pulled-statements}}). Either way, admission follows the steps of {{processing}}.

## Admission {#processing}

For a runtime presentation, the authorization server proceeds as follows, rejecting at the first failure ({{errors}}):

1. Obtain the statement, presented in the request ({{runtime-presentation}}) or pulled ({{pulled-statements}}).
2. Validate the statement ({{validation}}), completing every check that needs no client-controlled retrieval before any such retrieval ({{retrieval}}).
3. Verify the statement requirements of {{profiles}}, the `iat` floor of {{multi-instance}}, and the `client_id` rule of {{runtime-presentation}}.
4. Resolve the reviewed document and compare its digest ({{effective-metadata}}). A server that proceeds without the statement on a mismatch ends admission here.
5. Verify the sender constraint against that document ({{sender-constraint}}).
6. Evaluate the request against the reviewed metadata ({{effective-metadata}}).
7. Create the establishment ({{grant-lifecycle}}), unless the statement is review-only ({{review-only}}).

## Reviewed Metadata {#effective-metadata}

Having validated the statement, and before verifying the proof ({{processing}}), the authorization server MUST obtain the CIMD at the statement's `sub`, only by retrieval or from octets it has already digested for that identifier ({{retrieval}}), and compare its digest with the `cimd_digest` claim. If octets it already holds do not match, it retrieves the document once more before treating the mismatch as a change.

A match means the served document is the reviewed one, and its members are the client's metadata for the request. The server takes those members from the octets whose digest matched, not from another retrieval or a copy it has not digested. A mismatch means the document changed after review, and the authorization server MUST NOT treat the changed document as reviewed. It rejects the presentation or, by local policy, proceeds without the statement. If it proceeds, it applies its policy for CIMD clients it has not reviewed to the current document and creates no establishment for the statement.

The request is evaluated against the reviewed metadata as against any client's registered metadata ({{RFC6749}}, {{CIMD}}): any requested grant type, response type, or scope MUST fall within it. A document that omits `scope` sets no ceiling on scope beyond the server's own policy.

## Retrieval {#retrieval}

Every retrieval this specification causes, of the CIMD, a `jwks_uri` it names, or the statements its `software_statements_uri` names, inherits the server-side request forgery protections in Section 8.6 of {{CIMD}}. A trusted signature does not make a URL safe: the authorization server MUST apply its URL, redirect, address-range, transport, and content-type policy independently to every referenced location.

If a retrieval fails, the authorization server MUST reject the request and MUST NOT fall back to a weaker proof, except that a failed pull of statements, or a failed retrieval of a statement key set ({{issuer-trust}}) needed to verify a pulled statement, for a request opening a new grant does not reject the request, which proceeds as {{pulled-statements}} describes.

A retrieval does not complete only when it times out, its connection fails, or the response has a 429 (Too Many Requests) status code (Section 4 of {{RFC6585}}) or a 5xx status code. Any other failure to obtain an acceptable representation, such as another 4xx status code or a body that is not a JSON object, is final: a document is treated as failing validation, a key set as carrying no acceptable key, and a statements location as publishing no statement.

A trusting authorization server MAY reuse document octets it holds for digest comparison ({{effective-metadata}}), a key set retrieved from a document's `jwks_uri`, and a statement key set from either source ({{issuer-trust}}), only within ordinary HTTP caching bounds and a maximum reuse period of its own choosing. It MUST NOT let the publisher's cache directives alone (the client's for the document and its key set, the issuer's for a statement key set) determine how long held octets continue to match a statement or a removed key continues to verify ({{statement-validation}}). Without this rule, a key the document carries inline would continue to verify after the publisher removes it.

## Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for presentation. The software statement ({{profiles}}). It is consumed as a runtime presentation, a refresh replacement ({{refresh}}), or, if {{REGISTRATION}} is implemented, a delivery under that specification. A request carrying the parameter in any other role is rejected.

A request making a runtime presentation, or carrying a refresh replacement ({{refresh}}), MUST include the `client_id` parameter, even if the client authenticates, and its value, the client's CIMD URL, MUST exactly equal the statement's `sub` claim. If the `client_id` parameter is absent, the authorization server rejects the request. If the two differ in a runtime presentation, the authorization server MUST reject the presentation. A refresh replacement whose `sub` differs is rejected as {{refresh}} provides. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

A request that asks an authorization server to issue a software statement, rather than to consume one, MUST NOT carry the `software_statement` parameter.

An authorization server that does not implement this specification ignores the parameter (Sections 3.1 and 3.2 of {{RFC6749}}).

### Token Endpoint {#token-requests}

At the token endpoint, a runtime presentation is valid on a request using any one of the following grants:

* The `client_credentials` grant.
* The JWT or SAML 2.0 assertion grant (Section 2.1 of {{RFC7523}} and Section 2.1 of {{RFC7522}}).
* Another grant whose specification provides for runtime presentation under this specification.

Authorization-code redemption and refresh-token use continue an existing grant ({{grant-lifecycle}}, {{refresh}}).

### Authorization Endpoint {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}. The subsequent authorization request MUST use a `client_id` parameter exactly equal to the establishment's `sub` and MUST NOT include the `software_statement` parameter. A pulled statement needs no pushed authorization request ({{pulled-statements}}).

## Pulled Statements {#pulled-statements}

This specification defines the following client metadata member:

`software_statements_uri`:
: OPTIONAL. URL, in a CIMD, at which the client publishes software statements about itself. The member has no meaning in a registration request. The URL MUST use the `https` scheme. A request to it returns a JSON object whose `software_statements` member is an array of statements ({{profiles}}), each a string in JWS compact serialization. The metadata digest covers this member ({{metadata-digest}}).

An authorization server that supports pulled statements MAY retrieve this URL when it resolves the document of a client that presented no statement. In retrieving this URL, the server:

* MUST send a `GET` request with `Accept: application/json`.
* MUST NOT automatically follow redirects.
* MUST accept a new representation only from a `200` response with a JSON media type, and can instead reuse a stored representation that a `304 Not Modified` response revalidates (Section 15.4.5 of {{RFC9110}}).

Both the document and this URL are retrieved before any statement exists to validate, so the ordering of {{processing}} applies from the first statement verified onward.

The server considers only statements that meet all of the following conditions:

* The `sub` claim equals the document's client identifier.
* The statement validates under {{validation}} and permits presentation ({{profiles}}).
* The `cimd_digest` claim equals the digest of the document the server resolved.
* The `iat` floor of {{multi-instance}} does not supersede it.

The server ignores the rest.

If the server's trust configuration requires an issuer for this client ({{issuer-trust}}), it applies only that issuer's statement, and if none remains, the client is not reviewed. Otherwise, if more than one statement remains, the server keeps only the statement with the latest `iat` from each issuer and applies any one of those. A pulled statement the server does not apply does not advance the `iat` floor.

If retrieving the statements does not complete or no statement remains, the server applies its policy for CIMD clients it has not reviewed, subject to any required issuer ({{issuer-trust}}). If that policy or a required issuer requires reviewed software, the server rejects the request. A retrieval failure is never a withdrawal.

Having selected a statement, the server continues from step 5 of {{processing}}, with the document already resolved. The selection above satisfies the `client_id` rule of step 3.

The presenter of a pulled statement is bound as for a presented statement ({{sender-constraint}}), at the point the following table shows, unless the statement is review-only ({{review-only}}). Public and confidential clients are as {{sender-constraint}} defines them:

| Where the statement is pulled | Confidential client | Public client |
| --- | --- | --- |
| Token endpoint, opening a new grant | The request's client authentication binds the presenter | Review-only |
| Pushed authorization request endpoint | The client's authentication binds the presenter | {{public-client-presentation}} binds the presenter if the request uses PKCE with `S256` and binds a key through a DPoP proof or `dpop_jkt`. Otherwise review-only |
| Authorization endpoint | The client's authentication at code redemption binds the presenter, before any token is issued | {{public-client-presentation}} binds the presenter if the request uses PKCE with `S256` and binds a key through `dpop_jkt`, which redemption then proves. Otherwise review-only |

At code redemption, a confidential client whose statement was pulled at the authorization endpoint authenticates with a key the reviewed document carries, in its `jwks` or at its `jwks_uri` within the reuse bounds of {{retrieval}}, not one that only a later retrieval of the document carries, and that key becomes the establishment's proven key. Code redemption and refresh otherwise continue an establishment and keep the binding it already has ({{grant-lifecycle}}, {{refresh}}).

An authorization server advertises support through `software_statement_pull_supported` ({{authorization-server-metadata}}).

## Sender Constraint {#sender-constraint}

A runtime presentation that is not review-only ({{review-only}}) MUST be sender-constrained under one of two models, set by the `token_endpoint_auth_method` its reviewed document declares:

* Confidential client (a method other than `none`): by a key the statement attests ({{confidential-client-presentation}}).
* Public client (`none`): by an installation key the presenter binds ({{public-client-presentation}}), while its reviewed redirection URIs, with PKCE `S256`, bind the presentation itself.

The authorization server MUST reject a presentation under a reviewed document that omits the `token_endpoint_auth_method` member, even if the presentation is review-only. A statement pulled for such a document is instead review-only ({{review-only}}).

For a presentation that is not review-only, under either model:

* The presenter proves possession of the key its model requires through the applicable client authentication method or a DPoP proof {{RFC9449}} or, for a public client that binds its key through `dpop_jkt`, at code redemption ({{grant-lifecycle}}). If a request carries both, the key its client authentication proves is the proven key.
* The proof MUST be bound to the current request and validated with the replay protections of that mechanism.
* If the reviewed document declares a client authentication method in its `token_endpoint_auth_method` member ({{CIMD}}), the presenter MUST use that method.
* The authorization server MUST reject a presentation without such a proof or, for a confidential client, whose proven key the reviewed document does not carry.

### Confidential Clients {#confidential-client-presentation}

For a confidential client's presentation that is not review-only, the proven key MUST appear in the `jwks` or at the `jwks_uri` of the reviewed document ({{effective-metadata}}). The client authentication method the document declares therefore has to prove possession of such a key, as `private_key_jwt` (Section 2.2 of {{RFC7523}}) and `self_signed_tls_client_auth` (Section 2.2 of {{RFC8705}}) do.

The key set at the document's `jwks_uri` is obtained when the presentation is processed, by retrieval or from a copy reused within the bounds of {{retrieval}}. If a retrieval it needs does not complete, the presentation is rejected.

### Public Clients {#public-client-presentation}

Software distributed to end users cannot hold a key its reviewed document carries: a key shipped in every copy identifies the software, not the installation. Such a document declares `token_endpoint_auth_method` of `none` and carries no key material.

An authorization server MUST reject a presentation, even a review-only one, that opens a new grant at the token endpoint ({{token-requests}}) from a public client. Such a presentation opens no redirect and has nothing to bind it. A statement pulled for such a client at the token endpoint for a request opening a new grant is instead review-only ({{review-only}}).

For a public client's presentation that is not review-only, at the pushed authorization request endpoint, the authorization server:

* MUST require PKCE {{RFC7636}} with the `S256` method.
* MUST require the presenter to bind a key it holds, through the `dpop_jkt` parameter or, at the pushed authorization request endpoint, a DPoP proof (Section 10.1 of {{RFC9449}}), and records that key as the proven key of the establishment ({{grant-lifecycle}}).
* MUST NOT require that key to appear in the reviewed document.

A statement pulled for such a client at the pushed authorization request endpoint or the authorization endpoint ({{pulled-statements}}) is bound the same way, under the same requirements, if the request uses PKCE with the `S256` method and binds a key as those requirements provide.

The reviewed redirection URIs, not the proven key, bind such a presentation ({{public-client-security}}). The proven key identifies the installation the grant belongs to.

### Review-Only Presentations {#review-only}

The following are review-only:

1. Every presentation and pulled statement for a reviewed document that carries any redirection URI another application on the same device could claim, whatever authentication method the document declares. Such a URI is a non-`https` URI, such as a private-use scheme or an `http` loopback redirection URI (Sections 7.1 and 7.3 of {{RFC8252}}), or an `https` URI whose host is a loopback address or `localhost`. The application that claims such a URI receives the code, so neither the reviewed redirection URIs nor a key shipped in every copy of the software can bind the presenter.
2. A statement pulled for a document that omits `token_endpoint_auth_method`, wherever it is pulled. A presentation under such a document is rejected instead ({{sender-constraint}}).
3. A statement pulled for a public client at the token endpoint for a request opening a new grant.
4. A statement pulled for a public client at the pushed authorization request endpoint or the authorization endpoint without PKCE with the `S256` method and a key bound as {{public-client-presentation}} provides.

A review-only presentation or pulled statement creates no establishment, admits nothing, and does not advance the `iat` floor of {{multi-instance}}. The authorization server proceeds as it would for the same CIMD client presenting no statement ({{ui-audit}}).

The statement of a review-only presentation, or a review-only pulled statement, MUST NOT satisfy a policy requiring reviewed software. If the server's policy requires reviewed software, it rejects a review-only client.

## Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment: the state against which the subsequent authorization request, code redemption and refresh under the grant it opens are checked. It comprises the following:

* The validated `sub`.
* The statement identity (its `iss` and `jti`), its `iat` and effective expiry, and any claim by which a withdrawal mechanism locates it, such as `status`.
* The authorization server's own tenant the grant was opened for, if it hosts more than one.
* The reviewed metadata and the digest it matched ({{effective-metadata}}).
* The issuer trust decision.
* The sender-constraint mechanism and proven key, which for a statement pulled at the authorization endpoint for a confidential client is set at code redemption ({{pulled-statements}}).

The authorization server MUST bind the resulting `request_uri`, authorization code, refresh token, and other grant continuation state to the establishment, as applicable. An establishment ends once no continuation state bound to it remains usable.

A replacement accepted on refresh MUST apply only to the grant refreshed ({{refresh}}). Any other grant, `request_uri`, or authorization code bound to the same establishment keeps the state its request was evaluated against.

A token request that redeems an authorization code opened by a presentation:

* MUST have a `client_id` parameter exactly equal to the establishment's `sub`, even if the client authenticates.
* MUST demonstrate possession of the establishment's proven key under the same sender-constraint mechanism or, for a confidential client whose statement was pulled at the authorization endpoint ({{pulled-statements}}), authenticate with the key that becomes the proven key.
* MUST NOT carry the `software_statement` parameter.

A redemption that carries a statement, has a wrong client identifier, or fails the key binding is rejected. The prohibition on carrying a statement covers redemption of a code bound to an establishment. A registered client redeeming its own code can deliver a replacement statement under {{REGISTRATION}}, which that specification calls a delivery and which is not a presentation.

Expiry after presentation does not by itself invalidate an establishment already bound ({{refresh}}).

### Refresh {#refresh}

The authorization server MAY, by local policy, require a current statement on refresh-token use, and SHOULD require one once the establishment's recorded statement has expired ({{enforcement-bounds}}). The recorded statement satisfies that requirement while it is current, even after a later statement has raised the `iat` floor of {{multi-instance}}. A server that uses a withdrawal mechanism for the recorded statement's issuer SHOULD consult it for that statement at each refresh.

On refresh-token use, the authorization server proceeds as follows. A refresh rejected at any step leaves the grant's state unchanged.

1. Check the proven key. The authorization server:

   * MUST verify possession of the establishment's proven key under the same sender-constraint mechanism.
   * MUST, except for an establishment created under {{public-client-presentation}}, verify the document currently served at the establishment's `sub` still carries that key, in its `jwks` or at its `jwks_uri`, within the reuse bounds of {{retrieval}}.

   If either check fails, including when the document does not carry the key, cannot be obtained, or exceeds the size bound of {{metadata-digest}}, the server rejects the refresh. It completes these checks before considering any replacement.

2. Decide whether a replacement is needed. One is needed in either of the following cases, and otherwise none is, though the client can still offer one:

   * A refusal record covers the recorded statement.
   * The recorded statement has expired, and the server requires a current statement, by local policy or because a required issuer ({{issuer-trust}}) covers the establishment's `sub` in the establishment's tenant.

3. Obtain a replacement. A client can offer one on any refresh in the `software_statement` parameter. A statement with the recorded `iss` and `jti` is not a replacement and is treated as the recorded statement. When a replacement is needed and the request carries none, a server that supports pulled statements pulls, however the establishment was created, and takes as the replacement the newest statement with the recorded `iss` and `sub` among those {{pulled-statements}} considers. A refresh that needs a replacement and obtains none is rejected: a reviewed grant never continues under the policy for clients the server has not reviewed.

4. Validate the replacement. The following requirements apply to any replacement, and a refresh whose replacement fails them is rejected even when none was needed:

   * It MUST validate under {{validation}}, including its audience if it carries one.
   * Its `iss` and `sub` MUST equal the recorded statement's.
   * Its `iat` MUST be later than the recorded statement's.
   * Its `cimd_digest` MUST name the document currently served at that `sub`, which the server confirms by retrieval or by revalidating octets it holds.
   * It MUST authorize the establishment's proven key ({{sender-constraint}}) or, for an establishment created under {{public-client-presentation}}, name a document that still satisfies that section.

   A refresh whose replacement the `iat` floor of {{multi-instance}} supersedes is also rejected. A replacement that meets these requirements and that the floor does not supersede is accepted.

5. Enforce currency. If the authorization server holds a refusal record for the recorded statement, it MUST reject a refresh without an accepted replacement, whatever its policy on requiring a current statement otherwise. If a required issuer ({{issuer-trust}}) covers the establishment's `sub` in the establishment's tenant, the server MUST reject a refresh unless the recorded statement or an accepted replacement is a current statement from that issuer. A grant recorded under another issuer's statement cannot meet this, so its client opens a new grant by presentation.

6. Bound access and update. The refreshed access MUST fall within the metadata of the document an accepted replacement names or, without one, the establishment's reviewed metadata. If a replacement is accepted, every item of the refreshed grant's establishment ({{grant-lifecycle}}) other than the `sub`, the tenant, and the sender-constraint mechanism and proven key becomes the replacement's in one atomic update, and concurrent replacements resolve to the most recently issued statement. Without one, the grant's state is unchanged.

Refresh never rotates the establishment's key. A client that needs a new key performs a new presentation.

# Relationship to Other Work {#relationship}

This section is non-normative.

## Client Attestation {#relationship-attestation}

A software statement and a client attestation {{ABCA}} are both signed third-party assertions. A statement attests client software and the metadata its issuer evaluated. A client attestation attests a running instance and the key it holds, and is presented with a proof bound to the request.

Neither assertion grants access nor substitutes for the other. Runtime presentation therefore always requires a sender constraint ({{sender-constraint}}). Binding a presenter attestation defined elsewhere is an extension ({{extensions}}).

## Identity Assertions {#identity-assertions}

A statement records that a named party reviewed software, not that a particular customer currently permits that software to act in its tenant. A tenant-scoped decision (`aud_tenant`) authorizes no particular user or transaction. When a customer's identity provider mediates a grant, as with a cross-domain identity assertion {{IDJAG}}, it issues assertions per grant only for clients the customer permits. A statement serves when the issuer is not in the request path, or when a customer wants a portable record of what its review covered.

## OpenID Federation {#relationship-federation}

A trust mark {{OPENID-FED}} is also a signed third-party assertion about an entity, but it is resolved through a federation in which both parties enroll. A statement needs no federation: a trusting authorization server configures its issuer directly ({{issuer-trust}}), and the statement binds the exact octets of the client's own document rather than metadata the federation resolves.

# Error Responses {#errors}

A rejection uses the error response and HTTP status codes of its endpoint: Section 5.2 of {{RFC6749}} at the token endpoint, Section 2.3 of {{RFC9126}} at the pushed authorization request endpoint, and Section 4.1.2.1 of {{RFC6749}} at the authorization endpoint.

At the token and pushed authorization request endpoints, an authorization server SHOULD respond with HTTP status code 503 to a request it rejects with the `temporarily_unavailable` error code. {{REGISTRATION}} defines rejections at a registration endpoint and how registration validity is reported.

This specification assigns `temporarily_unavailable` only to a retrieval that did not complete ({{retrieval}}), never to a response over the size bound of {{metadata-digest}}. A request carrying the `software_statement` parameter in a role {{runtime-presentation}} does not permit is rejected with `invalid_request`.

## Requiring a Current Statement {#statement-required}

This specification defines the following error code:

`statement_required`:
: A current software statement is required and none was supplied or pulled, or the one supplied or pulled is expired, refused by a refusal record ({{withdrawal}}), or superseded under the `iat` floor of {{multi-instance}}. The code also applies when a statement offered as a replacement on refresh ({{refresh}}), or at the token or pushed authorization request endpoint under {{REGISTRATION}}, does not satisfy the requirements for one. The client recovers by obtaining a statement issued more recently.

## Presentation {#presentation-errors}

A presentation rejected at the token or pushed authorization request endpoint for a reason this specification defines uses `temporarily_unavailable` if a retrieval it needs, including a pull of statements ({{pulled-statements}}), did not complete, and otherwise the following error codes:

`invalid_client`:
: Any of the following:

  * The statement is malformed or fails validation ({{validation}}) for a reason other than expiry, a refusal record, or the `iat` floor.
  * The request's `client_id` differs from the statement's `sub` ({{runtime-presentation}}), or the request fails the sender constraint ({{sender-constraint}}), including a presentation under a document that omits `token_endpoint_auth_method` and one opening a new grant at the token endpoint for a public client ({{public-client-presentation}}).
  * The reviewed document cannot be obtained, including over the size bound, fails validation as {{CIMD}} requires, contains duplicate member names ({{validation}}), or does not match and the server does not proceed without the statement ({{effective-metadata}}).
  * The bound on establishments is reached ({{establishment-bounds}}).

`statement_required`:
: The statement is expired, refused by a refusal record, or superseded under the `iat` floor ({{statement-required}}), or a required statement is absent, including one from an issuer required for the tenant ({{issuer-trust}}).

`unauthorized_client`:
: The request exceeds the reviewed document's grant types or response types ({{effective-metadata}}), the document carries metadata this server's policy refuses, or the client is review-only where policy or a required issuer requires reviewed software ({{review-only}}), or the request is for software a required issuer excludes from the tenant ({{issuer-trust}}), in place of `statement_required`.

`invalid_scope`:
: The request exceeds the reviewed document's scope ({{effective-metadata}}), or the document carries a scope this server's policy refuses.

`invalid_request`:
: The request omits `client_id` ({{runtime-presentation}}).

A request rejected for another reason, such as an assertion grant whose assertion is invalid (Section 3.1 of {{RFC7523}}), uses the error code its grant or endpoint assigns.

## Code Redemption and Refresh {#continuation-errors}

Under an establishment, code redemption ({{grant-lifecycle}}) is rejected with `invalid_request` if it carries a statement, `invalid_grant` for a wrong client identifier or failed key binding, and `invalid_client` at the bound on establishments ({{establishment-bounds}}).

Under an establishment, a refresh-token request rejected for a reason this specification defines uses `temporarily_unavailable` if a retrieval it needs did not complete, including under a required issuer, and otherwise the following error codes:

`invalid_grant`:
: A check of step 1 of {{refresh}} fails.

`statement_required`:
: A replacement fails step 4 of {{refresh}} or the `iat` floor, a needed replacement is not obtained in step 3, including when a completed pull yields none, or the refresh fails step 5. These cases use `statement_required` in place of `invalid_grant`.

`unauthorized_client` or `invalid_scope`:
: The refreshed access exceeds the reviewed metadata or this server's policy, as for a presentation.

`invalid_request`:
: The request carries a replacement but omits `client_id` ({{runtime-presentation}}).

None of these rejections indicates refresh-token replay ({{RFC9700}}), whose detection this specification does not change. A refresh-token request rejected for another reason, such as an expired or revoked refresh token, uses the error codes of Section 5.2 of {{RFC6749}}.

## Authorization Endpoint {#authorization-errors}

At the authorization endpoint, if policy or a required issuer requires reviewed software ({{review-only}}, {{pulled-statements}}, {{issuer-trust}}), a review-only client, or software a required issuer excludes from the tenant, is rejected with `unauthorized_client`, and a client whose pull of statements, or retrieval of a statement key set needed to verify one, did not complete ({{retrieval}}) is rejected with `temporarily_unavailable`.

Apart from those two cases, at the authorization endpoint, if the authorization server requires a statement for a client it has not admitted on one, it MUST return the `statement_required` error code in the authorization error response (Section 4.1.2.1 of {{RFC6749}}).

The authorization server redirects with the error only after resolving the client's CIMD and validating the request's `redirect_uri` against it. If the authorization server cannot resolve the document, it MUST NOT redirect, and it reports the error to the resource owner instead.

An authorization request carrying the `software_statement` parameter ({{authorization-requests}}) is rejected with `invalid_request` in place of any other code of this section. The authorization server redirects with that error only after validating the request's `redirect_uri`: against the CIMD as the preceding paragraph requires or, for a registered client, against its registered redirection URIs (Section 4.1.2.1 of {{RFC6749}}).

## Proof Mechanism Errors {#proof-errors}

If the proof mechanism defines an error of its own for a proof it does not accept, such as `invalid_dpop_proof` for a DPoP proof that fails the checks of Section 4.3 of {{RFC9449}} or `use_dpop_nonce` for a nonce challenge (Section 8 of {{RFC9449}}), the request is rejected with that error in place of the code this specification assigns to a failed sender constraint.

A proof that its mechanism accepts but that proves a key other than the one this specification requires, such as a key the reviewed document does not carry or one other than the establishment's proven key, is not an invalid proof but a failed sender constraint.

## Error Descriptions {#error-descriptions}

The authorization server can return non-sensitive diagnostics in the `error_description` parameter to a client it has authenticated, but MUST NOT reveal details of its issuer trust configuration ({{issuer-trust}}), such as the client identifier namespaces an issuer can attest, to an unauthenticated requester.

If a statement is refused by a refusal record rather than merely expired, and the client is authenticated, the authorization server ought to indicate that a statement issued more recently is required.

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

The authorization server validates the statement, retrieves the document its digest names, verifies the client assertion against a key from that document's `jwks_uri`, and checks that the requested `scope` falls within the document's `scope`. It keeps no registration.

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

An authorization server receiving an ordinary authorization request with this `client_id` resolves the document, retrieves the statements, and keeps the one from a trusted issuer whose `cimd_digest` matches the document it retrieved. It completes the binding when the client authenticates with `private_key_jwt` at code redemption ({{pulled-statements}}).

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

The statement names one audience and permits only runtime presentation, as omitting `statement_uses` would. It carries no client metadata: the document its `cimd_digest` names supplies it.

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} values:

`software_statement_jwks_uri`:
: REQUIRED for an authorization server that issues statements under this specification. URL of a JWK Set containing only the keys with which it signs them. The URL MUST use the `https` scheme, and a trusting authorization server MUST validate the server's certificate when retrieving it.

  The keys in this set MUST NOT appear in the JWK Set at the server's `jwks_uri` metadata value, and the server MUST NOT sign anything other than statements under this specification with them. As a result, a key that signs another artifact, such as a Status List Token or a software statement of another profile, cannot verify a statement ({{legacy-servers}}).

`software_statement_signing_alg_values_supported`:
: REQUIRED for an authorization server that issues statements under this specification. A JSON array containing the asymmetric JWS `alg` values that the authorization server can use to sign them. The array MUST NOT contain `none` or a symmetric algorithm. An authorization server that only accepts software statements does not publish it.

`software_statement_pull_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server retrieves statements from a client's `software_statements_uri` member when admitting a client that presented none ({{pulled-statements}}). If omitted, the default value is `false`.

# Extension Points {#extensions}

The following extensions are left to separate specifications:

* Attested keys: a client attestation {{ABCA}}, including one from an attester the reviewed document names, vouching for a key that document does not carry ({{multi-instance}}).
* Statement conveyance within a client attestation, rather than as a request parameter.
* Runtime presentation on grants other than those {{token-requests}} names, such as token exchange.
* Partial review, by which an issuer vouches for particular members rather than a whole document ({{profiles}}).
* Withdrawal mechanisms ({{withdrawal}}), such as Token Status List {{STATUSLIST}}.

# Implementation Considerations {#implementation-considerations}

This section gives guidance to implementers. It adds no requirement.

## Caching and Reuse of Retrieved Content {#caching}

A server can revalidate octets it holds with a conditional request rather than retrieving them again ({{metadata-digest}}).

A failed retrieval ({{retrieval}}) is followed by back-off rather than caching, which {{CIMD}} forbids. A response from a `software_statements_uri` ought to be cached no longer than the document is, and can be revalidated with a conditional request, as the document can ({{pulled-statements}}).

## Retrieval and Processing Bounds {#processing-bounds}

An authorization server can bound the retrieval and processing work of {{external-retrieval}} as follows:

* Rate-limit presentations per statement identity, per subject, and per source, and pulls ({{pulled-statements}}) per client identifier and per source, before spending retrieval or storage on a new presentation.
* Refuse a presentation that would create a new establishment, before spending retrieval or storage on it, once a bound on establishments ({{establishment-bounds}}) is reached.
* Bound JWT size and parsing work, concurrent retrievals, response size, and response time.
* Bound the size of a `software_statements_uri` response and the number of statements examined separately from the document's. Several issuers' statements outgrow a document-sized bound, and a host can pad the array.
* Discard pulled statements from unconfigured issuers before verifying any signature.

## Bounding Establishments {#establishment-bounds}

A trusting authorization server can bound the number of establishments, and any registrations ({{REGISTRATION}}), derived from one issuer's statements about one piece of software, identified by `iss` and `sub` claims. How it counts toward a bound is local policy. Two choices keep a bound effective:

* State that anyone holding a copy of a statement can create does not count: a pushed authorization request and its unredeemed code, which anyone holding a copy of a public client's statement can approve with their own account, or a registration under which no access token has been issued. Otherwise a holder of a copy could exhaust the bound for the software's other instances.
* A count persists across replacements: a replacement accepted on refresh does not add to it.

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

An issuer ought to name an audience and, for a statement whose `statement_uses` claim permits registration, to omit the `aud` claim only if it accepts that any server trusting the issuer can register the software on the strength of a copy ({{REGISTRATION}}).

An issuer ought to omit the `aud_tenant` claim from a statement whose review applies wherever its `aud` reaches, such as a marketplace listing. The claim is also unnecessary if the authorization server has one tenant, or if the server's trust configuration confines the issuer to a tenant, as it can for an issuer that does not know the tenant's identifier.

## Statement Lifetime and Renewal {#lifetime-practice}

Short lifetimes tighten an issuer's control and increase renewal traffic. At a server that registers clients under {{REGISTRATION}}, the lifetime also sets the registration's renewal cadence. A fleet of statements issued together expires together, so issuers ought to stagger expiries or renew ahead of the boundary.

A changed document is always served before a statement over it exists. Until the new statement exists, a presentation under the earlier one fails the digest comparison ({{effective-metadata}}), and at registration a mismatch is fatal ({{REGISTRATION}}). The statement's lifetime bounds how stale an unobserved review can become.

Instances need the most recently issued statement from a shared source rather than a copy fixed at build or install time ({{multi-instance}}).

## Withdrawal Practice {#withdrawal-practice}

What a server does when it cannot consult a withdrawal mechanism it relies on is local policy ({{withdrawal}}). Refusing makes issuer availability a precondition for every request the statement governs. Proceeding leaves a withdrawn statement acceptable until it expires.

A refusal that the server's operator entered, rather than one derived from a withdrawal mechanism, persists on the operator's own terms.

Withdrawal does not replace lifetime: an issuer chooses `exp` assuming no trusting authorization server uses a withdrawal mechanism.

# Security Considerations {#security-considerations}

The security considerations of {{CIMD}} and Section 5 of {{RFC7591}} apply.

## Assurance Provided by a Statement {#what-statement-attests}

A statement binds an issuer's review to the octets of a CIMD, which describes an OAuth client configuration. It does not bind the software's code, dependencies, or behavior. Software can change while its document stays the same, and the statement remains acceptable until it expires or is withdrawn ({{withdrawal}}). Assurance about the software itself rests on the issuer's review process. Any change to the document's octets, including one that changes no metadata value, makes statements over the earlier octets fail the digest comparison ({{effective-metadata}}). The digest covers a URL the document carries, such as `logo_uri`, but not the content served at it, which can change after review.

The following table states what admission under this specification does and does not establish:

| Property | Established |
| --- | --- |
| A configured issuer signed the statement for this client identifier and, if the statement names an audience, one that includes this server | Yes |
| The served document is the one the issuer reviewed, byte for byte | Yes |
| The presenter controls a key the reviewed document carries | Yes, except under {{public-client-presentation}}. For a statement pulled at the authorization endpoint, possession is proved at code redemption, before any token is issued ({{pulled-statements}}). Inline `jwks` attests the keys themselves; `jwks_uri` attests only their location ({{statement-validation}}) |
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

Runtime presentation can cause the authorization server to retrieve the CIMD, a `jwks_uri` it names, or the statements its `software_statements_uri` names ({{retrieval}}).

These retrievals happen before any client is registered or any user has interacted, so an unauthenticated requester can cause them with one acceptable statement or, at a server that pulls statements, with none. The bounds of {{processing-bounds}} limit this work but do not prevent it.

## Expiry, Withdrawal, and Open Grants {#enforcement-bounds}

Expiry and withdrawal ({{withdrawal}}) govern new admissions. Neither revokes an issued access token. A grant opened by a statement lasts for the life of its refresh tokens, whatever the statement's lifetime, unless the server requires a current statement on refresh, holds a refusal record, or finds the proven key gone from the client's document ({{refresh}}). Removing an issuer from trust configuration, or narrowing what it can attest, does not by itself end a grant or a registration ({{REGISTRATION}}) already admitted on one of its statements. A server ends them by its own means, such as entering refusal records, revoking refresh tokens, or deleting registrations.

## Observable State {#oracle-considerations}

A `statement_required` refusal at the authorization endpoint ({{authorization-errors}}) travels through the user agent, so anyone who can see the redirection learns that this client has no established standing at this server. The disclosure is accepted because a client that cannot learn why it was refused cannot act. The disclosure carries no detail of the server's issuer trust configuration, which {{error-descriptions}} keeps from unauthenticated requesters.

# Privacy Considerations

The privacy considerations of {{CIMD}} and Section 6 of {{RFC7591}} apply.

A presentation or delivery reveals to the authorization server the client's issuer relationship and, if the statement carries an `aud` claim, the other authorization servers the client intends to use. Omitting that claim discloses nothing beyond the review. Pushed authorization requests ({{authorization-requests}}) keep statements out of browser history, referrers, and front-channel logs.

A central issuer also learns from renewals which of its statements are in active use, so issuance and renewal logs deserve the same care as the statements ({{ui-audit}}).

Retrieving the document tells its host that the server is resolving the client. Pulling statements ({{pulled-statements}}) tells the host of `software_statements_uri` the same. That host is an additional party only if the URL is on another origin. Published statements are readable by anyone, so their `aud` and `aud_tenant` claims disclose which servers and tenants a review names. A statement whose audience is sensitive is presented rather than published.

# IANA Considerations {#iana}

## OAuth Parameters Registry

This specification requests registration of the `software_statement` parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}, for runtime presentation and statement delivery of the artifact defined by {{RFC7591}}:

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

## Claims Defined Elsewhere

This specification uses two claims it does not define: `aud_tenant`, defined and registered by {{IDJAG}}, and `status`, defined and registered by {{STATUSLIST}}. This section requests no IANA action.

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

This specification also requests that IANA update the existing `temporarily_unavailable` entry, adding the token and pushed authorization request error responses to its usage locations and this specification as a reference:

Error Name:
: `temporarily_unavailable`

Error Usage Location:
: token error response and pushed authorization request error response, in addition to the locations already registered

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
: `software_statement_signing_alg_values_supported`

Metadata Description:
: JSON array containing the asymmetric JWS algorithms supported by the authorization server for signing software statements.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_jwks_uri`

Metadata Description:
: URL of the JWK Set containing only the keys with which the authorization server signs software statements.

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

# Open Design Questions {#open-questions}

This appendix is to be removed before publication. It records simpler designs the author would like the working group to weigh against this one. None changes what the statement attests: a configured issuer's review of a document's exact octets, which the client's authentication key or reviewed redirection URIs bind.

1. Issuer-hosted statements. The authorization server could obtain statements from the issuer, per client or as a signed list it synchronizes, instead of from the client. That would remove the `software_statement` parameter, `software_statements_uri`, the `iat` floor, and refusal records, since the server would always see the issuer's current statement. It would require issuers to serve statements with high availability, would tell an issuer which servers resolve which clients unless lists are synchronized, and departs from the client-carried statement of {{RFC7591}}.

2. Admission on every token request. The server could re-check a current statement and the served document on every token request instead of recording an establishment. That would remove establishments, refresh replacement, and the pinned proven key. Every refresh would then need a current statement and a retrieval of the document, and a grant would end when renewal lapses.

3. The public client's installation key. The server could rely on PKCE and on the sender-constraint and rotation practices of {{RFC9449}} and {{RFC9700}} instead of requiring a bound key, since the reviewed redirection URIs, not that key, bind a public client's presentation ({{public-client-presentation}}).

4. Scope. Requiring a pushed authorization request for every reviewed redirect flow would remove binding at code redemption ({{pulled-statements}}). Admitting only confidential clients in this specification, with public clients in a separate profile, would remove a second binding model.

# Acknowledgments
{:numbered="false"}

This mechanism began as the consumption part of a draft on software statement issuance and was split out so that consumption can evolve separately from issuance. It draws on working-group discussion of CIMDs and attestation-based client authentication.
