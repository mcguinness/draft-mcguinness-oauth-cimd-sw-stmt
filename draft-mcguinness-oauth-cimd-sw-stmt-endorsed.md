---
title: "Endorsed Instance Keys for CIMD Software Statements"
abbrev: CIMD Endorsed Keys
docname: draft-mcguinness-oauth-cimd-sw-stmt-endorsed-latest
category: std
submissiontype: IETF

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Client ID Metadata Document
 - Client Attestation
 - Native Application
 - Instance Binding

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
  RFC6838:
  RFC7515:
  RFC7519:
  RFC7591:
  RFC7636:
  RFC7638:
  RFC7800:
  RFC8252:
  RFC8414:
  RFC8725:
  RFC9111:
  RFC9126:
  RFC9207:
  RFC9449:
  RFC9700:
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  ABCA: I-D.ietf-oauth-attestation-based-client-auth
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "CIMD Software Statement"
    author:
      - fullname: Karl McGuinness

informative:
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "CIMD Software Statement Registration"
    author:
      - fullname: Karl McGuinness

--- abstract

A CIMD software statement binds a review of client software to the exact bytes of its Client ID Metadata Document. It does not, by itself, bind an installation of that software to a key. This specification defines an instance-binding extension in which an independently trusted client attester endorses an installation key for the reviewed document. The client proves that key through attestation-based client authentication when opening and continuing an authorization code grant. Where an attester can verify which software controls an installation's key, as on managed devices, this permits native applications, including command-line clients, to use loopback redirects while satisfying a policy requiring reviewed software. The specification defines attestation and metadata requirements, processing at the pushed authorization request and token endpoints, and the exceptions to the core software statement rules needed for endorsed keys.

--- middle

# Introduction

A native application's installations cannot safely authenticate by a private key distributed in every copy of the software. Each installation can generate its own key, but possession of that key establishes continuity, not the identity of the software using it. A copied Client ID Metadata Document {{CIMD}} URL and software statement can be presented with an attacker's own key.

{{STATEMENT}} therefore treats a document containing a redirect that another local application could claim as review-only: the server records the review but admits nothing on it, and such a presentation cannot satisfy a policy requiring reviewed software. This excludes loopback redirects, even though they are an established response-delivery mechanism for native applications under {{RFC8252}}.

Existing mechanisms do not close this gap. Loopback redirects with PKCE {{RFC7636}} protect an authorization code from interception, but any application can start a transaction with a native client's identifier, the client impersonation that Section 8.6 of {{RFC8252}} describes. Attestation-based client authentication {{ABCA}} alone shows that an installation holds a key an attester vouches for, not that the installation runs the software a reviewer approved for a particular document. The core's public-client binding relies on redirection URIs no other application can claim, which loopback URIs are not.

This specification defines an alternative binding made of three checks, which stay separate even when one organization operates both issuing roles:

* The software statement records approval of the software and its exact metadata.
* An instance attestation, from an independently trusted client attester under an evidence policy the authorization server accepts, associates an installation's key with that software and metadata.
* A proof under attestation-based client authentication demonstrates possession of that key.

The callback delivers the authorization response; it supplies no evidence of the presenter's software identity.

The extension serves authorization code flows using pushed authorization requests (PAR) {{RFC9126}} and refresh of the resulting grants. It defines neither registration nor new token-endpoint presentations for other grants. {{REGISTRATION}} is unchanged: it rejects loopback redirects, so a document selecting this profile is never registered, and an issuer omits `registration` from the `statement_uses` of a statement over one. Core and registration do not depend on this extension.

The binding is only as strong as the attester's evidence that the software, not merely the device or the user, controls the key ({{software-evidence}}). Such evidence is available today mainly on managed endpoints, through an enterprise agent or launcher ({{launcher}}). On an unmanaged desktop, or for an interpreted command-line client, an attester typically cannot meet the requirements of {{trust}}, and the client remains review-only under {{STATEMENT}}.

## Protocol Overview

The following sequence is non-normative:

1. A publisher serves a CIMD document declaring this instance binding, attestation-based client authentication, and loopback redirects. A reviewer issues a software statement over its exact bytes.
2. An installation obtains an attestation from an accepted attester. The attestation names the CIMD URL, metadata digest, instance key, and evidence policy.
3. The installation sends a PAR request with PKCE, the attestation, and a DPoP proof. It supplies the software statement or lets the server pull it.
4. The server validates both trust decisions and their bindings before returning a `request_uri`.
5. The user authorizes in an external browser. The code returns over loopback and is redeemed with the PKCE verifier and the endorsed key.
6. Refresh requires the original key, a current acceptable attestation, and a current acceptable software statement. A different key requires a new grant.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology from {{RFC6749}}, {{CIMD}}, and {{ABCA}}, and the terms Runtime Presentation, Effective Expiry, Refusal Record, Current Statement, Review-Only, Establishment, and Proven Key from {{STATEMENT}}, which this specification calls the core specification, or core. It also uses the core's `iat` floor: the `iat` of the most recent statement a server has accepted for an `iss` and `sub`, below which it rejects statements.

Instance Binding:
: The association of a proven instance key with the reviewed software and metadata under this specification. It does not assign a global installation identifier.

Instance Attestation:
: A Client Attestation JWT under {{ABCA}}, profiled by {{attestation}}. It records an attester's association of a key with a CIMD identifier and metadata digest under a named evidence policy. An instance attestation is current while its attestation effective expiry has not passed and its attester and evidence policy remain accepted.

Evidence Policy:
: A versioned set of requirements for establishing the association between software, an instance key, and reviewed metadata. It covers evidence collection, verification, freshness, and protection of key use.

Attestation Effective Expiry:
: An instance attestation's `exp`. An accepted evidence policy sets it no later than the time the attester's evidence was collected plus the policy's maximum evidence age ({{trust}}), so an attestation before its `exp` rests on evidence no older than that maximum.

The software statement's issuer and the client attester are separate roles. One entity can perform both, but neither role implies trust in the other.

# Scope and Profile Selection {#selection}

An authorization server implementing this specification MUST support PAR, PKCE with `S256` {{RFC7636}}, DPoP {{RFC9449}}, and the `attest_jwt_client_auth_dpop` method of {{ABCA}}, which uses a DPoP proof as the proof of possession (Sections 5.2 and 7.3 of {{ABCA}}). Clients MUST use that method at PAR and at every token request under this specification. The same asymmetric key is used for client authentication and DPoP binding. The normal Client Attestation PoP JWT mode is not supported by this profile. Each installation holds its own credential, the endorsed key, so a client under this profile is a confidential client although it is a native application, as Section 8.4 of {{RFC8252}} contemplates for per-instance credentials.

The client selects this profile by publishing `software_statement_instance_binding` with value `true` in its reviewed document ({{client-metadata}}). A server accepts a new establishment under the profile only if it supports the extension, the reviewed document selects it, and all admission checks succeed. The instance attestation's distinct `typ` identifies the corresponding attestation profile but does not alone enable admission.

Once an establishment is created, its recorded binding selects the profile for every continuation. Selection MUST NOT depend on whether a continuation happens to include an attestation or on a newly fetched document omitting the selection member. Such omissions fail the request; they do not change the grant's binding.

This specification permits only the authorization code grant opened through PAR, and refresh of that grant. It does not authorize private-use URI schemes, `localhost` hostnames, non-loopback redirects, implicit responses, device authorization, client credentials, or assertion grants. Other specifications can define additional bindings without changing the behavior of this profile.

# Client Metadata {#client-metadata}

This specification defines the following member of a Client ID Metadata Document:

`software_statement_instance_binding`:
: OPTIONAL. Boolean. When `true`, the client requires the instance binding defined by this specification for admissions under this document. If absent or `false`, the document does not select this profile. A value of any other type is invalid.

A document selecting this profile MUST meet all of the following requirements:

* Its `client_id` is its HTTPS document URL, as required by {{CIMD}}.
* Its `token_endpoint_auth_method` is the single string `attest_jwt_client_auth_dpop`. No alternative authentication method is offered under this profile.
* Its `grant_types` explicitly contains `authorization_code` and MAY also contain `refresh_token`. It contains no other grant type.
* Its `response_types` explicitly contains only `code`.
* Its `redirect_uris` is a nonempty array, every member of which meets {{loopback}}.
* It MUST NOT contain `jwks` or `jwks_uri`. Instance keys are authorized only through {{attestation-validation}}, not by document key material.

The document MAY contain `software_statements_uri` as defined by {{STATEMENT}}. Metadata governing requested access, such as `scope`, remains subject to that specification. Every member, including the binding selection, is covered by the statement's digest. This member is not a registration request parameter and does not change {{REGISTRATION}}.

## Loopback Redirects {#loopback}

Every reviewed redirect URI MUST use `http`, have the host exactly `127.0.0.1` or `[::1]`, and contain an absolute path. User information and fragments MUST NOT be present. A reviewed entry MUST NOT contain a port: the client uses the port of its listener, which Section 7.3 of {{RFC8252}} lets the server accept. The document MAY list both address forms.

The server MUST compare a requested redirect with a reviewed entry using exact string matching, except for the port variation permitted by Section 7.3 of {{RFC8252}}. No other component can vary: the address form, path, and any query remain fixed. In particular, a wildcard path, a hostname resolving to loopback, an alternative numerical spelling of the address, and a different loopback address do not match.

The client MUST bind its listener only to the selected loopback interface, open it before sending PAR, and close it when the transaction completes or is abandoned. It SHOULD obtain an available ephemeral port from the operating system. The authorization code is delivered to that listener; the authorization server MUST NOT fetch the redirect URI to test the listener.

# Authorization Server Metadata {#server-metadata}

This specification defines the following authorization server metadata member under {{RFC8414}}:

`software_statement_instance_binding_supported`:
: OPTIONAL. Boolean indicating support for this specification. If omitted, its value is `false`. A server advertising `true` MUST implement the requirements of this specification. Support does not promise acceptance of any particular client, statement issuer, attester, evidence policy, or tenant.

A supporting server MUST publish its `pushed_authorization_request_endpoint`, include `attest_jwt_client_auth_dpop` in `token_endpoint_auth_methods_supported`, advertise `S256` in `code_challenge_methods_supported`, and publish the algorithm metadata required by {{ABCA}} and {{RFC9449}} for the chosen method. It MUST support {{RFC9207}} and publish `authorization_response_iss_parameter_supported` as `true`.

A client MUST confirm support for this extension before starting a grant under it. The generic attestation method advertisement alone is insufficient. This profile requires PAR for its clients even when the server also serves other clients without PAR; it does not require the server's global `require_pushed_authorization_requests` value to be `true`.

# Attester Trust and Evidence {#trust}

An authorization server MUST establish attester trust independently of the request. For each accepted attester it MUST configure:

* The exact attester `iss` identifier and how its verification keys are resolved, as Section 10.8 of {{ABCA}} recommends that a profile fix: either an `x5c` certificate chain ({{RFC7515}}) validated to a configured trust anchor together with a configured mapping from the certificate to this attester, such as an exact subject or subject alternative name; or a `kid` matched against a JWK Set configured for this attester. A `jku` is followed only if it names that configured JWK Set. Sharing a trust anchor with another attester does not authorize a key for this attester's `iss`.
* Accepted asymmetric signing algorithms.
* Client identifier namespaces the attester may endorse.
* Accepted evidence-policy identifiers and, for each, a finite maximum evidence age.
* Tenant restrictions and any requirements for a particular attester or policy within a tenant.
* Clock-skew and evidence-freshness requirements and any local denial rules for instance keys.

The server MUST use the tenant determined for the grant by its own trusted processing, as {{STATEMENT}} requires. A client-supplied tenant label is not sufficient. The server MUST NOT infer attester trust from a statement issuer's signature, from the client document, or from a key-location claim in the attestation. An `iss`, a `jku` or `x5u` header parameter ({{RFC7515}}), or a similar value in an unverified JWT MUST NOT cause unconfigured key discovery. Retrieval from a configured key source remains subject to external-retrieval protections and bounded cache reuse.

The server cannot observe how an attester reached its decision, so it relies on the evidence policy. An evidence policy is acceptable for reviewed-software admission only if it meets all of the following requirements:

* Before issuing each instance attestation, the attester verifies control of the private instance key through an authenticated, replay-resistant exchange bound to the submitted public key.
* The attester evaluates evidence about the software instance and protection of key use, and sets each attestation's `exp` no later than the time that evidence was collected plus the policy's maximum evidence age. Rounding or randomizing `exp` (Section 11.1 of {{ABCA}}) only moves it earlier. A previous attestation is not evidence: a renewal rests on new evidence under the same bound.
* The attester establishes an authenticated association between the software identity and the CIMD URL, retrieves and validates the document under {{CIMD}}, and calculates its digest under {{STATEMENT}}.
* The attester confirms that the artifact or package identity established by the evidence is covered by that association for the document version being endorsed. The association is authorized by whoever controls the origin of the CIMD URL, or by an authority that controller has explicitly delegated, for example through a manifest served from that origin or signed under that controller's or delegate's authority. An enterprise inventory can authenticate or record such an authorization; listing a package and someone else's CIMD URL together does not create one. A URL, file path, digest, or publisher name asserted only by the requesting process does not suffice. The metadata digest is not an executable digest.
* The policy defines what identifies the software, how evidence authenticates that identity and the instance key, how use of the key is restricted to the software instance the evidence covers, a finite maximum evidence age, and the treatment of updates, scripts, plugins, and other executable extensions.
* The policy's identifier denotes immutable policy semantics: a material policy change requires a new identifier.
* The policy establishes an association between the software the statement covers and control of the endorsed key. User approval of a key, a publisher account login, possession of a shared embedded secret, or device integrity alone does not establish that association.

A server MUST NOT accept for this profile an evidence policy that does not meet these requirements. A deployment MAY use weaker evidence for other purposes but MUST NOT report it as satisfying this profile.

Attestation acquisition and platform evidence formats are out of scope. They require an interoperable agreement between the installation and its attester; the wire format below does not by itself supply a software-verification mechanism. {{launcher}} gives an implementation model and its prerequisites.

# Instance Attestation {#attestation}

An instance attestation is a Client Attestation JWT (Section 4 of {{ABCA}}) with the following additional requirements, which Section 13 of {{ABCA}} permits a profile to define. Together with the document's selection of this profile ({{selection}}), its type tells a server that this profile applies. Its JOSE `typ` MUST be `cimd-instance-attestation+jwt`. It MUST be digitally signed using an accepted asymmetric algorithm; a MAC or `none` MUST NOT be accepted. Its header and claims MUST contain no duplicate member names. Validation MUST apply {{RFC8725}}.

The attestation retains the `sub`, `cnf` ({{RFC7800}}), `iat`, and `exp` requirements of {{ABCA}}. Its `sub` MUST be the exact CIMD client identifier URL. In addition, it MUST contain:

`iss`:
: The attester's identifier, an HTTPS URL without a fragment. It is matched exactly against configured trust; URL normalization MUST NOT be used to broaden that match.

`cimd_digest`:
: The metadata digest defined by {{STATEMENT}}, computed over the document the attester evaluated. This claim has the same representation and comparison rules as in a software statement. Its use here records the attester's software-to-metadata association, not a second software review.

`cimd_instance_policy`:
: A string containing an absolute URI identifying the evidence policy under which the attester endorsed the instance key. The URI is an identifier, not a retrieval instruction. It MUST be compared exactly with an accepted policy identifier.

The attestation MUST have `exp` later than `iat`. Its `exp` bounds the age of the evidence behind it, as the accepted evidence policy requires ({{trust}}). Its `iat` is not an evidence timestamp, and an attester can round or randomize it as Section 11.1 of {{ABCA}} recommends. A server MUST reject an attestation whose `exp` has passed, or lies further in the future than the maximum evidence age it accepts for the attester and evidence policy, and MUST NOT reject an attestation only because its `iat` was rounded or randomized. No field supplied by the client can extend these bounds.

This specification does not require an attestation audience or installation identifier. If an `aud` is present, the receiving server MUST validate it under {{RFC7519}}. The proof targets a particular endpoint ({{proof}}); the attestation can be usable at more than one trusting server. An attestation is not tied to a statement's `jti`, so a statement renewed over unchanged bytes need not cause re-attestation.

## Validation {#attestation-validation}

At each PAR request, code redemption, and refresh, the authorization server MUST validate the instance attestation and its proof. It MUST verify all of the following before accepting the binding:

* The attestation meets {{attestation}} and the applicable {{ABCA}} validation requirements.
* The signature verifies with a key authorized for the exact `iss`, for this artifact type and accepted algorithm.
* The attester and policy are currently accepted for this client identifier and tenant, and the key is not denied by local policy.
* The request's `client_id`, the software statement's `sub`, and the attestation's `sub` are exactly equal.
* The attestation's `cimd_digest`, the applicable software statement's `cimd_digest`, and the digest of the reviewed document are equal.
* The attested public key and proven DPoP key are equal under the key-comparison rules of Section 7.3 of {{ABCA}}.

On continuation, the recorded key, attester, policy, and tenant constraints of {{establishment}} also apply. The server MUST NOT treat an attestation for another digest as permission to update metadata without the software statement replacement required by {{refresh}}.

## Proof of Possession {#proof}

The client sends the instance attestation in `OAuth-Client-Attestation` and uses DPoP combined mode under Section 7.3 of {{ABCA}}. It MUST NOT also send `OAuth-Client-Attestation-PoP`. A `dpop_jkt` parameter alone is not a proof. The authentication method and its validation remain those defined by {{ABCA}}; this profile supplies the attestation's type and additional bindings.

The authorization server MUST require a server-issued DPoP nonce at PAR and on every token request under this profile. It MUST enforce endpoint and method binding and reject replay of a proof within its acceptance window under {{RFC9449}}. Nonce challenges and retries use `use_dpop_nonce` and `DPoP-Nonce`; they do not consume a code, rotate a refresh token, create an establishment, or advance the `iat` floor. Nor does a request rejected before the instance is authenticated, or with `use_fresh_attestation` or `temporarily_unavailable`, so the client can retry with a fresh attestation.

If the PAR request also carries `dpop_jkt`, its JWK thumbprint ({{RFC7638}}) MUST identify the endorsed key; otherwise the server rejects the request. A client MUST generate a new proof for each request, including a retry. TLS protects request contents. DPoP does not sign the request body, and implementations MUST NOT treat a valid proof as a signature over the redirect URI, PKCE challenge, or statement.

# Runtime Admission {#admission}

## Pushed Authorization Request {#par}

The client MUST start every grant under this profile at the PAR endpoint. Its request MUST include `client_id`, `response_type=code`, `redirect_uri`, a PKCE `code_challenge` with `code_challenge_method=S256`, and a transaction-specific `state`. It MUST authenticate under {{proof}}. A request using a Request Object MUST meet {{RFC9126}} and MUST NOT provide conflicting parameter values; the following checks apply to the effective authorization request.

The client presents its software statement using `software_statement` as defined by {{STATEMENT}}, or the server obtains a statement through that specification's pulled-statement mechanism.

Before returning a successful PAR response, the server MUST:

1. Validate request syntax and the client identifier, resolve the applicable tenant, and authenticate the instance under {{trust}}, {{attestation}}, and {{proof}}. Checks needing no client-controlled retrieval precede such retrieval.
2. Validate the selected statement under {{STATEMENT}}, including issuer and tenant policy, audience, effective expiry, permission for runtime presentation, refusal records, and the applicable `iat` floor. Where statements are pulled, apply that specification's retrieval and selection rules.
3. Resolve the CIMD document and perform the digest and metadata checks of {{STATEMENT}}. Parse the same bytes it digested, enforce {{client-metadata}}, and complete the cross-artifact checks of {{attestation-validation}}.
4. Validate the requested redirect under {{loopback}}, require PKCE `S256`, and evaluate the request against the reviewed metadata and authorization policy.
5. Create or reuse an establishment under {{establishment}}, bind the accepted request to it, and return a PAR response under {{RFC9126}}.

For step 1, authentication establishes the attester, policy, client identifier, and proven key; it is not yet reviewed-software admission. An attestation whose digest has not been checked cannot establish the client. The ordering of core's statement checks before client-controlled retrieval applies once a statement is available. External-retrieval bounds, HTTP caching rules ({{RFC9111}}), and protection against server-side request forgery from {{STATEMENT}} and {{CIMD}} apply throughout.

The server MUST NOT fall back to review-only or unreviewed admission for the same request if any required check fails. If no acceptable statement is available, it returns `statement_required`. A failed request MUST NOT advance the statement's `iat` floor or alter an existing establishment.

## Establishment {#establishment}

An establishment records all state required by {{STATEMENT}} and, in addition:

* That this instance-binding profile governs the grant.
* The exact attester identifier and evidence-policy identifier.
* The attester trust decision and the attestation effective expiry most recently accepted.

The proven key is the attestation key used for DPoP. A server MAY reuse an establishment only when the core reuse conditions hold and the profile, attester, policy, and tenant also match. Individual authorization transactions still retain separate PKCE challenges, redirects, `state`, resource-owner decisions, and continuation state.

The client identifier, tenant, key, attester, policy, and binding profile MUST remain fixed for the lifetime of the grant. Changing any of them requires a new grant opened through PAR. A fresh attestation for the same tuple can renew its evidence, and a software statement replacement can update its reviewed metadata only as {{refresh}} permits. Neither operation changes the tuple.

The server MUST bind its `request_uri`, code, refresh token, and other grant continuation state to the establishment. Core's establishment limits and counting rules apply. Servers SHOULD also bound establishments per resource owner and tenant so that one installation cannot exhaust the capacity available to other installations.

## Browser Authorization {#authorization}

The client MUST use an external user-agent under {{RFC8252}}. The browser authorization request MUST use the `request_uri` from PAR and the same `client_id`. It MUST NOT carry the software statement, instance attestation, or an instance proof. The server MUST use the stored PAR parameters and MUST NOT allow front-channel parameters to change the redirect, PKCE challenge, key binding, or tenant.

The server MUST reject a direct authorization request without a PAR `request_uri` governed by this profile when the client document selects this profile or server policy requires it. A statement pulled at the authorization endpoint cannot create an exception to that requirement.

The server authenticates the resource owner and applies authorization policy independently of the software and instance checks. It MUST return the authorization response with the issuer identifier defined by {{RFC9207}}. The client MUST validate that identifier and `state` against the transaction before redeeming a code. Callback validation and OAuth security requirements in {{RFC8252}} and {{RFC9700}} continue to apply.

## Authorization Code Redemption {#redemption}

The client sends `grant_type=authorization_code`, the code, the same `client_id` and `redirect_uri`, and the PKCE `code_verifier`. It MUST authenticate with a currently acceptable instance attestation and a fresh proof under the recorded key. It MUST NOT send `software_statement` on this request, retaining core's prohibition on a presentation during code redemption.

Before issuing tokens, the server MUST verify the code and PKCE binding and every fixed establishment value in {{establishment}}. It MUST re-evaluate both trust decisions, require the recorded statement to be unexpired and not refused, and confirm that the current CIMD document still matches its recorded digest and meets {{client-metadata}}. Retrieval and cache reuse use core's freshness bounds. The attestation MUST match that digest and be within its effective expiry.

An attestation MAY have been renewed since PAR, but its key, attester, policy, and subject MUST match the establishment. If the metadata changed or the recorded statement needs replacement, the client MUST start a new PAR transaction; code redemption cannot replace the governing review. A raised `iat` floor alone does not invalidate a code already bound to an acceptable recorded statement.

The server MUST issue tokens only within the user's authorization, tenant policy, and reviewed metadata. It MUST bind any refresh token to the establishment, and MUST issue access tokens bound to the endorsed key with a `token_type` of `DPoP` (Section 5 of {{RFC9449}}). The effective expiry of each access token, whether the token is a JWT or an opaque reference, MUST NOT be later than the attestation effective expiry accepted for the request that obtained it. A client therefore refreshes, with a fresh attestation, at least as often as its attester's evidence age allows. A resource server that does not accept DPoP-bound access tokens cannot be used under this profile. A DPoP proof shows possession of the endorsed key at the resource server; it does not show that the software was re-evaluated at that moment.

# Grant Continuation {#refresh}

Every refresh request MUST carry the original client identifier, the refresh token, a current instance attestation, and a fresh DPoP proof under the establishment's original key. The server MUST authenticate the instance, verify the refresh-token binding, and enforce the fixed values of {{establishment}} before accepting any replacement statement.

On every refresh the server MUST require a current, unexpired, unrefused software statement and re-evaluate its issuer trust decision. This strengthens the optional statement-currency check in core. It MUST also verify that the currently served document matches the applicable statement and attestation digests and continues to meet {{client-metadata}}. Removal of the binding member or a change to another authentication method ends continuation under this profile.

The recorded statement can remain applicable while it is unexpired, not refused, and matches the current document, even if a newer statement has raised the `iat` floor for new admissions. If it no longer qualifies, a replacement is required. The client can present a replacement in `software_statement` or, for an establishment created from a pulled statement, the server pulls one as {{STATEMENT}} specifies. A request carrying the same `iss` and `jti` as the recorded statement is a recheck, not a replacement.

A replacement MUST satisfy core's replacement validation, including identical statement `iss` and `sub`, a later `iat`, and current metadata. Its decision MUST apply to the establishment's recorded tenant and runtime use. Instead of authorizing a document key or preserving core's public-client conditions, the replacement MUST preserve this profile and authorize the original instance key through a current attestation from the recorded attester under the recorded policy. The replacement and attestation MUST name the same current metadata digest. A replacement cannot expand access beyond the original grant, resource-owner authorization, or current server policy.

On success, the server MUST atomically update the recorded statement, metadata, digest, and trust decisions and apply core's concurrency rules for replacements. It also records the newly accepted attestation effective expiry. A failed attempt MUST NOT replace the recorded binding or rotate a refresh token. This does not prevent the server from recording a withdrawal or trust removal learned during processing, or applying independent refresh-token replay protections.

Attestation renewal for the original key is not key rotation and does not reset the `iat` floor or an establishment count. This specification defines no migration of an existing grant to a new key, attester, evidence policy, or tenant.

## Withdrawal and Trust Changes {#withdrawal}

A known refusal record for the software statement is enforced as {{STATEMENT}} requires. A current acceptable replacement can restore continuation only under {{refresh}}. A server using a statement withdrawal mechanism SHOULD consult it at code redemption and refresh within that mechanism's freshness rules. Failure to consult it does not clear an existing refusal.

At PAR, code redemption, and refresh, the server MUST enforce current attester trust and local denial rules. Removing an attester's authorization for this client and tenant, removing the policy, or denying the instance key prevents further use of the affected binding. A fresh attestation does not override those decisions. A deployment that cannot refresh trust data MUST NOT accept it beyond a configured maximum staleness bound.

This specification defines no interoperable attestation or instance revocation protocol. Deployments MAY use independently established mechanisms or local denial lists. In their absence, attestation effective expiry bounds acceptance of evidence that has become stale without the server learning of it. Withdrawal of a review, removal of trust, and expiry of evidence do not themselves revoke access tokens already issued. Their further use is bounded by the resource's token-validation and revocation mechanisms.

# Relationship to Core Processing {#core-exceptions}

For establishments governed by this specification, the following rules replace or add to the corresponding rules of {{STATEMENT}}:

| Core section | Core rule | Under this profile |
| --- | --- | --- |
| 4 | A required issuer configured for software whose documents are presented review-only excludes that software from the tenant. | A presentation processed under {{par}} is not review-only, so a current statement from the required issuer can admit the software. |
| 5.2 | A statement pulled at the authorization endpoint binds the presenter at code redemption. | The binding is set at PAR; an authorization request without a PAR `request_uri` governed by this profile is rejected ({{authorization}}). |
| 5.3, 5.5 | The statement and the document are checked before the proof. | The instance is authenticated first ({{par}}, step 1). Statement and document checks follow, and checks needing no client-controlled retrieval still precede such retrieval. |
| 5.2, 5.4.1, 5.5 | A request without an acceptable statement can proceed under the policy for unreviewed clients, a review-only presentation proceeds as a client presenting no statement, and a digest mismatch can proceed without the statement. | No fallback: any failed check rejects the request ({{par}}). |
| 5.4 | The proven key appears in the reviewed document's `jwks` or at its `jwks_uri`. | The proven key is the endorsed instance key ({{attestation-validation}}), and the document carries no key material ({{client-metadata}}). |
| 5.4.1 | A redirection URI another application could claim makes a presentation review-only, and a document declaring `none` is bound by its redirection URIs. | A presentation processed under {{par}} whose redirects meet {{loopback}} is not review-only. The client is confidential and authenticates with the method of {{selection}}, so the public-client rules do not apply. |
| 5.6 | Expiry after presentation does not invalidate an establishment, and redemption checks the client identifier and key. | Redemption also requires the recorded statement to be unexpired and unrefused, and the current document to match the recorded digest ({{redemption}}). |
| 5.6.1 | A current statement on refresh is a local-policy option, the document is checked for the proven key, and its digest is checked only for a replacement. | Every refresh requires a current statement, re-checks the document against the recorded digests, and verifies continued endorsement of the key instead of its presence in the document ({{refresh}}). |
| 8.3 | Code redemption is rejected with `invalid_request` or `invalid_grant`. | Redemption can also return `statement_required`, `use_fresh_attestation`, or `temporarily_unavailable` ({{errors}}). |
| 8.4 | A client not admitted on a statement receives `statement_required`. | A missing or invalid required pushed authorization request is rejected with `invalid_request` ({{errors}}). |

Code redemption and refresh continue an establishment and are not new token-endpoint presentations, so core's rejection of a token-endpoint presentation from a client whose document carries no key material is unchanged. All other core requirements continue to apply, including digest processing, issuer trust, statement audience and tenant checks, refusal records, the `iat` floor, and bounds on repeated consumption. The extension applies equally to presented and pulled statements. It adds no requirement to issue statements or implement {{REGISTRATION}}.

An implementation of core alone does not gain this exception. Its review-only treatment of claimable redirects continues to apply. A supporting server MUST NOT infer this profile's assurance from successful generic client authentication without all of this specification's checks.

# Error Responses {#errors}

The endpoint-specific syntax and response rules of {{RFC6749}}, {{RFC9126}}, {{STATEMENT}}, {{ABCA}}, and {{RFC9449}} apply. This specification defines no new error code. The following table applies at PAR, code redemption, and refresh, except where a condition names a particular endpoint. Other OAuth errors, such as `invalid_scope`, continue to apply to their respective failures.

| Error | Condition |
| --- | --- |
| `invalid_request` | Malformed or conflicting request parameters, or a `software_statement` parameter on code redemption |
| `invalid_client` | Missing or unacceptable attestation, untrusted attester or policy, a DPoP proof under a key other than the attested key, or a request carrying an instance attestation for a document that does not select this profile or violates {{client-metadata}} |
| `use_fresh_attestation` | An otherwise valid attestation whose `exp` has passed, or that names a digest other than the current reviewed document's |
| `use_dpop_nonce` | Missing or unacceptable server DPoP nonce |
| `invalid_grant` | At redemption or refresh, a validly authenticated instance does not match the recorded key, attester, policy, or client; or, at redemption, the current document no longer matches the recorded digest |
| `statement_required` | A required statement is missing, expired, refused, or superseded, or, at refresh, a needed replacement is missing or unacceptable |
| `temporarily_unavailable` | A retrieval needed for validation temporarily fails |

Invalid statements and reviewed documents at PAR otherwise use core's error rules. At refresh, an unacceptable replacement uses `statement_required` as in core. A document exceeding a configured size bound is not a transient failure; apply core's non-transient retrieval and refresh errors. An invalid or expired code or refresh token, failed PKCE check, or mismatched redemption redirect uses `invalid_grant`.

A DPoP proof that fails the checks of Section 4.3 of {{RFC9449}} is rejected with `invalid_dpop_proof`, and a missing or unacceptable nonce with `use_dpop_nonce`, in place of any code in the table above, as {{STATEMENT}} provides for proof-mechanism errors. A valid proof under a key other than the attestation's `cnf` key is not an invalid proof: authentication fails (Section 7.3 of {{ABCA}}), and the request is rejected with `invalid_client`, or with `invalid_client_attestation` as Section 7.4 of {{ABCA}} permits. At code redemption and refresh, an instance that authenticates under a key other than the establishment's recorded key fails the grant binding and is rejected with `invalid_grant`.

Servers MUST check basic request syntax before reporting errors requiring authentication. They MUST authenticate the instance before reporting errors about a code, refresh token, or the review governing that grant. Within authentication, they MUST verify attester identity, signature, and policy before returning `use_fresh_attestation`. Nonce challenges MAY precede completion of authentication and MUST NOT reveal tenant trust configuration. When several failures remain at one stage, a server MAY choose any applicable error without disclosing additional failed checks.

A `use_dpop_nonce` response MUST include `DPoP-Nonce`. Client attestation errors and status codes follow Section 7.4 of {{ABCA}}; other errors retain their endpoint's response format. A `software_statement` parameter on code redemption is `invalid_request`, even if it offers an otherwise valid replacement. A `statement_required` response at code redemption requires a new PAR transaction, not a retry with a statement attached to the code request.

At the authorization endpoint, a missing or invalid required PAR request is rejected with `invalid_request`. The server MUST NOT redirect an error unless it has validated the redirect under the applicable client metadata and transaction. Front-channel carriage of the statement or instance-attestation artifacts is rejected with `invalid_request`. Details of configured issuers, attesters, evidence policies, and tenant restrictions MUST NOT be exposed in error descriptions to unauthenticated callers, as {{STATEMENT}} requires of its issuer trust configuration.

# Examples {#examples}

The following examples are non-normative. JWT signatures and opaque protocol values are abbreviated. Form bodies are wrapped for readability. The metadata digest below is computed over the UTF-8 bytes of the displayed client document, using LF line endings, two-space indentation, and one final LF after the closing brace.

## Reviewed Client Metadata

~~~ json
{
  "client_id": "https://cli.example/client.json",
  "client_name": "Example CLI",
  "software_statement_instance_binding": true,
  "token_endpoint_auth_method": "attest_jwt_client_auth_dpop",
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "redirect_uris": ["http://127.0.0.1/callback"],
  "scope": "projects.read",
  "software_statements_uri": "https://cli.example/reviews.json"
}
~~~

The server retrieves a statement for this identifier and these exact bytes. The document's port-free callback entry permits a transaction-specific port under {{loopback}}.

## Instance Attestation

The decoded header and payload of the attestation are:

~~~ json
{
  "typ": "cimd-instance-attestation+jwt",
  "alg": "ES256",
  "kid": "attester-2026-10"
}
~~~

~~~ json
{
  "iss": "https://attester.example",
  "sub": "https://cli.example/client.json",
  "iat": 1792886400,
  "exp": 1792886700,
  "cimd_digest": "SvGs0Mf5jHoShvhZQdHsFxivqUgzylxKr5ppf4Kt-W0",
  "cimd_instance_policy": "https://attester.example/policy/cli/v1",
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "_bquDzEXibtIvC6ZswQz5Lwxb2dtGTktsxQKJYzCn5M",
      "y": "tt4th1jwXV4P_YUWtGNzYV61atziwiJsm9XOkQqqH5M"
    }
  }
}
~~~

The policy URI denotes agreed evidence requirements. The server does not fetch it to decide whether to trust the attester. The document digest must equal the software statement's digest. The public key represents one installation; it is absent from the shared document.

## PAR and Redemption

After obtaining a DPoP nonce, the CLI submits:

~~~ http
POST /par HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
OAuth-Client-Attestation: <instance-attestation>
DPoP: <proof-for-PAR-with-server-nonce>

client_id=https%3A%2F%2Fcli.example%2Fclient.json
&response_type=code
&redirect_uri=http%3A%2F%2F127.0.0.1%3A51004%2Fcallback
&scope=projects.read
&code_challenge=<S256-challenge>
&code_challenge_method=S256
&state=<transaction-state>
~~~

Here the server pulls the software statement. It binds the accepted request, key, attester, policy, and tenant to its returned `request_uri`. After authorization in the browser, the CLI receives a code at `http://127.0.0.1:51004/callback` and validates the response issuer and state. It redeems the code as follows:

~~~ http
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
OAuth-Client-Attestation: <current-instance-attestation>
DPoP: <new-proof-for-token-endpoint-with-server-nonce>

grant_type=authorization_code
&client_id=https%3A%2F%2Fcli.example%2Fclient.json
&code=<authorization-code>
&redirect_uri=http%3A%2F%2F127.0.0.1%3A51004%2Fcallback
&code_verifier=<PKCE-verifier>
~~~

The server verifies the original endorsed key and all continuation checks. It does not accept another installation's valid attestation for this code.

## Refresh and Replacement

~~~ http
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
OAuth-Client-Attestation: <current-instance-attestation>
DPoP: <new-proof-for-token-endpoint-with-server-nonce>

grant_type=refresh_token
&client_id=https%3A%2F%2Fcli.example%2Fclient.json
&refresh_token=<refresh-token>
~~~

If the review has expired, the server pulls a current replacement as core permits for a grant established from a pulled statement. If the replacement names changed metadata, the attestation must name that same new digest while retaining the recorded key, attester, and policy. Otherwise the refresh fails without changing the binding.

# Security Considerations {#security-considerations}

The security requirements of {{STATEMENT}}, {{ABCA}}, {{CIMD}}, {{RFC8252}}, and {{RFC9700}} apply. This section describes the additional trust boundary introduced by the extension.

## Software Evidence and Key Use {#software-evidence}

A valid signature proves that an accepted attester made an assertion; it does not improve the evidence behind it. An attester accepting self-reported executable hashes would let a copied statement and an arbitrary key satisfy the binding. The evidence and mapping requirements in {{trust}} prevent that substitution only when the deployment actually enforces them.

A key's non-exportability does not prove which process can use it. An acceptable evidence policy therefore requires key storage and any signing broker to restrict use to the software instance the policy covers ({{trust}}). A general signing service reachable by arbitrary local processes would let unreviewed software authenticate through an approved installation. A signing broker therefore exposes constrained operations rather than arbitrary signing: it produces proofs for the authorization server only within its own processing of an OAuth transaction, and proofs for resource requests only for resource endpoints the caller is authorized to reach and only with that caller's own access token. Remote attestation of a device without an application-to-key binding has the same limitation.

This profile cannot ensure benign user intent, prevent an approved application from being used as a proxy, or preserve an assurance after compromise of the platform on which its evidence depends. A server MUST present assurances no stronger than the accepted policy supports. Plugins, scripts, dynamic libraries, debuggers, and injected code can change the effective software identity and require explicit treatment in that policy.

## Loopback and Transaction Theft

An attacker can attempt to occupy a loopback port, intercept a response, or initiate its own authorization transaction using the published client identifier. PKCE and proof of the endorsed key protect redemption of an intercepted code. Admission of a new transaction additionally requires an acceptable instance attestation; PKCE or an unendorsed DPoP key alone supplies no software identity.

The client MUST keep its verifier and transaction state confidential and unpredictable as their respective specifications require. A launcher that handles authorization for another process has to bind the authenticated caller, intended authorization server, callback, verifier, and resulting grant state to the same local session. A reusable signing capability handed to that caller without equivalent controls is insufficient.

## Downgrade Prevention {#downgrade}

A server requiring reviewed software for a client namespace or tenant MUST enforce that requirement even when the requester omits every review artifact or presents an earlier document. A server requiring this binding MUST reject attempts to open a grant under an alternative authentication method, through a direct authorization request, or through another grant type. These requirements MUST be based on server policy, not inferred only from artifacts a requester can withhold.

The server MUST enforce the reviewed binding selection on every new admission under that document. A deployment permitting unrelated unreviewed clients MAY continue to do so under its own policy, but a failed request under this profile MUST NOT silently become an unreviewed request. Any independently admitted unreviewed client carries none of this extension's assurance.

Existing grants retain their recorded binding. Removing metadata, changing an attester or policy, or presenting a fresh proof under a new key cannot relax it. A server implementing only core continues to reject these clients where review is required.

## Artifact Substitution and Key Separation

The distinct attestation type, digest binding, subject equality, and fixed establishment values prevent a generic attestation or one for another reviewed document from being substituted. Implementations MUST apply mutually exclusive validation rules for software statements and instance attestations. Software-statement signing keys retain core's separation requirements and MUST NOT sign instance attestations, even if one organization performs both roles.

Attester authorization is scoped independently to client namespaces and tenants. An over-broad attester can undermine the binding for every namespace it is allowed to endorse. A review issuer and an attester colluding or both being compromised can make false assertions within the authority the server has assigned them.

## Freshness and Availability

Short effective lifetimes limit stale evidence but make attester availability important. A fresh proof demonstrates current access to a key, not that software was re-evaluated at proof time. Attestation renewal therefore follows the evidence policy and cannot consist solely of extending `exp` on old evidence.

Servers MUST enforce finite evidence and trust-data reuse bounds. A failure to retrieve metadata, reviews, or trust updates MUST NOT extend a recorded expiry or remove a refusal. An access token cannot outlive the attestation accepted when it was issued ({{redemption}}), but it can outlive a review withdrawn or attester trust removed after issuance; deployments requiring prompt enforcement need short token lifetimes or independent token revocation.

Metadata digest binding can require coordinated publication of a new document, review, and attestation. During an incomplete transition a request fails. A server MUST NOT substitute semantically similar JSON, omit members from the digest, or continue with an older digest beyond permitted cache reuse merely to avoid that failure.

## Resource Exhaustion

Attestations do not remove the need for bounded document retrieval, statement selection, proof replay caches, and PAR storage. Servers SHOULD limit pending PAR transactions separately from core's limits on establishments. Rate limits SHOULD account for client, instance key, tenant, and authenticated resource owner as available. An attester SHOULD limit enrollment and renewal so one approved installation cannot obtain unlimited keys to evade per-key controls.

# Privacy Considerations {#privacy}

An instance key reused at multiple servers can correlate installations across them. Clients SHOULD use separate instance keys for separate server relationships where the evidence mechanism permits, and SHOULD avoid adding stable installation identifiers unless needed by the deployment. This profile does not require a globally stable instance identifier or an attestation audience.

An attester may learn the target server from enrollment traffic, key-scoping choices, or timing even when no audience is carried. Deployments SHOULD minimize evidence and identifier disclosure to both attesters and authorization servers. An evidence-policy identifier can reveal enterprise management practices; it should identify policy semantics without embedding user or device identifiers.

Instance attestations and proofs MUST NOT be placed in browser URLs. Operators SHOULD avoid logging complete attestations, verifiers, codes, and refresh tokens. Audit records can retain the trust decisions, policy identifier, digest, and appropriately protected key identifiers without retaining credentials.

# IANA Considerations {#iana}

## OAuth Dynamic Client Registration Metadata Registry

This specification requests registration of the following value in the "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}:

Client Metadata Name:
: `software_statement_instance_binding`

Client Metadata Description:
: Boolean selecting endorsed instance-key binding for runtime admission using a CIMD software statement. Defined for Client ID Metadata Documents; it does not enable registration under this specification.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{client-metadata}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following value in the "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_instance_binding_supported`

Metadata Description:
: Boolean indicating support for endorsed instance-key binding of CIMD software statements.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{server-metadata}}

## JSON Web Token Claims Registry

This specification requests registration of the following claim in the "JSON Web Token Claims" registry established by {{RFC7519}}:

Claim Name:
: `cimd_instance_policy`

Claim Description:
: Absolute URI identifying the evidence policy under which an attester associates an instance key with software identified by a Client ID Metadata Document.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{attestation}}

This specification also requests that the `cimd_digest` registration requested by {{STATEMENT}} list this specification ({{attestation}}) as an additional reference. An instance attestation carries the same digest that a software statement carries.

The `iss`, `sub`, `iat`, `exp`, and `cnf` claims are defined elsewhere and are not re-registered here.

## OAuth Extensions Error Registry

For completeness of the registry, this specification requests that IANA add this specification as a reference for each of the following errors and extend its usage locations to include the pushed authorization request error response, which can already carry it under {{RFC9126}}.

Error Name:
: `use_fresh_attestation`

Existing Registration:
: {{ABCA}}

Error Usage Location:
: pushed authorization request error response, in addition to the locations already registered

Change Controller:
: IETF

Specification Document(s):
: {{ABCA}}, this specification ({{errors}})

Error Name:
: `invalid_client_attestation`

Existing Registration:
: {{ABCA}}

Error Usage Location:
: pushed authorization request error response, in addition to the locations already registered

Change Controller:
: IETF

Specification Document(s):
: {{ABCA}}, this specification ({{errors}})

Error Name:
: `use_dpop_nonce`

Existing Registration:
: {{RFC9449}}

Error Usage Location:
: pushed authorization request error response, in addition to the locations already registered

Change Controller:
: IETF

Specification Document(s):
: {{RFC9449}}, this specification ({{errors}})

Error Name:
: `invalid_dpop_proof`

Existing Registration:
: {{RFC9449}}

Error Usage Location:
: pushed authorization request error response, in addition to the locations already registered

Change Controller:
: IETF

Specification Document(s):
: {{RFC9449}}, this specification ({{errors}})

## Media Type Registration

This specification requests registration of the following media type under {{RFC6838}}:

Type name:
: application

Subtype name:
: cimd-instance-attestation+jwt

Required parameters:
: n/a

Optional parameters:
: n/a

Encoding considerations:
: 8bit. Values are JWTs in compact serialization.

Security considerations:
: See {{security-considerations}} of this specification and the security considerations of {{ABCA}} and {{RFC7519}}.

Interoperability considerations:
: n/a

Published specification:
: This specification

Applications that use this media type:
: Native OAuth clients, client attesters, and authorization servers using CIMD software statements with endorsed instance keys.

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

This specification uses the authentication method, HTTP fields, and error codes of {{ABCA}} without registering replacements.

--- back

# Managed CLI Deployment {#launcher}

This appendix is non-normative. A managed desktop can implement the evidence boundary with a trusted launcher or enterprise agent and an enterprise attestation service. Other mechanisms are possible if they meet {{trust}}.

The enterprise enrolls the launcher through an authenticated management channel. It records a manifest, authorized by the controller of the CIMD URL's origin or its delegate, associating a signed package or exact artifact with that CIMD identifier, and approves it for the applicable evidence policy. The launcher uses platform facilities to authenticate the actual calling process and validate that association. It does not trust a process's own claim about its executable path or hash.

The launcher creates or provisions the instance key in protected storage and obtains an attestation through an authenticated exchange with the enterprise service. That service verifies the evidence, the binding to the key, and the document digest before issuing the attestation. The service's signing key is distinct from the instance key and is accepted through the authorization server's attester trust configuration.

For each authorization transaction, the launcher performs the OAuth exchange on behalf of the verified caller or restricts key operations with equivalent controls. For resource requests, it produces DPoP proofs only for resource endpoints the caller is authorized to reach and only with that caller's own access token. It never signs arbitrary input, and produces a proof for the authorization server only within its own transaction processing. The local session binds caller identity, server, callback, PKCE verifier, and resulting grant. It rechecks caller authorization when the key is used, including refresh, and protects returned tokens from other processes.

An implementation must address changes between checking software and using the key, dynamic code loading, and the ability of an approved executable to proxy requests for others. Platforms without reliable process identity and key-use isolation cannot obtain that assurance merely by running the launcher with more privileges. A script-based CLI may require an execution environment whose policy covers its interpreter, script, and dependencies, or a narrower assurance than reviewed executable identity.

# Verification Scenarios

This appendix is non-normative. An implementation exercise should include the following cases in addition to successful admission and refresh:

* A copied statement with an unendorsed key fails before creating an establishment.
* A copied attestation with a different proof key fails authentication.
* A valid attestation for another client, digest, tenant authorization, or policy fails its corresponding binding check.
* A loopback code intercepted by another process cannot be redeemed without both the verifier and recorded key.
* A repeated proof and a proof intended for another endpoint are rejected.
* A nonce retry neither consumes grant state nor advances the `iat` floor.
* A refresh with a renewed attestation for the original tuple succeeds when the review remains current.
* A metadata replacement without a matching current attestation fails without changing the establishment.
* A new key, attester, policy, or tenant cannot take over an existing grant.
* Withdrawal or removal of trust prevents continuation despite a fresh proof.
* Omitting attestations or using direct authorization cannot bypass a server requirement for this binding.
* An unapproved local process cannot obtain usable signatures or tokens through the trusted launcher.

# Acknowledgments
{:numbered="false"}

This specification develops the endorsed-key extension point in {{STATEMENT}} and composes it with attestation-based client authentication. The initial deployment question was how a command-line application could retain loopback callbacks while demonstrating an association with reviewed software.
