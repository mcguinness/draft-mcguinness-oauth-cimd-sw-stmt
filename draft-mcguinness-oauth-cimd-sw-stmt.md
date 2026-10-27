---
title: "CIMD Software Statement"
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
  RFC8252:
  STATUSLIST: I-D.ietf-oauth-status-list
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "CIMD Software Statement Registration"
    author:
      - fullname: Karl McGuinness
  ABCA: I-D.ietf-oauth-attestation-based-client-auth
  OPENID-FED:
    target: https://openid.net/specs/openid-federation-1_0.html
    title: "OpenID Federation 1.0"

--- abstract

A Client ID Metadata Document carries a client's own claims, which can change at any time. This specification defines a software statement in which a trusted party records its review of the exact bytes of such a document, and how an authorization server enforces that review when it admits the client at request time, without a registration. The client presents the statement in a token or pushed authorization request, or publishes it where the reviewed document points. Possession of the statement alone does not complete a grant: the presenter is also bound to the reviewed document. Because the document remains the metadata, one review is portable across the authorization servers that trust its issuer. A companion specification covers RFC 7591 registration.

--- middle

# Introduction

A Client ID Metadata Document {{CIMD}} identifies a client by a URL and supplies its metadata from a document the client controls. That metadata is the client's own claim and can change at any time. An organization that reviews client software has no interoperable way to record its review of a particular version of the document, or to have an authorization server it has never dealt with enforce that review.

This specification profiles the software statement (Section 2.3 of {{RFC7591}}) for that purpose. A statement names the client by its document URL and binds the review to the document's exact octets by a digest ({{metadata-digest}}). A statement carried inside the document, as Section 4.3 of {{CIMD}} permits, cannot bind the review to the octets of the document that carries it ({{profiles}}).

An authorization server that trusts the statement's issuer ({{issuer-trust}}) admits the client at request time on the reviewed document, and no other, without creating a registration ({{cimd-presentation}}). The document remains the metadata and the statement records the decision, so one review is portable across every authorization server in the statement's audience.

This specification serves deployments in which a party other than the client decides which software can connect, such as a marketplace or registry that vouches for software to authorization servers it does not operate, or an enterprise that approves software for its users at a service run by someone else. It also serves server-hosted software, such as an agent, that obtains tokens with the `client_credentials` grant. An admitted client is treated as reviewed, rather than under the authorization server's policy for Client ID Metadata Document clients it has not reviewed. A native application whose redirection URIs another application on the device could claim, such as a loopback or private-use URI, gains only a record of the review: its presentations are review-only ({{public-client-presentation}}) until an extension binds an installation key that a third party vouches for ({{extensions}}).

A statement vouches for metadata, not for its presenter. Possession of a statement alone admits nothing: the presenter is bound to the reviewed document ({{sender-constraint}}).

This specification does not define how a client obtains a statement. {{REGISTRATION}} defines consumption of the same statement in an {{RFC7591}} registration request. It builds on this specification, which does not depend on it. {{relationship}} relates this specification to other work.

## Protocol Overview

The following non-normative sequence summarizes runtime admission:

1. Using its Client ID Metadata Document URL as `client_id`, the client presents its statement in a token request or, for a redirect flow, a pushed authorization request, or publishes it at the `software_statements_uri` its document names ({{runtime-presentation}}, {{pulled-statements}}).
2. The authorization server validates the statement ({{validation}}), resolves the reviewed document ({{effective-metadata}}), and verifies the sender constraint against it ({{sender-constraint}}).
3. The request proceeds under that document's metadata, without a registration.
4. The state on which the rest of the grant depends persists as an establishment ({{grant-lifecycle}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology from {{RFC6749}} (OAuth), {{RFC7591}} (client metadata and software statements), and {{CIMD}} (Client ID Metadata Documents).

This specification defines the following terms.

Issuer:
: The authorization server that signs a software statement.

Trusting Authorization Server:
: An authorization server that consumes a software statement, at runtime under this specification or at registration under {{REGISTRATION}}.

Runtime Presentation:
: The consumption of a validated software statement presented in a token request or pushed authorization request, or pulled from where the client's document points ({{pulled-statements}}), without creating a client registration.

Establishment:
: The durable state a successful runtime presentation creates for the grant it opens, or that a replacement accepted on refresh gives that grant, enumerated in {{grant-lifecycle}}.

Proven Key:
: The key for which the presenter demonstrates possession during runtime presentation ({{sender-constraint}}).

Effective Expiry:
: The earlier of a statement's `exp` and its `iat` plus the maximum statement lifetime the trusting authorization server honors for its issuer ({{issuer-trust}}). A statement is expired once its effective expiry has passed.

Refusal Record:
: State an authorization server holds when it has learned that a statement ceased to be acceptable before its expiry, whether from a withdrawal mechanism it uses for the issuer ({{withdrawal}}) or from its own operator.

Current Statement:
: A statement whose effective expiry has not passed and that matches no refusal record. For a new admission or a replacement ({{refresh}}, {{REGISTRATION}}), a statement superseded under the `iat` floor ({{multi-instance}}) is also not current. On refresh, an establishment's recorded statement remains current after a later statement raises the `iat` floor ({{refresh}}).

Review-Only:
: Describes a presentation, or a pulled statement, that the authorization server evaluates but that admits nothing ({{public-client-presentation}}).

# The Software Statement {#profiles}

The software statement is a compact JWT {{RFC7519}} protected by JWS {{RFC7515}}. Although {{RFC7591}} permits a MAC, a statement issued under this specification MUST use an asymmetric digital signature, and the `none` algorithm and symmetric algorithms MUST NOT be used. Issuers and trusting authorization servers follow {{RFC8725}}.

The JOSE header MUST include the `kid` header parameter, identifying the signing key in the issuer's statement key set ({{authorization-server-metadata}}), and the `typ` header parameter with the value `software-statement+jwt` (Section 3.11 of {{RFC8725}}), which is the media type of {{media-type}} without its `application/` prefix. Extensions can add claims. An incompatible revision would use a new `typ` value.

A statement asserts that its issuer evaluated the Client ID Metadata Document whose content the digest identifies, and vouches for that document to any party that accepts the issuer. The JWT payload contains the following claims.

`iss`:
: REQUIRED. The issuer's issuer identifier, as defined by {{RFC8414}}.

`sub`:
: REQUIRED. The exact client identifier URL of the Client ID Metadata Document the issuer reviewed, not a `client_id` assigned through {{RFC7591}} registration.

`aud`:
: OPTIONAL. One or more authorization server issuer identifiers ({{RFC8414}}) restricting which authorization servers can accept the statement. If the claim is absent, any authorization server that trusts the issuer for the subject can accept it ({{issuer-trust}}), and any holder of a copy can use it wherever `statement_uses` permits registration, which requires no key. An issuer therefore ought to name an audience, and omit the claim only if it accepts that any server trusting the issuer may register the software on the strength of a copy ({{REGISTRATION}}).

`aud_tenant`:
: OPTIONAL. A tenant identifier at a trusting authorization server, as {{IDJAG}} defines the claim, naming the tenant in which the statement's decision applies. A statement whose decision is confined to one of a server's tenants MUST carry an `aud` claim naming exactly one authorization server and, unless that server confines the issuer to the tenant through its own trust configuration ({{issuer-trust}}), an `aud_tenant` claim naming the tenant. A tenant identifier is meaningful only at one authorization server.

  A statement whose review applies wherever its `aud` reaches, such as a marketplace listing, carries no `aud_tenant`. Its absence does not mean that the statement applies in every tenant: an issuer also omits it if the authorization server has one tenant, or if the server's trust configuration confines the issuer to a tenant, as it can for an issuer that does not know the tenant's identifier.

`statement_uses`:
: OPTIONAL. A JSON array identifying the permitted uses of the statement: `registration` ({{REGISTRATION}}) and `presentation` ({{cimd-presentation}}). If the claim is absent, only `presentation` is permitted. A delivery renewing a registration ({{REGISTRATION}}) is a use at registration. A refresh replacement ({{refresh}}) and a pulled statement ({{pulled-statements}}) are presentation.

`iat`:
: REQUIRED. A NumericDate value, which MUST be an integer, giving the time the statement was issued. An issuer MUST NOT issue two statements for given `iss` and `sub` claims with the same `iat` claim, and MUST ensure the value increases strictly across its signing nodes ({{refresh}}, {{multi-instance}}).

`exp`:
: REQUIRED. A NumericDate value giving the expiration time.

`jti`:
: REQUIRED. A unique identifier for the statement within the issuer's namespace.

`cimd_digest`:
: REQUIRED. The metadata digest ({{metadata-digest}}) of the Client ID Metadata Document the issuer evaluated, binding the statement to that document's exact content.

`status`:
: OPTIONAL. The `status` claim as {{STATUSLIST}} defines it, by which a statement locates itself in a Status List Token. Only a withdrawal mechanism that uses the claim processes it. This specification does not ({{withdrawal}}).

A statement issued under this specification MUST NOT contain a claim named in the IANA "OAuth Dynamic Client Registration Metadata" registry, and a trusting authorization server MUST NOT take client metadata from a statement's claims: the reviewed metadata is the document the digest names.

{{CIMD}} permits a Client ID Metadata Document to carry a `software_statement` member. A trusting authorization server MUST NOT consume a statement of this profile from that member, and MUST NOT treat that member as satisfying any requirement of this specification. A statement cannot describe the document that carries it: its digest would have to cover its own bytes. A statement found in that member names an earlier version of the document. Statements of other kinds in the member are out of scope for this specification.

An authorization server implementing this specification MUST reject a statement of this profile in an {{RFC7591}} registration request with the `unapproved_software_statement` error code (Section 3.2.2 of {{RFC7591}}), unless it consumes the statement as {{REGISTRATION}} specifies. The server's configured statement key set ({{issuer-trust}}) verifies the statement, so a generic registration path would otherwise attach the issuer's review to metadata the request supplies ({{legacy-servers}}).

## Metadata Digest {#metadata-digest}

The metadata digest is the unpadded base64url-encoded SHA-256 hash {{RFC6234}} of the retrieved representation body after removal of content coding. The body is not transcoded, normalized, or re-serialized. Any byte order mark or trailing newline is included.

Retrieval for digest purposes follows {{CIMD}} and MUST NOT use content negotiation: the retrieving party sends the `Accept` header field that {{CIMD}} specifies for retrieving the document, or `Accept: application/json` if it specifies none, and no other variant selection. Equal digests show that the issuer and the trusting authorization server obtained the same octets, so any condition that makes retrieval depend on who is asking defeats the comparison.

A trusting authorization server MUST treat a response exceeding its configured size bound (Section 8.7 of {{CIMD}}) as a retrieval failure rather than digesting what it read. For a runtime presentation, retrying cannot succeed, so the request is rejected with `invalid_client` rather than `temporarily_unavailable` ({{errors}}), except on refresh-token use, for which {{refresh}} takes precedence.

A server MUST NOT compare a digest against any representation other than one it retrieved itself or stored under this section. A shared cache might hold a variant selected for another request. A `304 Not Modified` response shows that stored octets, and their digest, are still current only if it carries a strong entity tag that matches the one stored with them under strong comparison (Section 8.8.3.2 of {{RFC9110}}). Otherwise, the server retrieves the body again.

## Validating a Statement {#validation}

Before accepting a statement, a trusting authorization server MUST ensure the following:

* The `typ` JOSE header parameter has the value `software-statement+jwt`.
* The signature verifies under a key trusted for the exact `iss` value, obtained as {{issuer-trust}} requires.
* Every required claim of {{profiles}} is present and of the correct type.
* The statement is not expired, and its `iat` is not unreasonably far in the future according to the server's clock-skew policy.
* If the statement carries `aud`, a value in it exactly matches one of the server's audience identifiers that its trust configuration permits the statement's issuer to name ({{issuer-trust}}).
* If the statement carries `aud_tenant`, `aud` carries exactly one value and `aud_tenant` identifies the tenant this request belongs to, resolved before the statement is evaluated, by the server's own means or from a signed assertion it has validated, and never from a value the client chooses.
* If the server requires a tenant-scoped decision for the request, the statement carries `aud_tenant`, or the server's trust configuration confines the statement's issuer to the request's tenant ({{issuer-trust}}).
* If the statement carries `statement_uses`, the claim names the use being made of the statement, and a value the server does not recognize is treated as naming no use it serves. If the claim is absent, the use being made is presentation.
* The `sub` claim is a client identifier URL conforming to {{CIMD}} and falls within the identifier scope for which this server accepts the issuer ({{issuer-trust}}).
* Any claim the server does not recognize, and any client metadata claim, which {{profiles}} forbids an issuer to include, is ignored.
* The JWT validation guidance in {{RFC8725}} has been applied.

A trusting authorization server that hosts more than one tenant MUST evaluate every tenant-specific check, such as the `aud_tenant` claim and a required issuer ({{issuer-trust}}), against the tenant that scopes what it issues on the statement, never selected by a value the client chooses.

A trusting authorization server MUST reject a statement for which it holds a refusal record ({{withdrawal}}). Outside registration, the rejection is `statement_required` ({{errors}}). A server that uses a withdrawal mechanism for the statement's issuer SHOULD consult it before accepting the statement for a new admission, within the caching and freshness rules that mechanism defines.

A trusting authorization server resolving the reviewed document MUST reject a document containing duplicate object member names. Parsers interpret such names differently despite an identical digest.

Rejections outside a registration endpoint use {{errors}}.

## Withdrawal {#withdrawal}

An issuer can end a decision before the statement's `exp` by withdrawing the statement. Some status mechanisms call this revocation. This specification uses a distinct term because OAuth revocation ({{RFC7009}}) ends tokens, while a withdrawal revokes no issued token: a server holding a refusal record refuses new admissions and refresh under the statement ({{refresh}}, {{enforcement-bounds}}).

How a trusting authorization server learns of a withdrawal is a withdrawal mechanism. This specification requires no withdrawal mechanism and defines none. Extensions define how a mechanism applies to these statements ({{extensions}}).

A refusal derived from a withdrawal mechanism lasts until a later authoritative result from that mechanism supersedes it, or until an expiry the mechanism itself defines for the withdrawal. A server MUST NOT retain such a refusal once it is superseded or expired in that way. A failure to consult the mechanism, or cached evidence that has lapsed, neither supersedes nor expires a refusal. A refusal that the server's operator entered persists on that operator's own terms.

A withdrawal constrains and never relaxes. A statement for which no withdrawal mechanism reports a withdrawal, or for which the server cannot consult the mechanism it uses, is bounded by its effective expiry as it would be otherwise, and a server MUST NOT treat any withdrawal mechanism as grounds to accept a statement past its effective expiry.

What a server does when it cannot consult a mechanism it relies on is local policy. Refusing makes issuer availability a precondition for every request the statement governs. Proceeding leaves a withdrawn statement acceptable until it expires.

An issuer that uses a withdrawal mechanism and issues a replacement narrower than the statement it replaces SHOULD withdraw the earlier statement. A replacement is narrower if the document it names no longer carries a key, redirection URI, or scope the earlier document carried, if its `aud`, `aud_tenant`, or `statement_uses` is narrower, or if its remaining lifetime is shorter.

What was removed might be why the replacement was issued. A trusting authorization server that has not yet seen the replacement, or at which it does not validate, would otherwise accept the earlier statement until it expires.

Withdrawal does not replace lifetime: an issuer chooses `exp` assuming no trusting authorization server uses a withdrawal mechanism.

# Issuer Trust Establishment {#issuer-trust}

A trusting authorization server accepts statements only from configured issuers. Trust is established out of band, for example through a marketplace publisher program or shared enterprise operation. This specification defines no in-band issuer discovery or trust decision.

For each trusted issuer, a trusting authorization server records at least the following:

* The exact `iss` identifier it will accept.
* The source of that issuer's statement signing keys: the `software_statement_jwks_uri` in the issuer's authorization server metadata ({{authorization-server-metadata}}), discovered from the configured `iss` as {{RFC8414}} describes, and never its `jwks_uri`.
* The signing algorithms it will accept from the issuer.
* The client identifier namespaces the issuer can attest through `sub`.
* The audience identifiers the issuer can name.
* The maximum statement lifetime it will honor, which sets each statement's effective expiry.
* Its policy on repeated and multiple registration ({{multi-instance}}).

These inputs, not the signature alone, define acceptance. A trusting authorization server MUST derive trust from this local configuration and MUST NOT derive it from an `iss`, `jku`, `x5u`, or other key-location value carried in a presented statement.

Trust configuration SHOULD constrain each issuer to the client identifier namespaces it is expected to attest, for example URLs under the domains of the software publishers it serves. {{validation}} rejects a statement outside that scope. Without such a scope, a compromised or over-broad issuer can mint acceptable statements about any client software.

When a trusting authorization server configures an issuer whose decisions are confined to one of its tenants, as a customer's own issuer is, it SHOULD record with that configuration the client identifier namespaces for which the issuer's decision is required in that tenant.

A server that records such a requirement MUST NOT admit software in a covered namespace in that tenant without a current statement from the required issuer, whatever its policy for clients it has not reviewed. Without this rule, withholding the statement would restore access the customer withdrew. The requirement is configured rather than inferred from the statements the server sees, so a holder cannot evade a customer's withdrawal by presenting or publishing only a broader statement, such as a marketplace listing.

A required issuer configured for software whose documents are presented review-only ({{public-client-presentation}}) excludes that software from the tenant.

## Multi-Tenant Issuers {#multi-tenant-issuers}

If an issuer reviews software for several customers whose decisions are each trusted independently, it MUST issue under a distinct `iss` value for each customer, so that each customer's decisions, keys, and `iat` floor ({{multi-instance}}) stay separate. Under one `iss`, one customer's renewal would supersede another's review. A statement carries no claim naming the issuer's customer.

# Runtime Presentation {#cimd-presentation}

A client presents its statement in a request ({{runtime-presentation}}) or publishes it where its document points ({{pulled-statements}}).

## Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for presentation. The software statement ({{profiles}}). It is consumed as a runtime presentation, a refresh replacement ({{refresh}}), or, if {{REGISTRATION}} is implemented, a delivery under that specification. A request carrying the parameter in any other role is rejected with `invalid_request`.

The request's `client_id` is the client's Client ID Metadata Document URL. It MUST exactly equal the statement's `sub` claim. If the two differ, the authorization server MUST reject the presentation with the `invalid_client` error code. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

The request carries the proof that {{sender-constraint}} requires. A successful presentation establishes the client for the request and for the grant state derived from it ({{grant-lifecycle}}).

At the token endpoint, a runtime presentation is valid on a request using any one of the following grants:

* The `client_credentials` grant.
* The JWT or SAML 2.0 assertion grant (Section 2.1 of {{RFC7523}} and Section 2.1 of {{RFC7522}}).
* Another grant whose specification provides for runtime presentation under this specification.

Authorization-code redemption and refresh-token use continue an existing grant ({{grant-lifecycle}}, {{refresh}}). A request that asks an authorization server to issue a software statement, rather than to consume one, MUST NOT carry the `software_statement` parameter.

An authorization server that does not implement this specification ignores the parameter (Sections 3.1 and 3.2 of {{RFC6749}}), so a client can present wherever its statement's audience allows.

### Authorization Endpoint {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}. A pulled statement needs no pushed authorization request ({{pulled-statements}}). The pushed authorization request endpoint processes the statement and its proof as {{processing}} describes. The subsequent authorization request MUST use a `client_id` parameter exactly equal to the establishment's `sub` and MUST NOT include the `software_statement` parameter. A statement therefore never appears in a front-channel URL.

## Pulled Statements {#pulled-statements}

A client can publish its statements rather than present them. Its Client ID Metadata Document then carries the following client metadata member:

`software_statements_uri`:
: OPTIONAL. URL, in a Client ID Metadata Document, at which the client publishes software statements about itself. The member has no meaning in a registration request. The URL MUST use the `https` scheme. A request to it returns a JSON object whose `software_statements` member is an array of statements ({{profiles}}), each a string in JWS compact serialization. The metadata digest covers this member ({{metadata-digest}}), so changing where statements are found changes the document's digest, and statements over the earlier document no longer match.

An authorization server that supports pulled statements MAY retrieve this URL when it resolves the document of a client that presented no statement. In retrieving this URL, the server does the following:

* Send a `GET` request with `Accept: application/json`, do not automatically follow redirects, and accept only a `200` response with a JSON media type.
* Apply the protections {{CIMD}} sets for URLs a document contains and the bounds of {{external-retrieval}}.
* Bound the response size and the number of statements examined separately from the document's. Several issuers' statements outgrow a document-sized bound, and a host can pad the array.
* Discard statements from unconfigured issuers before verifying any signature.
* Cache a successful response no longer than the document is cached, and cache a failure no longer than {{CIMD}} permits for the document.

The authorization server can revalidate a cached response from that URL with a conditional request, as it can the document ({{metadata-digest}}). Both the document and this URL are retrieved before any statement exists to validate, so the ordering of {{processing}} applies from the first statement verified onward.

The server considers only statements that meet all of the following conditions:

* The `sub` claim equals the document's client identifier.
* The statement validates under {{validation}} and permits presentation ({{profiles}}).
* The `cimd_digest` claim equals the digest of the document the server resolved.

The server ignores the rest.

If the server's trust configuration requires an issuer for this client ({{issuer-trust}}), it applies only that issuer's statement, and if none remains, the client is not reviewed. Otherwise, if more than one statement remains, the server keeps only the statement with the latest `iat` from each issuer and applies any one of those, subject to the `iat` floor of {{multi-instance}}. A pulled statement the server does not apply does not advance the `iat` floor.

Having selected a statement, the server continues from step 4 of {{processing}}, with the pulled statement in place of a presented one and the document already resolved. The selection above satisfies the `client_id` rule of step 2.

The presenter of a pulled statement is bound as for a presented statement ({{sender-constraint}}), at the point the request allows, as follows:

* At the token endpoint, the request's client authentication or DPoP proof binds the presenter.
* At the pushed authorization request endpoint, a confidential client's authentication binds the presenter, and a public client whose request binds a key through a DPoP proof or `dpop_jkt` is subject to {{public-client-presentation}}, including PKCE.
* At the authorization endpoint, binding completes at code redemption, before any token is issued:
  * A confidential client authenticates at code redemption with a key the digested document carries, in its `jwks` or at its `jwks_uri` within the reuse bounds of {{sender-constraint}}, not one that only a later retrieval of the document carries, and that key becomes the establishment's proven key.
  * A public client's authorization request that binds a key through `dpop_jkt` is subject to {{public-client-presentation}}, including PKCE, and redemption proves the bound key.
* A public client opening a new grant at the token endpoint, or sending a pushed or ordinary authorization request that binds no key, has nothing to bind, so a statement pulled for it is review-only ({{public-client-presentation}}), as is one pulled for a document whose redirection URIs that section does not accept.
* Code redemption and refresh continue an establishment and keep the binding it already has ({{grant-lifecycle}}, {{refresh}}), except that redemption sets the proven key of an establishment whose statement was pulled at the authorization endpoint for a confidential client.

If retrieval does not complete or no statement remains, the server applies its policy for Client ID Metadata Document clients it has not reviewed, subject to any required issuer ({{issuer-trust}}). If that policy requires reviewed software, the server rejects the request with `temporarily_unavailable` if retrieval did not complete and with `statement_required` if no statement remains. A retrieval failure is never a withdrawal.

When a refresh needs a replacement ({{refresh}}) for an establishment created from a pulled statement, the server pulls again and applies the newest statement with the recorded statement's `iss` and `sub` as the replacement under that section. If the newest is the recorded statement itself, nothing is replaced, and the refresh is evaluated against the recorded statement as {{refresh}} describes. If the re-pull does not complete or yields no replacement, the refresh fails as {{refresh}} provides. A reviewed grant never continues under the policy for clients the server has not reviewed.

An authorization server advertises support through `software_statement_pull_supported` ({{authorization-server-metadata}}).

## Presentation Processing {#processing}

On receiving a presentation, the authorization server proceeds as follows, rejecting as {{errors}} defines at the first failure:

1. Validate the statement ({{validation}}), completing every check that does not require a client-controlled network retrieval before performing any such retrieval.
2. Verify the statement requirements of {{profiles}}, the `iat` floor of {{multi-instance}}, and the `client_id` rule of {{runtime-presentation}}.
3. Resolve the reviewed document ({{effective-metadata}}), which is the first step requiring a client-controlled retrieval.
4. Verify the sender constraint against that document ({{sender-constraint}}), then evaluate the request against its metadata ({{effective-metadata}}).
5. On success, create the establishment ({{grant-lifecycle}}).

## Sender Constraint {#sender-constraint}

A runtime presentation MUST be sender-constrained by a key the statement attests or, under {{public-client-presentation}}, by the key the presenter binds there. The presenter proves possession of that key through the applicable client authentication method or a DPoP proof {{RFC9449}}. If a request carries both, the key its client authentication proves is the proven key. The proof MUST be bound to the current request and validated with the replay protections of that mechanism.

Except as {{public-client-presentation}} provides, the proven key MUST appear in the `jwks` or at the `jwks_uri` of the reviewed document ({{effective-metadata}}). If the reviewed document declares one or more client authentication methods ({{CIMD}}), the presenter MUST use one of them. The authorization server MUST reject a presentation without such a proof, or whose proven key the reviewed document does not carry.

This specification requires a reviewed document to declare its client authentication method, so that the issuer's review and the trusting authorization server's treatment of the client do not depend on a default. The authorization server MUST reject a presentation under a reviewed document that omits the `token_endpoint_auth_method` member. It treats a statement pulled for such a document as review-only ({{public-client-presentation}}).

A document carrying a redirection URI another application could claim is presented review-only whatever its authentication method ({{public-client-presentation}}), and the requirements above do not apply to that presentation. Two rejections still apply to such a document: a presentation under a document that omits `token_endpoint_auth_method`, as above, and a presentation at the token endpoint from a client whose reviewed document carries no key material ({{public-client-presentation}}).

A statement whose reviewed document carries no key material can be consumed at registration if its `statement_uses` claim permits. {{REGISTRATION}} permits registering a document declaring `token_endpoint_auth_method` of `none` only under rules that bind it as {{public-client-presentation}} does. At runtime, a statement for such a document is consumed under {{public-client-presentation}}.

A key at the document's `jwks_uri` is retrieved at presentation time. If that retrieval fails, the presentation is rejected with `temporarily_unavailable`. A server MAY reuse a recently retrieved key set within ordinary HTTP caching bounds, subject to a maximum reuse period of its own choosing. The server MUST NOT let the client's cache directives alone determine how long a removed key continues to verify ({{external-retrieval}}).

### Public Clients {#public-client-presentation}

Software distributed to end users cannot hold a key its reviewed document carries: a key shipped in every copy identifies the software, not the installation. Such a document declares `token_endpoint_auth_method` of `none` and carries no key material.

If the client's reviewed document declares `token_endpoint_auth_method` of `none` and none of its redirection URIs is one another application could claim (see below), a presentation at the pushed authorization request endpoint, or a statement pulled at that endpoint or at the authorization endpoint ({{pulled-statements}}), is bound instead by its reviewed redirection URIs. For such a presentation, the authorization server:

* MUST require PKCE {{RFC7636}} with the `S256` method.
* MUST require the presenter to bind a key it holds, through the `dpop_jkt` parameter or, at the pushed authorization request endpoint, a DPoP proof (Section 10.1 of {{RFC9449}}), and records that key as the proven key of the establishment ({{grant-lifecycle}}).
* MUST NOT require that key to appear in the reviewed document.

The reviewed document, not the proven key, binds the presentation here: an authorization code opened by such a presentation is delivered only to a redirection URI the issuer reviewed, so a holder of a copied statement cannot receive a code that another resource owner approves in a user agent the holder does not control ({{public-client-security}}). The proven key identifies the installation the grant belongs to. Code redemption and refresh prove that key ({{grant-lifecycle}}, {{refresh}}).

A presentation is review-only if the reviewed document carries any redirection URI that another application on the same device could claim: a non-`https` URI, such as a private-use scheme or an `http` loopback redirection URI (Sections 7.1 and 7.3 of {{RFC8252}}), or an `https` URI whose host is a loopback address or `localhost`. One such URI makes every presentation and pulled statement for the document review-only, whatever authentication method the document declares. The application that claims such a URI receives the code, so neither the reviewed redirection URIs nor a key shipped in every copy of the software can bind the presenter.

A review-only presentation creates no establishment, admits nothing, and does not advance the `iat` floor of {{multi-instance}}, since it changes nothing for the software's other instances. The authorization server proceeds as it would for the same Client ID Metadata Document client presenting no statement.

The statement of a review-only presentation MUST NOT satisfy a policy requiring reviewed software, and the authorization server SHOULD NOT present that review to the user as an assurance about the presenter. If the server's policy requires reviewed software, it rejects a review-only client with `unauthorized_client`.

For a review-only presentation, the authorization server can record the statement's issuer for audit and inventory.

An authorization server MUST reject, with the `invalid_client` error code, a presentation at the token endpoint under {{runtime-presentation}} from a client whose reviewed document carries no key material. Such a presentation opens no redirect and has nothing to bind it. A statement pulled for such a client at the token endpoint is instead review-only ({{pulled-statements}}).

## Reviewed Metadata {#effective-metadata}

Having validated the statement, and before verifying the proof ({{processing}}), the authorization server MUST obtain the Client ID Metadata Document at the statement's `sub`, only by retrieval or from octets it has already digested for that identifier, and compare its digest with the `cimd_digest` claim. If octets it already holds do not match, it retrieves the document once more before treating the mismatch as a change, since a statement over an updated document would otherwise fail at a server still holding older bytes.

Octets the authorization server already holds serve for that comparison only within a maximum reuse period of its own choosing, as a key set does ({{sender-constraint}}). The authorization server MUST NOT let the client's cache directives alone determine how long held octets continue to match a statement. Without this rule, a key the document carries inline would continue to verify after the publisher removes it.

A match means the served document is the reviewed one, and its members are the client's metadata for the request. A mismatch means the document changed after review, and the authorization server MUST NOT treat the changed document as reviewed. It rejects the presentation with `invalid_client` or, by local policy, proceeds without the statement. If it proceeds, it applies its policy for Client ID Metadata Document clients it has not reviewed to the current document and creates no establishment for the statement.

At registration, a mismatch is fatal ({{REGISTRATION}}). Either way, the remedy is a statement over the current document. Because an issuer reviews the document it retrieves, a changed document is always served before a statement over it exists, and the statement's lifetime bounds how stale an unobserved review can become.

The request is evaluated against the reviewed metadata as against any client's registered metadata ({{RFC6749}}, {{CIMD}}): any requested grant type, response type, or scope MUST fall within it. A document that omits `scope` sets no ceiling on scope beyond the server's own policy.

A request made when the bound on establishments of {{multi-instance}} is reached is rejected with `invalid_client`. Its statement and proof are sound, so the authorization server ought to state that reason in `error_description`, and the client can retry once earlier establishments are released.

## Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment comprising the following:

* The validated `sub`.
* The statement identity (its `iss` and `jti`), its `iat` and effective expiry, and any claim by which a withdrawal mechanism locates it, such as `status`.
* The authorization server's own tenant the grant was opened for, if it hosts more than one.
* The reviewed metadata and the digest it matched ({{effective-metadata}}).
* The issuer trust decision.
* The sender-constraint mechanism and proven key.

An authorization server can reuse an establishment across presentations that resolve to the same `sub`, statement, and proven key, rather than creating one per request.

The establishment persists while the grant depends on it. The authorization server MUST bind the resulting `request_uri`, authorization code, refresh token, and other grant continuation state to the establishment, as applicable, and can discard it once no such state references it.

A replacement accepted on refresh MUST apply only to the grant refreshed ({{refresh}}). Any other grant, `request_uri`, or authorization code bound to the same establishment keeps the state its request was evaluated against.

A token request that redeems an authorization code opened by a presentation:

* MUST have a `client_id` parameter exactly equal to the establishment's `sub`, even if the client authenticates.
* MUST demonstrate possession of the establishment's proven key under the same sender-constraint mechanism or, for a confidential client whose statement was pulled at the authorization endpoint ({{pulled-statements}}), authenticate with the key that becomes the proven key.
* MUST NOT carry the `software_statement` parameter.

A redemption carrying a statement is rejected with `invalid_request`. A wrong client identifier or failed key binding is rejected with `invalid_grant`. The prohibition on carrying a statement covers redemption of a code bound to an establishment. A registered client redeeming its own code can deliver a replacement statement under {{REGISTRATION}}, which that specification calls a delivery and which is not a presentation.

Expiry after presentation does not by itself invalidate an establishment already bound. The authorization server controls continued use as {{refresh}} defines.

### Refresh {#refresh}

On refresh-token use, the authorization server MUST verify possession of the establishment's proven key under the same sender-constraint mechanism. Except for an establishment created under {{public-client-presentation}}, it MUST also verify that the document currently served at the establishment's `sub` still carries that key, in its `jwks` or at its `jwks_uri`, within the reuse bounds of {{sender-constraint}}, and rejects the refresh with the `invalid_grant` error code if that document does not carry the key or exceeds the size bound of {{metadata-digest}}.

On refresh-token use, the authorization server MAY, by local policy, additionally require a current unexpired statement, and SHOULD require one once the establishment's recorded statement has expired ({{enforcement-bounds}}). The recorded statement satisfies that requirement while it is unexpired and no refusal record covers it, even after a later statement has raised the `iat` floor of {{multi-instance}}. A replacement is needed only once the recorded statement expires or a refusal record covers it.

If the authorization server holds a refusal record for the establishment's recorded statement, it MUST reject a refresh without a presented or pulled replacement satisfying this section, whatever its policy on requiring a current statement otherwise: a withdrawal ends grant continuation at once rather than waiting on local policy. To let a withdrawal reach open grants and not only new ones, a server that uses a withdrawal mechanism for the recorded statement's issuer SHOULD consult it for that statement at each refresh.

When a replacement is needed, the client presents it in the `software_statement` parameter of the refresh request, or, for an establishment created from a pulled statement, the server pulls one ({{pulled-statements}}). A statement with the recorded statement's `iss` and `jti` is not a replacement: if the client offers it again, the server rechecks it as the recorded statement, under the first two paragraphs of this section. The following requirements apply to the replacement:

* It MUST validate under {{validation}}, including its audience if it carries one.
* The `iss` and `sub` claims MUST equal those of the recorded statement.
* The `iat` claim MUST be later than that of the recorded statement.
* The `cimd_digest` claim MUST name the document currently served at that `sub`, which the server confirms by retrieval or by revalidating octets it holds.
* The replacement MUST authorize the establishment's proven key ({{sender-constraint}}) or, for an establishment created under {{public-client-presentation}}, name a document that still satisfies that section.

The refreshed access MUST fall within the metadata of the document the replacement names.

On success, the refreshed grant's statement identity, `iat`, effective expiry, reviewed metadata, and trust decision are replaced in a single atomic update, and concurrent replacements resolve to the most recently issued statement. A server that shares the establishment with other grants gives the refreshed grant its own establishment, so no other grant or outstanding authorization code changes ({{grant-lifecycle}}). A refresh that fails these requirements, or omits a replacement that is needed, leaves the grant's state unchanged and is rejected with `statement_required`, or with `temporarily_unavailable` if a retrieval needed to check the replacement did not complete for a reason other than the size bound of {{metadata-digest}}.

Refresh never rotates the establishment's key. A client that needs a new key performs a new presentation.

# Repeated Consumption of One Statement {#multi-instance}

A statement attests client software, identified by `sub`, not its runtime instances. This specification defines no instance identifier. One unexpired statement is therefore consumable more than once: at each trusting authorization server in its audience and, if the server registers clients ({{REGISTRATION}}) and its policy permits, in more than one registration at that server.

To prevent an issuer's statements for different tenants from superseding one another, a multi-tenant authorization server keeps the `iat` floor and the bounds described below per tenant ({{tenant-confusion}}).

A trusting authorization server MUST retain, per `iss` and `sub` claims, the `iat` claim of the most recent statement it has accepted, as its `iat` floor. It MUST reject a statement whose `iat` claim is earlier than the retained `iat`, whether it arrives as a replacement, a new registration, or a new presentation. A client holding a superseded, broader statement therefore cannot defer a narrowing by opening a fresh registration or establishment with it. The server retains the `iat` floor at least as long as the maximum statement lifetime it honors for that issuer.

The first presentation of a replacement that establishes the client supersedes its predecessor for new admissions by every instance of the software at that server, so instances need the current statement from a shared source rather than a copy fixed at build or install time. Grants the predecessor already opened continue under {{refresh}}.

A trusting authorization server SHOULD bound the number of establishments, and any registrations ({{REGISTRATION}}), derived from one issuer's statements about one piece of software, identified by `iss` and `sub` claims. How the server counts toward a bound is local policy, within two limits:

* State that anyone holding a copy of a statement can create does not count: a pushed authorization request and its unredeemed code, which anyone holding a copy of a public client's statement can approve with their own account, or a registration under which no access token has been issued. Otherwise a holder of a copy could exhaust the bound for the software's other instances.
* A count persists across replacements: an establishment created for a refreshed grant does not add to it, and one keyed on `jti` would reset at every renewal.

For software admitted under {{public-client-presentation}}, whose every installation opens its own establishment, the server SHOULD also bound establishments per resource owner, so that one user cannot exhaust a bound the software's other users share.

Software whose instances authenticate as the client with keys of their own, which the reviewed document does not carry, can neither prove nor register those keys ({{sender-constraint}}, {{REGISTRATION}}) and depends on the attested-key extension of {{extensions}}.

# Relationship to Other Work {#relationship}

This section is non-normative.

## Client Attestation {#relationship-attestation}

A software statement and a client attestation {{ABCA}} are both signed third-party assertions. A statement attests client software and the metadata its issuer evaluated. It is signed by an issuer trusted for a set of client identifiers and accepted wherever that issuer is trusted. A client attestation attests a running instance and the key it holds, and is presented with a proof bound to the request.

Neither assertion grants access nor substitutes for the other. Runtime presentation therefore always requires a sender constraint ({{sender-constraint}}). Binding a presenter attestation defined elsewhere is an extension ({{extensions}}).

## Identity Assertions {#identity-assertions}

A statement records that a named party reviewed software, not that a particular customer currently permits that software to act in its tenant. A tenant-scoped decision (`aud_tenant`) authorizes no particular user or transaction. When a customer's identity provider mediates a grant, as with a cross-domain identity assertion {{IDJAG}}, it already answers the customer's question per grant by issuing assertions only for clients the customer permits. A statement serves when the issuer is not in the request path, as in the following cases:

* A publisher program vouches to servers it will never see.
* A provider admits software it has never registered.
* A customer wants a portable record of what its review covered.

## OpenID Federation {#relationship-federation}

A trust mark {{OPENID-FED}} is also a signed third-party assertion about an entity, but it is resolved through a federation that supplies key discovery, policy, and delegation, and in which both parties enroll. A statement needs no federation: a trusting authorization server configures its issuer directly ({{issuer-trust}}), and the statement binds the exact octets of the client's own document rather than metadata the federation resolves.

# Error Responses {#errors}

A rejection uses the error response and HTTP status codes specified for the endpoint where it occurs:

* At the token endpoint, Section 5.2 of {{RFC6749}} applies.
* At the pushed authorization request endpoint, Section 2.3 of {{RFC9126}} applies.
* At the authorization endpoint, Section 4.1.2.1 of {{RFC6749}} applies.

At the token and pushed authorization request endpoints, an authorization server SHOULD respond with HTTP status code 503 to a request it rejects with the `temporarily_unavailable` error code. The status code lets a client distinguish a retrieval worth retrying from a condition that needs a new statement. {{REGISTRATION}} defines rejections at a registration endpoint and how registration validity is reported.

## Requiring a Current Statement {#statement-required}

This specification defines the following error code:

`statement_required`:
: A current software statement is required and none was supplied or pulled, or the one supplied or pulled is expired, refused by a refusal record ({{withdrawal}}), or superseded under the `iat` floor of {{multi-instance}}. The code also applies when a statement offered as a replacement ({{refresh}}, {{REGISTRATION}}) does not satisfy the requirements for one. Re-presenting the same statement cannot succeed. The client recovers by obtaining a statement issued more recently.

Whenever this specification requires a current statement on a refresh-token request, `statement_required` replaces `invalid_grant`. The distinct code keeps a client from reading the rejection as an invalid refresh token ({{RFC9700}}).

## Presentation {#presentation-errors}

A presentation rejected at the token or pushed authorization request endpoint for a reason this specification defines uses the following error codes:

`invalid_client`:
: The statement or the client cannot be established, in any of the following cases:

  * The statement is malformed, or fails signature or claim validation ({{validation}}) for a reason other than expiry.
  * The statement is valid but not acceptable here: its issuer is not configured, its `aud` excludes this server, its `sub` is outside the issuer's scope, its `aud_tenant` is not this request's tenant or is absent when required, or its `statement_uses` excludes this use.
  * The request's `client_id` differs from the statement's `sub` ({{runtime-presentation}}).
  * The reviewed document contains duplicate member names ({{validation}}) or exceeds the server's size bound ({{metadata-digest}}).
  * The document's digest does not match `cimd_digest` and the server does not proceed without the statement ({{effective-metadata}}).
  * The request fails the sender constraint, including a presentation under a document that omits `token_endpoint_auth_method` and one at the token endpoint under a document with no key material ({{sender-constraint}}, {{public-client-presentation}}).
  * The bound on establishments of {{multi-instance}} is reached ({{effective-metadata}}).

`statement_required`:
: The statement is expired, refused by a refusal record, or superseded, or a required statement is absent, including one from an issuer required for the tenant ({{issuer-trust}}).

`unauthorized_client`:
: The request exceeds the reviewed document's grant types or response types ({{effective-metadata}}), the document carries metadata this server's policy refuses, or the client is review-only and this server requires reviewed software ({{public-client-presentation}}).

`invalid_scope`:
: The request exceeds the reviewed document's scope ({{effective-metadata}}), or the document carries a scope this server's policy refuses.

`temporarily_unavailable`:
: A retrieval the presentation needs did not complete, for a reason other than the document size bound ({{metadata-digest}}).

`invalid_request`:
: The request carries `software_statement` in a role {{runtime-presentation}} does not permit.

A request rejected for another reason, such as an assertion grant whose assertion is invalid (Section 3.1 of {{RFC7523}}), uses the error code its grant or endpoint assigns.

## Code Redemption and Refresh {#continuation-errors}

A token request that redeems an authorization code opened by a presentation is rejected with `invalid_request` if it carries `software_statement`, with `invalid_grant` if its client identifier or key binding fails ({{grant-lifecycle}}), and with `invalid_client` if the bound on establishments of {{multi-instance}} is reached ({{effective-metadata}}).

Under an establishment, a refresh-token request rejected for a reason this specification defines uses the following error codes:

`statement_required`:
: A current statement or a replacement is needed ({{refresh}}) and none is supplied or pulled, or the one supplied or pulled fails {{refresh}}: it is malformed, invalid, not acceptable here, expired, refused, or superseded, or it names a document whose digest does not match, that contains duplicate member names, that exceeds the size bound, or that no longer satisfies {{public-client-presentation}}.

`invalid_grant`:
: The request does not prove the establishment's proven key, or the document currently served at the establishment's `sub` no longer carries that key or exceeds the document size bound ({{refresh}}).

`temporarily_unavailable`:
: A retrieval needed to check the replacement or the proven key did not complete, for a reason other than the document size bound ({{metadata-digest}}).

`unauthorized_client` or `invalid_scope`:
: The refreshed access would exceed the reviewed document's metadata or this server's policy, as for a presentation.

`invalid_request`:
: The request carries `software_statement` in a role {{runtime-presentation}} does not permit.

None of these rejections indicates refresh-token replay ({{RFC9700}}), and neither does a rejection reporting registration validity ({{REGISTRATION}}). A refresh-token request rejected for another reason, such as an expired or revoked refresh token, uses the error codes of Section 5.2 of {{RFC6749}}. This specification does not change how an authorization server detects refresh-token replay.

## Authorization Endpoint {#authorization-errors}

A statement is never presented at the authorization endpoint ({{authorization-requests}}) but can be pulled there ({{pulled-statements}}). There, if reviewed software is required ({{public-client-presentation}}, {{pulled-statements}}), a review-only client is rejected with `unauthorized_client`, and a client whose pulled statements could not be retrieved is rejected with `temporarily_unavailable`.

Apart from those two cases, at the authorization endpoint, if the authorization server requires a statement for a client it has not admitted on one, it MUST return the `statement_required` error code in the authorization error response (Section 4.1.2.1 of {{RFC6749}}). The code tells the client to obtain a statement and return through the pushed authorization request endpoint. Without the code, a client cannot tell a missing review from a policy it will never satisfy.

The authorization server redirects with the error only after resolving the client's Client ID Metadata Document and validating the request's `redirect_uri` against it. If the authorization server cannot resolve the document, it MUST NOT redirect, and it reports the error to the resource owner instead.

An authorization request carrying the `software_statement` parameter ({{authorization-requests}}) is rejected with `invalid_request` in place of any other code of this section. The authorization server redirects with that error only after validating the request's `redirect_uri`: for a client identified by its Client ID Metadata Document URL, against that document as the preceding paragraph requires, and for a registered client, against its registered redirection URIs (Section 4.1.2.1 of {{RFC6749}}).

## Proof Mechanism Errors {#proof-errors}

If the proof mechanism defines an error of its own for a proof it does not accept, such as `invalid_dpop_proof` for a DPoP proof that fails the checks of Section 4.3 of {{RFC9449}} or `use_dpop_nonce` for a nonce challenge (Section 8 of {{RFC9449}}), the request is rejected with that error in place of the code this specification assigns to a failed sender constraint.

A proof that its mechanism accepts but that proves a key other than the one this specification requires, such as a key the reviewed document does not carry or one other than the establishment's proven key, is not an invalid proof: it fails the sender constraint, and is rejected with the code this specification assigns to that failure.

## Error Descriptions {#error-descriptions}

These codes do not tell a client whether to seek a corrected statement or a different issuer. The authorization server can return non-sensitive diagnostics in the `error_description` parameter to a client it has authenticated, but MUST NOT reveal details of its issuer trust configuration ({{issuer-trust}}), such as the client identifier namespaces an issuer can attest, to an unauthenticated requester.

If a statement is refused by a refusal record rather than merely expired, and the client is authenticated, the authorization server ought to indicate that a statement issued more recently is required.

# Examples {#example}

Both examples are non-normative.

## Presenting at the Token Endpoint

The following is an example of a client presenting a statement at the token endpoint of a server holding no record for the client (line breaks for display purposes only). The reviewed document names a `jwks_uri`, the `private_key_jwt` authentication method, and the `client_credentials` grant type, so the client authenticates with a key that document carries. The `client_id` is the statement's `sub`.

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

The following is an example of a Client ID Metadata Document that names where its statements are published:

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
* Runtime presentation on grants other than those {{runtime-presentation}} names, such as token exchange.
* Partial review, by which an issuer vouches for particular members rather than a whole document ({{profiles}}).
* Withdrawal mechanisms ({{withdrawal}}), such as Token Status List {{STATUSLIST}}.

# Security Considerations {#security-considerations}

The security considerations of {{CIMD}} and Section 5 of {{RFC7591}} apply.

## Statement Theft and Replay {#statement-validation}

A copied statement admits nothing without the sender constraint ({{sender-constraint}}). Admitting only keys the reviewed document carries also prevents downgrade. An extension admitting attested keys ({{extensions}}) has to address downgrade itself.

Attesting a `jwks_uri` attests the location, not its contents: a compromised key host can add keys that satisfy the proof without a digest change. If that matters, an issuer attests `jwks` inline and accepts that rotation changes the digest. A server reusing a cached key set ({{sender-constraint}}) also accepts that a just-removed key can briefly continue to verify.

## Servers That Do Not Implement This Specification {#legacy-servers}

A statement carries no client metadata, so an {{RFC7591}} server that verified one without implementing this specification would take every metadata value from the registration request, attaching the issuer's approval to an attacker's redirection URIs and keys. Keeping statement signing keys at `software_statement_jwks_uri`, out of `jwks_uri`, and away from statements of other profiles ({{authorization-server-metadata}}) guards against this. A server that discovers an issuer's keys through the `jwks_uri` in its {{RFC8414}} metadata finds none that verify a statement. A server is exposed only if its operator configures the statement key set in a server that does not implement this specification.

## Copied Statements and Public Clients {#public-client-security}

A party holding a copied statement for a public client can open a pushed authorization request in the reviewed software's name. It cannot receive an authorization code that another resource owner approves in a user agent the party does not control ({{public-client-presentation}}), so other users are exposed only to a consent prompt carrying the reviewed software's name and branding. Rate limits ({{external-retrieval}}) and the issuer's choice of audience and lifetime bound that exposure.

A party that controls the user agent receives the code: an application embedding a user agent (Section 8.12 of {{RFC8252}}), or a holder authorizing with its own account. It redeems the code with a PKCE verifier and key of its own choosing. An admission under {{public-client-presentation}} therefore shows where codes are delivered, not which software holds the grant, and a tenant's required issuer ({{issuer-trust}}) cannot keep unreviewed software out for a user who chooses to run it. This is the client impersonation that Section 8.6 of {{RFC8252}} describes for public clients. An attested-key extension ({{extensions}}) would strengthen the binding by giving the instance a key a third party vouches for.

## Tenant Confusion at a Multi-Tenant Authorization Server {#tenant-confusion}

A multi-tenant server that derives the tenant one way to check a decision and another way to scope a token lets a client reach one tenant's resources on another tenant's decision. {{validation}} requires one tenant for both, selected by no value the client chooses. The `aud_tenant` claim makes that failure detectable: a statement naming a tenant cannot be used in another.

This specification defines no tenant parameter, and a Client ID Metadata Document URL is the same `client_id` in every tenant, so neither a client nor an issuer can verify that a server honored the tenant rule of {{validation}}.

The `aud_tenant` claim names a tenant at the trusting authorization server, not which of the client's own customers is asking. Software that acts for many customers under one client identifier and key presents the same statement and key for each, so a statement naming a tenant admits the software there on behalf of every customer it serves, and any of those customers can direct the software at another customer's tenant. A statement attests software and cannot prevent this. A deployment that needs a customer-to-tenant binding records it at the trusting authorization server.

## External Retrieval and Resource Exhaustion {#external-retrieval}

Runtime presentation can cause the authorization server to retrieve the Client ID Metadata Document, a `jwks_uri` it names, or the statements its `software_statements_uri` names. Every such retrieval inherits the server-side request forgery protections of {{CIMD}}. A trusted signature does not make a URL safe: the authorization server MUST apply its URL, redirect, address-range, transport, and content-type policy independently to every referenced location.

These retrievals occur before any client is registered or any user has interacted, so an unauthenticated requester holding one acceptable statement can cause this work. An authorization server SHOULD therefore bound this work as follows:

* Rate-limit presentations per statement identity, per subject, and per source, and pulls ({{pulled-statements}}) per client identifier and per source, before spending retrieval or storage on a new presentation.
* Refuse a presentation that would create a new establishment, before spending retrieval or storage on it, once the bound of {{multi-instance}} for its `iss` and `sub` is reached.
* Bound JWT size and parsing work, concurrent retrievals, response size, and response time.
* Cache successful retrieval results within the document's caching directives, and back off after a failure without caching it, which {{CIMD}} forbids.

A retrieval failure leaves the relevant metadata or proof unverified, so the authorization server MUST reject the request and MUST NOT fall back to a weaker proof. A failed pull of statements for a request that opens a new grant is the exception: it leaves the client unreviewed, as {{pulled-statements}} describes. A failed pull of a replacement that a refresh requires is not an exception: the refresh fails ({{refresh}}).

## Statement Lifetime {#enforcement-bounds}

Expiry and withdrawal ({{withdrawal}}) govern new admissions. Neither revokes an issued access token. A grant opened by a statement lasts for the life of its refresh tokens, whatever the statement's lifetime, unless the server requires a current statement on refresh, holds a refusal record, or finds the proven key gone from the client's document ({{refresh}}). A narrowed review takes effect when the client publishes the narrower document and obtains a statement over it.

Short lifetimes tighten an issuer's control and increase renewal traffic. At a server that registers clients under {{REGISTRATION}}, the lifetime also sets the registration's renewal cadence. A fleet of statements issued together expires together, so issuers ought to stagger expiries or renew ahead of the boundary.

## Observable State {#oracle-considerations}

A `statement_required` refusal at the authorization endpoint ({{errors}}) travels through the user agent, so anyone who can see the redirection learns that this client has no established standing at this server. The disclosure is accepted because a client that cannot learn why it was refused cannot act. The disclosure carries no detail of the server's issuer trust configuration, which {{errors}} keeps from unauthenticated requesters.

## Statement Handling

Authorization servers ought to avoid logging software statements, which are sensitive in transit and at rest. A statement names reviewed software and, if it carries an audience, that software's intended relationships.

# Privacy Considerations

The privacy considerations of {{CIMD}} and Section 6 of {{RFC7591}} apply.

A presentation or delivery reveals to the authorization server the client's issuer relationship and, if the statement carries an `aud` claim, the other authorization servers the client intends to use. Omitting that claim discloses nothing beyond the review. Pushed authorization requests ({{authorization-requests}}) keep statements out of browser history, referrers, and front-channel logs.

A central issuer also learns from renewals which of its statements are in active use, so issuance and renewal logs deserve the same care as the statements.

Retrieving the document tells its host that the server is resolving the client. Pulling statements ({{pulled-statements}}) tells the host of `software_statements_uri` the same. That host is an additional party only if the URL is on another origin. Published statements are readable by anyone, so their `aud` and `aud_tenant` claims disclose which servers and tenants a review names. A statement whose audience is sensitive is presented rather than published.

# IANA Considerations {#iana}

## OAuth Parameters Registry

This specification requests registration of the `software_statement` parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}, for runtime presentation and statement delivery of the artifact defined by {{RFC7591}}:

Parameter Name:
: `software_statement`

Parameter Usage Location:
: authorization request, token request

Note:
: In an authorization request the parameter is conveyed only through the pushed authorization request endpoint {{RFC9126}}, never in a front-channel URL ({{authorization-requests}}).

Change Controller:
: IETF

Specification Document(s):
: This specification, {{runtime-presentation}}

The "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}} already contains a `software_statement` member, which this registration does not affect. The name is reused because the artifact is the same {{RFC7591}} software statement, carried to the same server for the same purpose. A distinct name would make a client carry one artifact under two names, and a server would not recognize at the token endpoint what it accepts at the registration endpoint.

## Media Type Registration {#media-type}

This specification requests registration of the `application/software-statement+jwt` media type in the IANA "Media Types" registry {{RFC6838}}.

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
: Authorization servers and clients that issue, request, or accept OAuth 2.0 software statements

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

This specification uses two claims it does not define: `aud_tenant`, defined and registered by {{IDJAG}}, and `status`, defined and registered by {{STATUSLIST}}.

## JSON Web Token Claims Registry

This specification requests registration of the following values in the IANA "JSON Web Token Claims" registry established by {{RFC7519}}:

Claim Name:
: `cimd_digest`

Claim Description:
: Unpadded base64url-encoded SHA-256 digest of the retrieved octets of the Client ID Metadata Document evaluated during software statement issuance

Change Controller:
: IESG

Specification Document(s):
: This specification, {{profiles}}

Claim Name:
: `statement_uses`

Claim Description:
: Permitted uses of a software statement

Change Controller:
: IESG

Specification Document(s):
: This specification, {{profiles}}

## OAuth Extensions Error Registry

This specification requests registration of the following error code in the IANA "OAuth Extensions Error Registry" established by {{RFC6749}}:

Error Name:
: `statement_required`

Error Usage Location:
: authorization error response, token error response, pushed authorization request error response

Related Protocol Extension:
: CIMD Software Statement

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

This specification requests registration of the following client metadata member, published by a client in its Client ID Metadata Document:

Client Metadata Name:
: `software_statements_uri`

Client Metadata Description:
: URL, in a Client ID Metadata Document, at which the client publishes software statements about itself for an authorization server to retrieve. It is not meaningful in a registration request.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{pulled-statements}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following values in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_signing_alg_values_supported`

Metadata Description:
: JSON array containing the asymmetric JWS algorithms supported by the authorization server for signing software statements.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_jwks_uri`

Metadata Description:
: URL of the JWK Set containing only the keys with which the authorization server signs software statements.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_pull_supported`

Metadata Description:
: Boolean value indicating whether the authorization server retrieves software statements from a client's `software_statements_uri`.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

--- back

# Acknowledgments
{:numbered="false"}

This mechanism began as the consumption part of a draft on software statement issuance and was split out so that consumption can evolve separately from issuance. It draws on working-group discussion of Client ID Metadata Documents and attestation-based client authentication.
