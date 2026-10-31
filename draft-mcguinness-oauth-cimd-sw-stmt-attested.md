---
title: "Attested Instance Keys for Client ID Metadata Document Software Statements"
abbrev: oauth-cimd-sw-stmt-attested
docname: draft-mcguinness-oauth-cimd-sw-stmt-attested-latest
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
  RFC9126:
  RFC9207:
  RFC9449:
  RFC9700:
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  ABCA: I-D.ietf-oauth-attestation-based-client-auth
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "Client ID Metadata Document Software Statement"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-00
  LIFECYCLE:
    target: https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt/draft-mcguinness-oauth-cimd-sw-stmt-lifecycle.html
    title: "Client ID Metadata Document Software Statement Lifecycle"
    author:
      - fullname: Karl McGuinness
    date: 2026-10

informative:
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "Client ID Metadata Document Software Statement Registration"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-registration-00
  APPLE-ENCLAVE:
    target: https://developer.apple.com/documentation/security/protecting-keys-with-the-secure-enclave
    title: "Protecting keys with the Secure Enclave"
    author:
      - org: Apple Inc.
  APPLE-KEYCHAIN:
    target: https://developer.apple.com/documentation/technotes/tn3137-on-mac-keychains
    title: "TN3137: On Mac keychain APIs and implementations"
    author:
      - org: Apple Inc.
  APPLE-XPC:
    target: https://developer.apple.com/documentation/foundation/nsxpcconnection/setcodesigningrequirement(_:)
    title: "setCodeSigningRequirement(_:)"
    author:
      - org: Apple Inc.
  APPLE-RUNTIME:
    target: https://developer.apple.com/documentation/security/hardened-runtime
    title: "Hardened Runtime"
    author:
      - org: Apple Inc.
  APPLE-LIBVAL:
    target: https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.disable-library-validation
    title: "Disable Library Validation Entitlement"
    author:
      - org: Apple Inc.
  APPLE-DYLD:
    target: https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.allow-dyld-environment-variables
    title: "Allow DYLD environment variables entitlement"
    author:
      - org: Apple Inc.
  APPLE-DEBUGGER:
    target: https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.debugger
    title: "Debugging tool entitlement"
    author:
      - org: Apple Inc.
  APPLE-MDA:
    target: https://support.apple.com/guide/deployment/managed-device-attestation-dep28afbde6a/web
    title: "Managed Device Attestation for Apple devices"
    author:
      - org: Apple Inc.
  APPLE-APPATTEST:
    target: https://developer.apple.com/documentation/devicecheck/establishing-your-app-s-integrity
    title: "Establishing your app's integrity"
    author:
      - org: Apple Inc.
  APPLE-APPCONFIG:
    target: https://developer.apple.com/documentation/devicemanagement/appmanaged
    title: "AppManaged"
    author:
      - org: Apple Inc.
  APPLE-WWDC26:
    target: https://developer.apple.com/videos/play/wwdc2026/201/
    title: "Secure your apps with App Attest"
    author:
      - org: Apple Inc.
    date: 2026
    seriesinfo:
      WWDC26: Session 201

--- abstract

A software statement for a Client ID Metadata Document (CIMD) binds a review of client software to the exact bytes of that document. It does not, by itself, bind an installation of that software to a key. This specification defines an instance-binding extension in which an independently trusted client attester attests an installation's key for the reviewed document. The client proves that key through attestation-based client authentication when opening and continuing an authorization code grant. When an attester can verify which software controls an installation's key, as on managed devices, this specification permits native applications, including command-line clients, to use loopback redirects while satisfying a policy requiring reviewed software. The specification defines attestation and metadata requirements, processing at the pushed authorization request and token endpoints, and the exceptions to the base software statement rules needed for attested keys.

--- middle

# Introduction

A native application's installations cannot safely authenticate by a private key distributed in every copy of the software. Each installation can generate its own key, but possession of that key establishes continuity, not the identity of the software using it. A copied Client ID Metadata Document (CIMD) URL {{CIMD}} and software statement can be presented with an attacker's own key.

{{STATEMENT}} therefore treats a presentation under a document containing a redirection URI that another application on the same device could claim as review-only: the server evaluates the statement but admits nothing on it, and such a presentation cannot satisfy a policy requiring reviewed software. This excludes loopback redirects, even though they are an established response-delivery mechanism for native applications under {{RFC8252}}.

Existing mechanisms do not provide reviewed-software admission for loopback clients. Loopback redirects with Proof Key for Code Exchange (PKCE) {{RFC7636}} protect an authorization code from interception, but any application can start a transaction with a native client's identifier, the client impersonation that Section 8.6 of {{RFC8252}} describes. Attestation-based client authentication {{ABCA}} alone shows that an installation holds a key an attester vouches for, not that the installation runs the software a statement issuer approved for a particular document. The public-client binding of {{STATEMENT}} relies on redirection URIs no other application can claim, which loopback URIs are not.

This specification defines an alternative binding made of three elements, which stay separate even when one organization operates both issuing roles:

* The software statement records approval of the software, as its publisher identifies it ({{trust}}), and its exact metadata.
* An instance attestation, from an independently trusted client attester under an evidence policy the authorization server accepts, associates an installation's key with that software and metadata.
* A proof under attestation-based client authentication demonstrates possession of that key.

The callback delivers the authorization response; it supplies no evidence of the presenter's software identity.

This specification serves authorization code flows using pushed authorization requests (PAR) {{RFC9126}} and refresh of the resulting grants. It defines neither registration nor new token-endpoint presentations for other grants. {{REGISTRATION}} is unchanged: it rejects loopback redirects, so a document selecting this profile is never registered, whatever the `statement_uses` claim of a statement over it permits. Neither {{STATEMENT}} nor {{REGISTRATION}} depends on this specification.

The binding is only as strong as the attester's evidence that the software, not merely the device or the user, controls the key ({{software-evidence}}). Such evidence is available today mainly on managed devices, through an enterprise agent or launcher ({{launcher}}). On an unmanaged desktop, or for an interpreted command-line client, an attester typically cannot meet the requirements of {{trust}}, and the client remains review-only under {{STATEMENT}}.

## Protocol Overview

The following non-normative sequence summarizes admission under this profile:

1. A publisher serves a CIMD that selects this profile and declares attestation-based client authentication and loopback redirects. A statement issuer signs a software statement over its exact bytes.
2. An installation obtains an attestation from an accepted attester. The attestation names the client identifier URL, metadata digest, instance key, and evidence policy.
3. The installation sends a pushed authorization request with PKCE, the attestation, and a Demonstrating Proof of Possession (DPoP) proof. It supplies the software statement or lets the server pull it.
4. The server validates both trust decisions (in the statement issuer and in the attester) and their bindings before returning a `request_uri`.
5. The user authorizes in an external browser. The code returns over loopback and is redeemed with the PKCE verifier and the attested key.
6. Refresh requires the proven key, a current instance attestation, and a current software statement. A different key requires a new grant.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology from {{RFC6749}}, {{RFC7591}}, {{CIMD}}, and {{ABCA}}, the terms defined in Section 2 of {{STATEMENT}}, and the terms defined in {{LIFECYCLE}}. An installation is a Client Instance as defined by {{ABCA}}.

This specification depends on {{LIFECYCLE}}, because continuous review and replacement are part of its security model: every refresh requires a current statement, refusal records are enforced at code redemption and refresh, and a replacement statement keeps a grant in step with a later review ({{redemption}}, {{refresh}}).

This specification defines the following terms:

Instance Attestation:
: A Client Attestation JWT under {{ABCA}}, profiled by {{attestation}}. It records an attester's association of a key with a client identifier URL and metadata digest under a named evidence policy. An instance attestation is current while its `exp` has not passed and its attester and evidence policy remain accepted.

Evidence Policy:
: A versioned set of requirements for establishing the association between software, an instance key, and reviewed metadata. It covers evidence collection, verification, freshness, and protection of key use.

Attested Key:
: The key an instance attestation's `cnf` claim names.

The software statement's issuer and the client attester are separate roles. One entity can perform both, but neither role implies trust in the other.

# Scope and Profile Selection {#selection}

An authorization server implementing this specification MUST support PAR, PKCE with `S256` {{RFC7636}}, DPoP {{RFC9449}}, and the `attest_jwt_client_auth_dpop` method of {{ABCA}}, which uses a DPoP proof as the proof of possession (Sections 5.2 and 7.3 of {{ABCA}}). Clients MUST use that method at the PAR endpoint and at every token request under this specification. This profile does not support the Client Attestation PoP JWT (Section 5.1 of {{ABCA}}).

Each installation holds its own credential, the attested key, so a client under this profile is a confidential client although it is a native application, as Section 8.4 of {{RFC8252}} contemplates for per-instance credentials. A presentation processed under {{par}} whose redirection URIs meet {{loopback}} is not review-only (Section 5.6.3 of {{STATEMENT}}).

The client selects this profile by publishing the `software_statement_instance_binding` client metadata member with the value `true` in its reviewed document ({{client-metadata}}). A server creates a new establishment under the profile only if the server supports this specification, the reviewed document selects the profile, and all admission checks succeed.

Once an establishment is created, its recorded binding selects the profile for every continuation. Selection MUST NOT depend on whether a continuation happens to include an attestation or on a newly fetched document omitting the selection member. Such omissions fail the request; they do not change the grant's binding.

This specification permits only the authorization code grant opened through PAR, and refresh of that grant. It does not authorize private-use URI schemes, `localhost` hostnames, non-loopback redirects, implicit responses, device authorization, client credentials, or assertion grants. Other specifications can define additional bindings without changing the behavior of this profile.

## Required Binding {#required-binding}

A server requiring reviewed software for a client namespace or tenant MUST enforce that requirement even when the requester omits every review artifact or presents a statement over an earlier document. A server requiring this binding MUST reject attempts to open a grant under an alternative authentication method, through a direct authorization request, or through another grant type. These requirements MUST be based on server policy, not inferred only from artifacts a requester can withhold.

The server MUST enforce the reviewed binding selection on every new admission under a reviewed document that selects this profile. A deployment permitting unrelated unreviewed clients can continue to do so under its own policy, but a failed request under this profile is rejected rather than admitted under that policy ({{par}}). Any independently admitted unreviewed client carries none of the assurance of this specification.

# Client Metadata {#client-metadata}

This specification defines the following member of a CIMD:

`software_statement_instance_binding`:
: OPTIONAL. Boolean value. When `true`, the client requires this profile for admissions under this document. If absent or `false`, the document does not select this profile. A value of any other type is invalid.

A document selecting this profile MUST meet all of the following requirements:

* Its `client_id` is its HTTPS document URL, as required by {{CIMD}}.
* Its `token_endpoint_auth_method` is the single string `attest_jwt_client_auth_dpop`. No alternative authentication method is offered under this profile.
* Its `grant_types` member explicitly contains `authorization_code` and MAY also contain `refresh_token`. It contains no other grant type.
* Its `response_types` explicitly contains only `code`.
* Its `redirect_uris` is a nonempty array, every member of which meets {{loopback}}.
* It contains neither the `jwks` nor the `jwks_uri` member. Instance keys are authorized only through {{attestation-validation}}, not by document key material.

The document MAY contain the `software_statements_uri` member as defined by {{STATEMENT}}. Metadata governing requested access, such as `scope`, remains subject to that specification. Every member, including the binding selection, is covered by the statement's digest.

## Loopback Redirects {#loopback}

Every reviewed redirection URI MUST use `http`, have the host exactly `127.0.0.1` or `[::1]`, and contain an absolute path. User information and fragments MUST NOT be present. A reviewed entry MUST NOT contain a port: the client uses the port of its listener, which Section 7.3 of {{RFC8252}} lets the server accept. The document MAY list both address forms.

The server MUST compare a requested redirect with a reviewed entry using exact string matching, except for the port variation permitted by Section 7.3 of {{RFC8252}}. No other component can vary: the address form, path, and any query remain fixed. In particular, a wildcard path, a hostname resolving to loopback, an alternative numerical spelling of the address, and a different loopback address do not match.

The client MUST bind its listener only to the selected loopback interface, open it before sending the pushed authorization request, and close it when the transaction completes or is abandoned. It can obtain an available ephemeral port from the operating system, as Section 7.3 of {{RFC8252}} anticipates. The authorization code is delivered to that listener; the authorization server MUST NOT fetch the redirection URI to test the listener.

# Attester Trust and Evidence {#trust}

An authorization server MUST establish attester trust independently of the request. For each accepted attester it MUST configure:

* The exact attester `iss` identifier and how the attester's verification keys are resolved. Section 10.8 of {{ABCA}} recommends that a profile fix key resolution. Keys are resolved by one of the following:
  * An `x5c` certificate chain ({{RFC7515}}) validated to a configured trust anchor, together with a configured mapping from the certificate to this attester, such as an exact subject or subject alternative name. Sharing a trust anchor with another attester does not authorize a key for this attester's `iss`.
  * A `kid` matched against a JWK Set configured for this attester. A `jku` is followed only if it names that configured JWK Set.
* Accepted asymmetric signing algorithms.
* Client identifier namespaces for which the attester can issue instance attestations.
* Accepted evidence-policy identifiers and, for each, a finite maximum evidence age.
* Tenant restrictions and any requirements for a particular attester or policy within a tenant.
* Clock-skew and evidence-freshness requirements and any local denial rules for instance keys.

The server MUST use the tenant determined for the grant by its own trusted processing, as {{STATEMENT}} requires. A client-supplied tenant label is not sufficient. The server MUST NOT infer attester trust from a statement issuer's signature, from the client document, or from a key-location value in the attestation. An `iss`, a `jku` or `x5u` header parameter ({{RFC7515}}), or a similar value in an unverified JWT MUST NOT cause unconfigured key discovery. Retrieval from a configured key source is subject to the retrieval protections that {{STATEMENT}} applies to the retrievals it causes, and the server MUST NOT reuse retrieved trust data, including an attester's key set, beyond a finite maximum reuse period of its own choosing.

The server cannot observe how an attester reached its decision, so it relies on the evidence policy. An evidence policy is acceptable for reviewed-software admission only if it meets all of the following requirements:

* Before issuing each instance attestation, the attester verifies control of the private instance key through an authenticated, replay-resistant exchange bound to the submitted public key.
* The attester evaluates evidence about the software instance and protection of key use, and sets each attestation's `exp` no later than the time that evidence was collected plus the policy's maximum evidence age. The attester rounds or randomizes `exp` (Section 11.1 of {{ABCA}}) only toward an earlier time, a constraint this profile adds. A previous attestation is not evidence: a renewal rests on new evidence under the same bound.
* The attester establishes an authenticated association between the software identity and the client identifier URL, retrieves and validates the document under {{CIMD}}, and calculates its digest under {{STATEMENT}}.
* The attester confirms that the artifact or package identity established by the evidence is covered by that association for the document version the attestation names. The association is authorized by whoever controls the origin of the client identifier URL, or by an authority that controller has explicitly delegated, for example through a manifest served from that origin or signed under that controller's or delegate's authority. An enterprise inventory can authenticate or record such an authorization; listing a package and someone else's client identifier URL together does not create one. A URL, file path, digest, or publisher name asserted only by the requesting process does not suffice. The metadata digest is not an executable digest.
* The policy defines what identifies the software, how evidence authenticates that identity and the instance key, how use of the key is restricted to the software instance the evidence covers, a finite maximum evidence age, and the treatment of updates, scripts, plugins, and other executable extensions.
* The policy's identifier denotes immutable policy semantics: a material policy change requires a new identifier.
* The policy establishes an association between the software the statement covers and control of the attested key. User approval of a key, a publisher account login, possession of a shared embedded secret, or device integrity alone does not establish that association.

A server MUST NOT accept for this profile an evidence policy that does not meet these requirements. A server can use weaker evidence for other purposes but MUST NOT report it as satisfying this profile.

A statement approves the document and the software that its publisher (the controller of the document URL's origin, or a delegate) associates with it, not a particular executable. The publisher decides which releases that association covers, under the evidence policy's treatment of updates, so a new release with unchanged metadata needs no new statement. This profile cannot enforce an issuer's approval of particular releases.

Attestation acquisition and platform evidence formats are out of scope. They require an interoperable agreement between the installation and its attester; the wire format below does not by itself supply a software-verification mechanism. {{launcher}} gives an implementation model and its prerequisites.

# Instance Attestation {#attestation}

An instance attestation is a Client Attestation JWT (Section 4 of {{ABCA}}) with the following additional requirements. Section 4 of {{ABCA}} permits additional claims, and Section 13 of {{ABCA}} permits a profile its own `typ`. Together with the document's selection of this profile ({{selection}}), its type tells a server that this profile applies. Its `typ` JOSE header parameter MUST have the value `cimd-instance-attestation+jwt`, which is the media type of {{media-type}} without its `application/` prefix. It MUST be digitally signed using an accepted asymmetric algorithm. A MAC or `none` MUST NOT be accepted. Its header and claims MUST contain no duplicate member names. Validation MUST apply {{RFC8725}}.

The attestation contains the following claims. The `sub`, `iat`, `exp`, and `cnf` claims also retain the requirements of Section 4 of {{ABCA}}.

`iss`:
: REQUIRED. The attester's identifier, an HTTPS URL without a fragment. It is matched exactly against the configured attester `iss` identifier ({{trust}}). URL normalization MUST NOT be used to broaden that match.

`sub`:
: REQUIRED. It MUST be the exact client identifier URL.

`aud`:
: OPTIONAL. If present, the receiving server MUST validate it under {{RFC7519}} against its own issuer identifier ({{RFC8414}}). The proof targets a particular endpoint ({{proof}}). The attestation can be used at more than one authorization server.

`iat`:
: REQUIRED. It is not an evidence timestamp, and an attester can round or randomize it as Section 11.1 of {{ABCA}} recommends.

`exp`:
: REQUIRED. It MUST be later than the `iat` claim. It bounds the age of the evidence behind the attestation, as the accepted evidence policy requires ({{trust}}).

`cnf`:
: REQUIRED. The attested key ({{RFC7800}}), in the form Section 4 of {{ABCA}} requires.

`cimd_digest`:
: REQUIRED. The metadata digest defined by {{STATEMENT}}, computed over the document the attester evaluated. This claim has the same representation and comparison rules as in a software statement. Its use here records the attester's software-to-metadata association, not a second software review.

`cimd_instance_policy`:
: REQUIRED. A string containing an absolute URI identifying the evidence policy under which the attester attested the instance key. The URI is an identifier, not a retrieval instruction. It MUST be compared exactly with an accepted policy identifier.

A server MUST reject an attestation whose `exp` claim has passed or is later than the current time plus the maximum evidence age it accepts for the attester and evidence policy. It MUST NOT reject an attestation only because its `iat` claim was rounded or randomized. No field supplied by the client can extend these bounds.

An attestation is not tied to a statement's `jti`, so a statement renewed over unchanged bytes need not cause re-attestation.

## Validation {#attestation-validation}

At each pushed authorization request, code redemption, and refresh, the authorization server MUST validate the instance attestation and its proof. It MUST verify all of the following before accepting the binding:

* The attestation meets {{attestation}} and the applicable {{ABCA}} validation requirements.
* The signature verifies with a key authorized for the exact `iss`, for this artifact type and accepted algorithm.
* The attester and policy are currently accepted for this client identifier and tenant, and the key is not denied by local policy.
* The request's `client_id`, the software statement's `sub`, and the attestation's `sub` are exactly equal.
* The attestation's `cimd_digest`, the applicable software statement's `cimd_digest`, and the digest of the reviewed document are equal.
* The attested key and the key of the DPoP proof are equal under the key-comparison rules of Section 7.3 of {{ABCA}}.

On continuation, the proven key and the recorded attester, policy, and tenant constraints of {{establishment}} also apply. The server MUST NOT treat an attestation for another digest as permission to update metadata without the software statement replacement required by {{refresh}}.

A server MUST NOT accept a software statement as an instance attestation, or an instance attestation as a software statement. Software statement signing keys retain the separation requirements of {{STATEMENT}}, so they never sign instance attestations, even if one organization performs both roles.

## Proof of Possession {#proof}

The client sends the instance attestation in the `OAuth-Client-Attestation` HTTP header field and uses DPoP combined mode under Section 7.3 of {{ABCA}}. It MUST NOT also send the `OAuth-Client-Attestation-PoP` HTTP header field, and the server MUST reject a request under this profile that does. A `dpop_jkt` parameter alone is not a proof. The authentication method and its validation remain those defined by {{ABCA}}; this profile supplies the attestation's type and additional bindings.

The authorization server MUST require a server-issued DPoP nonce at the PAR endpoint and on every token request under this profile. It MUST enforce endpoint and method binding and reject replay of a proof within its acceptance window under {{RFC9449}}. Nonce challenges and retries use the `use_dpop_nonce` error code and the `DPoP-Nonce` HTTP header field. They do not consume a code, rotate a refresh token, create an establishment, or advance the `iat` floor. A request rejected before the instance is authenticated, or rejected with `use_fresh_attestation` or `temporarily_unavailable`, has none of these effects either, so the client can retry with a fresh attestation.

If the pushed authorization request also carries the `dpop_jkt` parameter, Section 10.1 of {{RFC9449}} requires the server to reject the request unless the `dpop_jkt` value is the JWK thumbprint ({{RFC7638}}) of the DPoP proof's key, which {{attestation-validation}} makes the attested key. A client MUST generate a new proof for each request, including a retry. TLS protects request contents. DPoP does not sign the request body, and implementations MUST NOT treat a valid proof as a signature over the redirection URI, PKCE challenge, or statement.

# Runtime Admission {#admission}

This section specifies processing at the pushed authorization request, authorization, and token endpoints when a grant is opened under this profile.

## Pushed Authorization Request {#par}

The client MUST start every grant under this profile at the PAR endpoint. Its request MUST include `client_id`, `response_type=code`, `redirect_uri`, and a PKCE `code_challenge` with `code_challenge_method=S256`. It MAY include the `state` parameter: the server requires PKCE ({{server-metadata}}), so the client can rely on PKCE for CSRF protection, as Section 2.1 of {{RFC9700}} permits. It MUST authenticate under {{proof}}. A request using a Request Object MUST meet {{RFC9126}} and MUST NOT provide conflicting parameter values; the following checks apply to the effective authorization request.

The client presents its software statement using `software_statement` as defined by {{STATEMENT}}, or the server obtains a statement through that specification's pulled-statement mechanism.

Before returning a successful PAR response, the server MUST:

1. Validate request syntax and the client identifier, resolve the applicable tenant, and authenticate the instance under {{trust}}, {{attestation}}, and {{proof}}. Checks needing no client-controlled retrieval precede such retrieval.
2. Validate the selected statement (Section 3.2 of {{STATEMENT}}) and apply the refusal records and `iat` floor (Sections 3 and 4 of {{LIFECYCLE}}). If statements are pulled, step 3 resolves the document first, since the document names where statements are published, and the retrieval and selection rules of {{STATEMENT}} apply.
3. Resolve the CIMD (Section 5.3 of {{STATEMENT}}), perform the digest and metadata checks of Section 5.2 of {{STATEMENT}}, enforce {{client-metadata}}, and complete the cross-artifact checks of {{attestation-validation}}.
4. Validate the requested redirect under {{loopback}}, require PKCE `S256`, and evaluate the request against the reviewed metadata and authorization policy.
5. Create the establishment ({{establishment}}), bind the accepted request to it, and return a PAR response under {{RFC9126}}.

If any required check fails, the server MUST NOT fall back to review-only or unreviewed admission for the same request. A request that carries no instance attestation for a document selecting this profile is an example: the server learns from the document (step 3) that this profile applies, then rejects the request with `invalid_client`. If the instance authenticates (step 1) but no statement is presented or remains after selection, it returns `statement_required`. A failed request MUST NOT advance the `iat` floor.

## Establishment {#establishment}

An establishment records all state required by {{STATEMENT}} and, in addition:

* That this instance-binding profile governs the grant.
* The exact attester identifier and evidence-policy identifier.
* The attester trust decision.

The proven key is the attested key, which is also the DPoP key.

The client identifier, tenant, key, attester, policy, and binding profile MUST remain fixed for the lifetime of the grant. Changing any of them requires a new grant opened through PAR. A fresh attestation that keeps these values can renew the grant's evidence, and a software statement replacement can update the grant's reviewed metadata only as {{refresh}} permits. Neither operation changes these values.

## Browser Authorization {#authorization}

The client MUST use an external user-agent under {{RFC8252}}. The browser authorization request MUST use the `request_uri` parameter from the PAR response and the same `client_id` parameter. Besides the software statement, which Section 5.4.2 of {{STATEMENT}} excludes, it MUST NOT carry the instance attestation or an instance proof. The server MUST use the stored PAR parameters and MUST NOT allow front-channel parameters to change the redirect, PKCE challenge, key binding, or tenant.

When the client's CIMD selects this profile or server policy requires it, the server MUST reject a direct authorization request without a PAR `request_uri` governed by this profile. A statement pulled at the authorization endpoint cannot create an exception to that requirement.

The server authenticates the resource owner and applies authorization policy independently of the software and instance checks. It MUST return the authorization response with the issuer identifier in the `iss` parameter defined by {{RFC9207}}. The client MUST validate that identifier, and the `state` parameter if it sent one, against the transaction before redeeming a code. Callback validation and OAuth security requirements in {{RFC8252}} and {{RFC9700}} continue to apply.

## Authorization Code Redemption {#redemption}

The client sends `grant_type=authorization_code`, the code, the same `client_id` and `redirect_uri`, and the PKCE `code_verifier`. It MUST authenticate with a current instance attestation and a fresh proof under the proven key. It does not send the `software_statement` parameter, which {{STATEMENT}} prohibits on code redemption.

Before issuing tokens, the server MUST verify the code and PKCE binding and every fixed establishment value in {{establishment}}. It MUST re-evaluate both trust decisions, require the recorded statement to be unexpired and not refused, and confirm that the current CIMD still matches its recorded digest and meets {{client-metadata}}. Retrieval and cache reuse use the reuse bounds of {{STATEMENT}}. The attestation MUST match that digest and not be past its `exp`. A server using a statement withdrawal mechanism SHOULD consult it at code redemption, within that mechanism's freshness rules, as Section 5.1 of {{LIFECYCLE}} provides for refresh.

If the metadata changed or the recorded statement needs replacement, the client MUST start a new PAR transaction; code redemption cannot replace the governing review. A raised `iat` floor alone does not invalidate a code already bound to an acceptable recorded statement.

The server MUST issue tokens only within the user's authorization, tenant policy, and reviewed metadata. It MUST issue access tokens, at redemption and on refresh ({{refresh}}), bound to the proven key with a `token_type` of `DPoP` (Section 5 of {{RFC9449}}). The expiry of each access token, whether the token is a JWT or an opaque reference, MUST NOT be later than the `exp` of the instance attestation accepted for the request that obtained it ({{lifetime-practice}}). A resource server that does not accept DPoP-bound access tokens cannot be used under this profile.

# Refresh {#refresh}

Every refresh request MUST carry the original `client_id` and authenticate under {{proof}} with a current instance attestation and a fresh proof under the establishment's proven key. In place of the document-key check of Section 5.7.1 of {{STATEMENT}}, the server MUST complete the checks of {{attestation-validation}}, verify the refresh-token binding, and enforce the fixed values of {{establishment}} before it considers any replacement.

On every refresh, the server MUST require a current software statement and re-evaluate its issuer trust decision. It MUST also verify that the currently served document matches the applicable statement and attestation digests and meets {{client-metadata}}. If the document fails {{client-metadata}}, for example by omitting the `software_statement_instance_binding` member or naming another authentication method, the refresh is rejected with `invalid_grant` without seeking a replacement.

A replacement (Section 5.2 of {{LIFECYCLE}}) is needed when the recorded statement is expired, refused, or no longer matches the served document. In place of the key bullet of step 3 of Section 5.2 of {{LIFECYCLE}}, a current attestation of the proven key, from the recorded attester under the recorded policy, MUST name the replacement's metadata digest.

# Exceptions to Software Statement Processing {#core-exceptions}

The following table summarizes where this profile departs from {{STATEMENT}} and {{LIFECYCLE}}.

| Section | Rule | Under this profile |
| --- | --- | --- |
| 4 | A required issuer configured for software whose presentations are review-only excludes that software from the tenant. | A presentation processed under {{par}} is not review-only, so a current statement from the required issuer can admit the software. |
| 5.1, 5.2 | The statement and the document are checked before the proof. | The instance is authenticated first ({{par}}, step 1). Statement and document checks follow, and checks needing no client-controlled retrieval still precede such retrieval. |
| 5.2, 5.5, 5.6.3 | A request without an acceptable statement can proceed under the policy for unreviewed clients, a review-only presentation proceeds as a client presenting no statement, and a digest mismatch can proceed without the statement. | No fallback: any failed check rejects the request ({{par}}). |
| 5.4.2, 5.5.2 | A pulled statement needs no PAR, and one pulled at the authorization endpoint binds the presenter at code redemption. | The binding is set at the PAR endpoint. An authorization request without a PAR `request_uri` governed by this profile is rejected ({{authorization}}). |
| 5.6, 5.6.1 | The proven key appears in the reviewed document's `jwks` or at its `jwks_uri`. | The proven key is the attested key ({{attestation-validation}}), and the document carries no key material ({{client-metadata}}). |
| 5.6.3 | A redirection URI another application could claim makes a presentation review-only. | Loopback redirects under {{par}} do not make a presentation review-only ({{selection}}). |
| 5.7 | Redemption checks the client identifier and key, not the statement's expiry. | Redemption also requires the recorded statement to be unexpired and unrefused, and the current document to match the recorded digest ({{redemption}}). |
| 5.7, 5.7.1, 13.7 | Redemption and refresh do not re-evaluate issuer trust, so removing an issuer ends no open grant. | Redemption and refresh re-evaluate issuer trust ({{redemption}}, {{refresh}}). |
| 5.7.1 | Refresh checks possession of the proven key and that the currently served document still carries that key. | Refresh verifies a current attestation of the proven key instead of its presence in the document ({{refresh}}). |
| 6.3 | Code redemption is rejected with `invalid_grant` or, at the establishment bound, `invalid_client`. | Redemption can also return `statement_required` ({{error-codes}}). |
| 6.4 | A client not admitted on a statement receives `statement_required`. | A missing or invalid required pushed authorization request is rejected with `invalid_request` ({{authorization-errors}}). |
| 5.1 of {{LIFECYCLE}} | A current statement on refresh is recommended once the recorded statement expires. | Every refresh requires a current statement ({{refresh}}). |
| 5.2 of {{LIFECYCLE}} | A replacement is needed only when the recorded statement is refused or expired, the document's digest is checked only for a replacement, and the replacement's document carries the proven key. | Every refresh re-checks the document against the recorded digests, a replacement is also needed when the recorded statement no longer matches the document, and a current attestation of the proven key takes the place of the key check ({{refresh}}). |

All other requirements of {{STATEMENT}} and {{LIFECYCLE}} continue to apply.

# Error Responses {#errors}

This specification defines no new error code.

## Error Codes {#error-codes}

Under this profile, a pushed authorization request, code redemption, or refresh is rejected with the following error codes. Other rejections use the codes that {{STATEMENT}}, {{LIFECYCLE}}, {{ABCA}}, and {{RFC9449}} assign.

`invalid_client`:
: Any of the following, or `invalid_client_attestation` in its place (Section 7.4 of {{ABCA}}):

  * The attestation is missing or unacceptable, or its attester or evidence policy is not trusted.
  * The request carries the `OAuth-Client-Attestation-PoP` header field.
  * A valid DPoP proof is under a key other than the attested key. This fails authentication, not the proof (Section 7.3 of {{ABCA}}).
  * At the PAR endpoint, the request carries an instance attestation for a document that does not select this profile or violates {{client-metadata}}.

`use_fresh_attestation`:
: An otherwise valid attestation that this profile treats as not fresh enough (Section 7.4 of {{ABCA}}): its `exp` has passed, or it names a digest other than that of the document currently served while that document matches the applicable statement.

`invalid_grant`:
: Any of the following:

  * At code redemption or refresh, a validly authenticated instance does not match a fixed establishment value of {{establishment}}, including an instance that authenticates under a key other than the proven key.
  * At code redemption or refresh, the current document exceeds a configured size bound.
  * At code redemption, the current document no longer matches the recorded digest.
  * At refresh, the current document no longer selects this profile or meets {{client-metadata}}.

`statement_required`:
: Any of the following:

  * At code redemption, the recorded statement is expired or refused.
  * At code redemption or refresh, the recorded statement's issuer is no longer trusted.

A `use_dpop_nonce` response MUST include the `DPoP-Nonce` HTTP header field.

## Error Precedence and Disclosure {#error-precedence}

Servers MUST check basic request syntax before reporting errors requiring authentication. They MUST authenticate the instance before reporting errors about a code, a refresh token, or the review governing the grant. Within authentication, they MUST verify attester identity, signature, and policy before returning the `use_fresh_attestation` error code. Nonce challenges MAY precede completion of authentication and MUST NOT reveal tenant trust configuration. Within a stage, the order of Section 6 of {{STATEMENT}} applies, and the response does not disclose other failed checks.

Details of configured issuers, attesters, evidence policies, and tenant restrictions MUST NOT be exposed in error descriptions to unauthenticated requesters, as {{STATEMENT}} requires of its issuer trust configuration.

## Authorization Endpoint {#authorization-errors}

At the authorization endpoint, when {{authorization}} requires a PAR `request_uri`, a request without a valid one is rejected with `invalid_request`.

# Examples {#examples}

The following examples are non-normative. In the HTTP examples, JWTs and opaque protocol values are shown as placeholders in angle brackets. The metadata digest below is computed over the UTF-8 bytes of the displayed client document, using LF line endings, two-space indentation, and one final LF after the closing brace.

## Reviewed Client Metadata

The following is an example of a CIMD that selects this profile:

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

The following is an example of the decoded header and payload of an instance attestation:

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

The public key represents one installation; it is absent from the shared document.

## Pushed Authorization Request and Code Redemption

The following is an example of a pushed authorization request that the client sends after obtaining a DPoP nonce (line breaks for display purposes only):

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

Here the server pulls the software statement. It binds the accepted request, key, attester, policy, and tenant to its returned `request_uri`. After authorization in the browser, the client receives a code at `http://127.0.0.1:51004/callback` and validates the response issuer and state. The following is an example of the request that redeems the code (line breaks for display purposes only):

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

The server verifies the proven key and all continuation checks. It does not accept another installation's valid attestation for this code.

## Refresh and Replacement

The following is an example of a refresh request (line breaks for display purposes only):

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

If the recorded statement has expired, a server that supports pulled statements pulls a current replacement, as {{LIFECYCLE}} provides for any establishment. If the replacement names changed metadata, the attestation needs to name that same new digest while retaining the proven key and the recorded attester and policy. Otherwise the refresh fails without changing the binding.

# Authorization Server Metadata {#server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} value:

`software_statement_instance_binding_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server supports this specification. If omitted, the default value is `false`. A server advertising `true` MUST implement the requirements of this specification. Support does not promise acceptance of any particular client, statement issuer, attester, evidence policy, or tenant.

A supporting server MUST publish its `pushed_authorization_request_endpoint` metadata value, include `attest_jwt_client_auth_dpop` in the `token_endpoint_auth_methods_supported` metadata value, advertise `S256` in the `code_challenge_methods_supported` metadata value, and publish the `dpop_signing_alg_values_supported` ({{RFC9449}}) and `client_attestation_signing_alg_values_supported` (Section 8 of {{ABCA}}) metadata values. It MUST support {{RFC9207}} and publish the `authorization_response_iss_parameter_supported` metadata value as `true`.

A client MUST confirm support for this specification before starting a grant under it. The generic attestation method advertisement alone is insufficient. This profile requires PAR for its clients even when the server also serves other clients without PAR; it does not require the server's global `require_pushed_authorization_requests` value to be `true`.

# Implementation Considerations {#implementation-considerations}

This section gives guidance to implementers. It adds no requirement.

## Bounding Establishments and Pending Requests {#resource-bounds}

A server can bound establishments as {{STATEMENT}} describes, and also per resource owner and tenant, so that one installation cannot exhaust the capacity available to other installations. It can limit pending pushed authorization requests separately from any bound on establishments, and its rate limits can account for client, instance key, tenant, and authenticated resource owner as available.

## Signing Brokers {#signing-brokers}

A signing broker ought to expose constrained operations rather than arbitrary signing: it produces proofs for the authorization server only within its own processing of an OAuth transaction, and proofs for resource requests only for resource endpoints the caller is authorized to reach and only with that caller's own access token. A broker that handles authorization for another process ought to bind the authenticated caller, intended authorization server, callback, PKCE verifier, and resulting grant to one local session ({{software-evidence}}).

## User Interface, Audit, and Logging {#ui-audit}

A server ought not to present assurances stronger than the accepted evidence policy supports.

Authorization servers ought to avoid logging complete attestations, verifiers, codes, and refresh tokens. Audit records can retain the trust decisions, policy identifier, digest, and appropriately protected key identifiers without retaining credentials.

# Deployment Considerations {#deployment-considerations}

This section gives guidance to operators and issuers. It adds no requirement.

## Revocation and Enforcement Latency {#revocation-practice}

This specification defines no attestation or instance revocation protocol. Without one, an instance attestation's `exp` bounds acceptance of evidence that has become stale without the server learning of it. A deployment can use independently established revocation mechanisms or local denial lists. A deployment that requires prompt enforcement needs short token lifetimes or independent token revocation, because withdrawal of a review (Section 3 of {{LIFECYCLE}}), removal of trust, and denial of an instance key do not revoke access tokens already issued.

## Attester Enrollment {#attester-practice}

An attester ought to limit enrollment and renewal so that one approved installation cannot obtain unlimited keys to evade per-key controls, such as denial of an instance key ({{attestation-validation}}).

## Evidence Age and Token Lifetime {#lifetime-practice}

Because an access token expires no later than the attestation accepted for it ({{redemption}}), a client holding a refresh token refreshes, with a fresh attestation, at least as often as its attester's evidence age allows, and a client without one starts a new authorization request at the PAR endpoint. {{operations}} works through an example.

# Security Considerations {#security-considerations}

The security considerations of {{STATEMENT}}, {{LIFECYCLE}}, {{ABCA}}, {{CIMD}}, {{RFC8252}}, and {{RFC9700}} apply. This section describes the additional trust boundary introduced by this specification.

## Software Evidence and Key Use {#software-evidence}

A valid signature proves that an accepted attester made an assertion; it does not improve the evidence behind it. An attester accepting self-reported executable hashes would let a copied statement and an arbitrary key satisfy the binding. The evidence and mapping requirements in {{trust}} prevent that substitution only when the deployment enforces them.

A key's non-exportability does not prove which process can use it. An acceptable evidence policy therefore requires key storage and any signing broker to restrict use to the software instance the policy covers ({{trust}}). A general signing service reachable by arbitrary local processes would let unreviewed software authenticate through an approved installation ({{signing-brokers}}). Remote attestation of a device without an application-to-key binding likewise does not prove which process can use the key.

This profile cannot ensure benign user intent, prevent an approved application from being used as a proxy, or preserve an assurance after compromise of the platform on which its evidence depends, so a server ought not to present assurances beyond what the accepted policy supports ({{ui-audit}}). Plugins, scripts, dynamic libraries, debuggers, and injected code can change the effective software identity and require explicit treatment in the evidence policy ({{trust}}).

## Loopback and Transaction Theft

An attacker can attempt to occupy a loopback port, intercept a response, or initiate its own authorization transaction using the published client identifier. PKCE and proof of the proven key protect redemption of an intercepted code. Admission of a new transaction additionally requires an acceptable instance attestation; PKCE or an unattested DPoP key alone supplies no software identity.

The client keeps its verifier and transaction state confidential and unpredictable, as their respective specifications provide. A launcher handling authorization for another process acts as the signing broker of {{signing-brokers}}.

## Downgrade Prevention {#downgrade}

A requester can try to avoid this profile by omitting every review artifact, by presenting a statement over an earlier document, or by opening a grant another way. Because {{required-binding}} bases a server's requirement for reviewed software on its own policy, withholding an artifact does not avoid it.

Existing grants retain their recorded binding ({{selection}}, {{establishment}}). A server implementing only {{STATEMENT}} continues to reject clients that select this profile where review is required.

## Artifact Substitution and Key Separation

The distinct attestation type, digest binding, subject equality, and fixed establishment values prevent a generic attestation or one for another reviewed document from being substituted, and key separation keeps a statement signing key from also signing an attestation ({{attestation-validation}}).

Attester authorization is scoped independently to client namespaces and tenants. An over-broad attester can undermine the binding for every namespace for which it can issue instance attestations. A statement issuer and an attester that collude, or that are both compromised, can make false assertions within the authority the server has assigned them.

## Freshness and Availability

Short effective lifetimes limit stale evidence but make attester availability important. The maximum evidence age ({{attestation}}) and the trust-data reuse bound ({{trust}}) limit how long stale evidence or trust data is accepted. A fresh proof demonstrates current access to a key, not that software was re-evaluated at proof time.

An access token cannot outlive the attestation accepted when it was issued ({{redemption}}), but it can outlive a review withdrawn or attester trust removed after issuance ({{revocation-practice}}).

Metadata digest binding can require coordinated publication of a new document, review, and attestation ({{operations}}). During an incomplete transition a request fails: {{STATEMENT}} does not treat the changed document as reviewed, and this profile does not fall back ({{par}}).

## Resource Exhaustion

Attestations do not remove the need for bounded document retrieval, statement selection, proof replay caches, and PAR storage ({{STATEMENT}}, {{resource-bounds}}). An installation that can enroll unlimited keys evades per-key controls ({{attester-practice}}).

# Privacy Considerations {#privacy}

The privacy considerations of {{STATEMENT}}, {{LIFECYCLE}}, and Section 11 of {{ABCA}} apply.

An instance key reused at multiple servers can correlate installations across them. Clients SHOULD use separate instance keys for separate server relationships where the evidence mechanism permits, and SHOULD avoid adding stable installation identifiers unless needed by the deployment. This profile does not require a globally stable instance identifier or an attestation audience.

An attester might learn the target server from enrollment traffic, key-scoping choices, or timing even when no audience is carried. Deployments ought to minimize evidence and identifier disclosure to both attesters and authorization servers. An evidence-policy identifier can reveal enterprise management practices; it ought to identify policy semantics without embedding user or device identifiers.

Instance attestations and proofs never appear in browser URLs ({{authorization}}). Logs and audit records can link installations to people and devices ({{ui-audit}}).

# IANA Considerations {#iana}

## OAuth Dynamic Client Registration Metadata Registry

This specification requests registration of the following value in the IANA "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}:

Client Metadata Name:
: `software_statement_instance_binding`

Client Metadata Description:
: Boolean value selecting attested instance-key binding for runtime admission using a software statement for a Client ID Metadata Document. It is defined for Client ID Metadata Documents and does not enable registration under this specification.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{client-metadata}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_instance_binding_supported`

Metadata Description:
: Boolean value indicating support for attested instance-key binding of software statements for Client ID Metadata Documents.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{server-metadata}}

## JSON Web Token Claims Registry

This specification requests registration of the following claim in the IANA "JSON Web Token Claims" registry established by {{RFC7519}}:

Claim Name:
: `cimd_instance_policy`

Claim Description:
: Absolute URI identifying the evidence policy under which an attester associates an instance key with software identified by a Client ID Metadata Document.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{attestation}}

This specification also requests that the `cimd_digest` registration requested by {{STATEMENT}} list this specification ({{attestation}}) as an additional reference.

## Media Type Registration {#media-type}

This specification requests registration of the `application/cimd-instance-attestation+jwt` media type in the IANA "Media Types" registry {{RFC6838}}.

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
: See {{security-considerations}} of this specification, Section 12 of {{ABCA}}, and Section 11 of {{RFC7519}}.

Interoperability considerations:
: n/a

Published specification:
: This specification

Applications that use this media type:
: Native OAuth clients, client attesters, and authorization servers using CIMD software statements with attested instance keys.

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

## Values Defined Elsewhere

The `iss`, `sub`, `aud`, `iat`, `exp`, and `cnf` claims are defined elsewhere and are not re-registered here. This specification uses the authentication method, HTTP fields, and error codes of {{ABCA}} without registering replacements. Section 2.3 of {{RFC9126}} lets a pushed authorization request error response carry the error codes of an extension involved in processing the request, so this specification requests no change to the registrations of the error codes it uses there.

--- back

# Managed Command-Line Client Deployment {#launcher}

This appendix is non-normative. A managed desktop can implement the evidence boundary with a trusted launcher or enterprise agent and an enterprise attestation service. Other mechanisms are possible if they meet {{trust}}.

The enterprise enrolls the launcher through an authenticated management channel. It records a manifest, authorized by the controller of the client identifier URL's origin or its delegate, that associates a signed package or exact artifact with that client identifier URL. It approves the manifest for the applicable evidence policy. The launcher uses platform facilities to authenticate the actual calling process and validate that association. It does not trust a process's own claim about its executable path or hash.

The launcher creates or provisions the instance key in protected storage and obtains an attestation through an authenticated exchange with the enterprise service. That service verifies the evidence, the binding to the key, and the document digest before issuing the attestation. The service's signing key is distinct from the instance key and is accepted through the authorization server's attester trust configuration.

For each authorization transaction, the launcher performs the OAuth exchange on behalf of the verified caller, as the signing broker of {{signing-brokers}}. It rechecks caller authorization when the key is used, including refresh, and protects returned tokens from other processes.

An implementation needs to address changes between checking software and using the key, dynamic code loading, and the ability of an executable the evidence policy covers to proxy requests for others. Platforms without reliable process identity and key-use isolation cannot obtain this profile's assurance by running the launcher with more privileges. A script-based command-line client might require an execution environment whose policy covers its interpreter, script, and dependencies, or a narrower assurance than verified executable identity.

## Example: macOS {#launcher-macos}

This example uses macOS 27 or later on Apple silicon, with a launcher application for which App Attest is available {{APPLE-WWDC26}}.

Software identity:
: The publisher signs each release with an Apple-issued developer certificate, under the hardened runtime {{APPLE-RUNTIME}} and one signing identifier. A manifest served from the client identifier URL's origin and naming the publisher's Team ID and that identifier covers each such release. A manifest naming code directory hashes covers only the listed builds.

Key isolation:
: The launcher is a per-user application whose provisioning profile authorizes a data protection keychain access group for the launcher alone. macOS grants such groups only through entitlements a provisioning profile authorizes, and only in a user context {{APPLE-KEYCHAIN}}. The launcher creates a P-256 instance key in the Secure Enclave, from which key material cannot be transferred {{APPLE-ENCLAVE}}, and keeps it under that access group, so the client cannot use the key directly.

Caller identification:
: The client connects to the launcher over XPC. The launcher sets a code-signing requirement on the connection that requires an Apple-issued certificate chain, the publisher's Team ID, and the client's signing identifier, and the system invalidates a connection whose peer does not satisfy it {{APPLE-XPC}}. The launcher never relies on a process ID or a reported path.

Code injection:
: The evidence policy requires that the client's signature carry no entitlement that disables library validation {{APPLE-LIBVAL}}, lets the dynamic linker read `DYLD_` variables {{APPLE-DYLD}}, or lets a debugger obtain the client's task port (`get-task-allow`, {{APPLE-DEBUGGER}}). The client then loads only libraries signed by Apple or by its own team.

Launcher authentication:
: Managed Device Attestation shows that the device is a genuine Apple device the organization manages {{APPLE-MDA}}, not which program is reporting. The launcher therefore registers an App Attest key, which Apple certifies as belonging to the launcher {{APPLE-APPATTEST}}. To tie that key to the managed device, the launcher returns with the attestation a single-use secret that the device management service delivered through managed app configuration, which macOS applies only to an application with a matching code signature {{APPLE-APPCONFIG}}. Each later report, carrying the instance public key and the client's signing identity, bears an App Attest assertion.

Evidence and renewal:
: No Apple attestation covers the instance key, so the enterprise service relies on the authenticated launcher's report that the key is in the Secure Enclave and usable only by the launcher. The launcher acts as the signing broker of {{signing-brokers}}. Before the maximum evidence age expires, it re-checks the device's management state and the client's signature and obtains an attestation on that new evidence.

Unsupported case:
: A script run by a signed interpreter with unrestricted imports is not covered. The verified signature is the interpreter's, and library validation does not constrain the scripts and modules it reads. Such a client remains review-only under {{STATEMENT}}.

# Operational Consequences {#operations}

This appendix is non-normative. It assumes an evidence policy whose maximum evidence age is eight hours.

* An attestation on evidence collected at 09:00 expires by 17:00, and so does every access token issued on it ({{redemption}}). Before then, a client holding a refresh token refreshes on new evidence. A client without one starts a new authorization request.
* A metadata change, such as a new loopback redirect or scope, needs a statement and attestations over the new digest before each grant refreshes onto it ({{refresh}}). An authorization in progress during the change restarts at the PAR endpoint ({{redemption}}).
* A new evidence-policy identifier ({{trust}}) or a new key requires a new grant ({{establishment}}).

# Acknowledgments
{:numbered="false"}

This specification develops the attested-key extension point in {{STATEMENT}} and composes it with attestation-based client authentication. The initial deployment question was how a command-line application could retain loopback callbacks while demonstrating an association with reviewed software.
