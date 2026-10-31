---
title: "Client ID Metadata Document Software Statement Issuance"
abbrev: oauth-cimd-sw-stmt-issuance
docname: draft-mcguinness-oauth-cimd-sw-stmt-issuance-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Dynamic Client Registration
 - Deferred Processing
 - Client ID Metadata Document
 - Approval

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "Client ID Metadata Document Software Statement"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-00
  RFC6749:
  RFC7521:
  RFC7523:
  RFC7591:
  RFC7636:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC9207:
  RFC8628:
  RFC9126:
  RFC9396:
  RFC9449:
  RFC9700:
  DTR: I-D.ietf-oauth-deferred-token-response
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  OAUTH-MRT:
    target: https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html
    title: "OAuth 2.0 Multiple Response Type Encoding Practices"
    author:
      - ins: B. de Medeiros
        name: Breno de Medeiros
        role: editor
      - fullname: Marius Scurtescu
      - fullname: Paul Tarjan
      - fullname: Michael B. Jones
    date: 2014-02-25
  FORM-POST:
    target: https://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html
    title: "OAuth 2.0 Form Post Response Mode"
    author:
      - fullname: Michael B. Jones
      - fullname: Brian Campbell
    date: 2015-04-27
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "Client ID Metadata Document Software Statement Registration"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-registration-00

informative:
  LIFECYCLE:
    target: https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt/draft-mcguinness-oauth-cimd-sw-stmt-lifecycle.html
    title: "Client ID Metadata Document Software Statement Lifecycle"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
  ATTESTERS: I-D.mcguinness-oauth-client-attesters
  APPROVAL-DCR: I-D.dellaert-oauth-approval-based-dcr
  AU-CDR:
    target: https://consumerdatastandardsaustralia.github.io/standards/
    title: "Consumer Data Standards (Australia)"
  UK-OPEN-BANKING:
    target: https://openbankinguk.github.io/dcr-docs-pub/v3.3/dynamic-client-registration.html
    title: "Open Banking UK Dynamic Client Registration Specification v3.3"

--- abstract

RFC 7591 defines how a client presents a software statement and how a registration endpoint consumes it, but not how the client obtains one. This specification defines OAuth 2.0 flows for issuing a software statement to a client identified by a Client ID Metadata Document (CIMD) and not registered with the authorization server.

A client holding an initial access token that authorizes issuance, or a statement it can renew, uses OAuth 2.0 Token Exchange (RFC 8693) without a redirect. A client can instead use a redirect flow: the authorization endpoint returns a short-lived `software_statement_code`, which the client redeems through a new token endpoint grant. In both flows, a completed decision returns a statement, and a pending decision uses Deferred Token Response and polling. The statement never appears in an authorization response URL.

The client presents the statement at registration or at runtime, or publishes it for servers to pull. Companion specifications define the statement, its validation, and its consumption.

--- middle

# Introduction

{{RFC7591}} standardizes the consumption of a software statement, a signed JWT asserting client metadata (Section 2.3 of {{RFC7591}}), but not its issuance. Issuance today relies on manual provisioning or proprietary processes. The UK Open Banking Directory {{UK-OPEN-BANKING}} and the Australian Consumer Data Right Register {{AU-CDR}} each built a central issuer, yet a client needs a separate integration for each.

{{STATEMENT}} profiles that statement to carry no client metadata and instead vouch for the exact octets of a Client ID Metadata Document (CIMD), and defines its validation and consumption. This specification defines how a client obtains one, through OAuth token exchange ({{token-exchange-profile}}) or a redirect flow ({{authorization-request}}). An issuer whose review cannot complete within a request can defer its response under {{DTR}}.

A statement makes one issuer's decision portable to the authorization servers in its audience that trust the issuer ({{STATEMENT}}). If the trust decision is local to one authorization server, an {{RFC7591}} initial access token for registration, pre-registration {{CIMD}}, or approval-based registration {{APPROVAL-DCR}} can establish the client directly.

Approval workflow, approver identity, external approval integration, acceptance of any particular issuer, and issuer discovery are out of scope. {{STATEMENT}} does not depend on this specification.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology defined in {{RFC6749}}, {{RFC7591}}, {{CIMD}}, and {{DTR}}, and the terms defined in Section 2 of {{STATEMENT}}.

This specification defines the following terms:

Software Statement Code:
: The short-lived, single-use artifact returned by the software statement code response ({{software-statement-code-response}}) and redeemed at the token endpoint ({{software-statement-code-redemption}}).

Deferring Issuer:
: An issuer that advertises `true` for `deferred_token_response_supported` ({{authorization-server-metadata}}) and so can defer a decision ({{deferred-processing}}).

# Protocol Overview

A client that holds an initial access token authorizing issuance, or a statement it can renew, uses token exchange at the token endpoint without a user agent ({{token-exchange-profile}}). A client can instead initiate the redirect flow at the authorization endpoint ({{authorization-request}}) and redeem the resulting software statement code at the token endpoint.

The issuer fetches and snapshots the client's CIMD ({{metadata-snapshot}}), decides whether to issue, and signs the statement. The client carries the statement to trusting authorization servers in its audience, at registration ({{REGISTRATION}}) or at runtime ({{STATEMENT}}). A publisher's backend acting for a client is not a party this specification authenticates. Issuance that such a backend authorizes on a client's behalf is out of scope.

~~~
                +--------------------------+
                | Client ID Metadata       |
                | Document (client_id URL) |
                +--------------------------+
                  ^                      ^
            hosts |                      | fetches and
                  |                      |   snapshots
+----------+      |                      |      +---------------+
|          +------+                      +------+    Issuing    |
|  Client  |                                    | Authorization |
|          |--(1) software statement request -->|    Server     |
|          |<-(2) software statement -----------|   (approval)  |
+----------+      (signed JWT)                  +---------------+
     |
     | (3) registration or runtime presentation
     |     carrying the software statement
     v
+---------------+  +---------------+       +---------------+
|   Trusting    |  |   Trusting    |  ...  |   Trusting    |
| Authorization |  | Authorization |       | Authorization |
|   Server 1    |  |   Server 2    |       |   Server M    |
+---------------+  +---------------+       +---------------+
~~~

## Redirect Flow

~~~
+--------+                                  +----------------------+
| Client |                                  | Authorization Server |
+--------+                                  +----------------------+
    |                                                 |
    | (A) Authorization request                       |
    |     response_type=software_statement_code       |
    |     client_id=<metadata document URL>           |
    |     code_challenge, [audience], [dpop_jkt]      |
    |------------------------------------------------>|
    |                                                 |
    | (B) Software statement code response            |
    |     (software_statement_code, state, iss)       |
    |<------------------------------------------------|
    |                                                 |
    | (C) Redemption                                  |
    |     (software_statement_code, code_verifier,    |
    |      completion_mode=deferred)                  |
    |------------------------------------------------>|
    |                                                 |
    | (D) Software statement response or              |
    |     deferred response (deferral_code)           |
    |<------------------------------------------------|
    |                                                 |
    | (E) Poll(s) (deferral_code), if deferred        |
    |------------------------------------------------>|
    |     authorization_pending or final response     |
    |<------------------------------------------------|
~~~

At (A), the authorization server validates the request, performs any user-agent interaction or initiates an approval workflow, and returns a software statement code at (B).

At (C), the client redeems the code with its Proof Key for Code Exchange (PKCE) verifier. The response at (D) carries the statement or a {{DTR}} `deferral_code`, with which the client polls at (E). Errors follow {{RFC6749}} and {{DTR}}.

# Client Identification and Authentication {#client-identity}

The `client_id` parameter in every request defined by this specification MUST be a client identifier URL conforming to {{CIMD}}. The authorization server MUST obtain and validate the document at that URL according to {{CIMD}}.

In the redirect flow, a client identifier whose CIMD cannot be retrieved or validated, including a document rejected for duplicate member names ({{metadata-snapshot}}), is invalid for the purposes of Section 4.1.2.1 of {{RFC6749}}.

The client authenticates to the token endpoint using the `token_endpoint_auth_method` and related key metadata in its CIMD. The authorization server classifies the client from that member:

* `none` (explicit) establishes a public client.
* Any other value establishes a confidential client:
  * The authorization server MUST require exactly that method. {{CIMD}} states this rule for `private_key_jwt`, and this specification applies it to every declared method.
  * If the authorization server does not support the declared method, it MUST reject the request rather than treat the client as public, with the `unauthorized_client` error code at the authorization endpoint and with the `invalid_client` error code (Section 5.2 of {{RFC6749}}) at the token endpoint.
* An omitted value establishes neither, and any request identifying such a client MUST be rejected, with the `invalid_request` error code at the authorization endpoint, returned to the redirection URI validated against the document, and with the `invalid_client` error code (Section 5.2 of {{RFC6749}}) at the token endpoint.

Classification uses the singular `token_endpoint_auth_method`. A list of declared supported methods is not interpreted.

A public client binds its requests with Demonstrating Proof of Possession (DPoP) {{RFC9449}}: in the redirect flow through the `dpop_jkt` parameter ({{authorization-request}}), and in the token exchange profile ({{token-exchange-profile}}) by a DPoP proof, which a public client MUST include on the exchange request. A confidential client MAY use DPoP in addition to its client authentication method.

## Metadata Snapshot {#metadata-snapshot}

CIMDs can change while a request is pending. For each request, the authorization server retrieves and validates the document ({{client-identity}}) and computes the metadata digest of the exact octets it retrieved, as Section 3.1 of {{STATEMENT}} requires. These octets and their digest are the request's metadata snapshot. The authorization server MUST bind the snapshot and the requested audience to each software statement code and deferral it creates, and issues the statement over that snapshot.

The authorization server MAY retrieve the document again before issuing the software statement. If it does so and the digest differs, it MUST either re-evaluate the request under the new document, binding the new document as the snapshot, or reject the request. This applies even if the change touches no member its policy examines. A statement over a superseded snapshot fails the digest comparison (Section 5.2 of {{STATEMENT}}).

The authorization server MUST NOT combine values from different document versions. An approval recorded against a superseded snapshot does not carry forward without a fresh issuance-policy decision.

Replacing the snapshot does not alter the sender-constraint context fixed for a deferral at origination ({{deferred-processing}}). If a replacement snapshot no longer authorizes the key material behind that context, for example because the client-authentication key is absent from the new document, the authorization server MUST invalidate the deferral. The client then makes a new request under its current keys.

The authorization server MUST reject a document containing duplicate object member names (Section 5.2 of {{STATEMENT}}).

An issuer can require `jwks` inline (Section 13.2 of {{STATEMENT}}).

{{CIMD}} permits a document to carry a `software_statement` member. An issuer evaluates the document as served and MUST NOT refuse a document because it carries the member. Trusting authorization servers ignore a statement embedded there (Section 3 of {{STATEMENT}}), and refusing would leave a client that published its statement unable to renew it.

# Common Request Rules {#common-rules}

## Audience {#audience}

The `audience` parameter of an authorization request ({{authorization-request}}) or token exchange ({{token-exchange-profile}}) names a target service at which the client intends to use the statement, with the semantics of Section 2.1 of {{RFC8693}}. The parameter can be repeated to request several. Each value MUST be an authorization server issuer identifier as defined by {{RFC8414}}. Values MUST NOT be duplicated, and order is insignificant.

The authorization server selects the final audience according to policy. It MUST NOT place in the statement's `aud` claim any value the request did not carry, except as {{renewal}} provides.

If no requested audience is acceptable, the authorization server MUST reject the request with the `invalid_target` error code, as defined by {{RFC8707}} at the authorization endpoint and by Section 2.2.2 of {{RFC8693}} at the token endpoint. An authorization server whose policy requires a restricted audience rejects a request carrying none with the same error.

These `audience` semantics apply only to software statement requests and do not affect proprietary uses of the parameter for access-token targeting.

## Prohibited Parameters {#prohibited-parameters}

A software statement request does not grant access to a protected resource, and it asks for a statement rather than presenting one. An authorization request, software statement code redemption, or token exchange under this specification MUST NOT contain the `scope` parameter, the `resource` parameter ({{RFC8707}}), the `authorization_details` parameter ({{RFC9396}}), or the `software_statement` parameter (Section 5.4 of {{STATEMENT}}). The authorization server MUST reject a request containing any of them with the `invalid_request` error code. The `audience` parameter ({{audience}}) is not prohibited: it describes the requested artifact rather than requesting access.

# Token Exchange Profile {#token-exchange-profile}

A client that already holds a token carrying issuance authority MAY exchange that token for a statement using OAuth 2.0 Token Exchange {{RFC8693}} instead of the redirect flow ({{authorization-request}}).

The client sends a token exchange request as defined in Section 2.1 of {{RFC8693}}, with the following parameters:

`requested_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`subject_token` and `subject_token_type`:
: REQUIRED. One of the following:

  * **First issuance.** An initial access token, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:access_token`: an authorization credential issued out of band by this authorization server that pre-authorizes software statement issuance, analogous to the initial access token of {{RFC7591}} ({{initial-access-tokens}}). The credential itself is deployment-defined.
  * **Renewal.** A software statement this authorization server previously issued with a `sub` claim equal to the request's `client_id`, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:software-statement` ({{renewal}}).

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`audience`:
: OPTIONAL. The requested audience ({{audience}}), bounded on renewal as {{renewal}} requires.

`completion_mode`:
: As defined for software statement code redemption ({{software-statement-code-redemption}}).

The request MUST NOT contain the `actor_token` or `actor_token_type` parameters, and the prohibitions of {{prohibited-parameters}} apply.

The client authenticates, and a public client constrains the request, as described in {{client-identity}}. An assertion-based method such as `private_key_jwt` {{RFC7521}} {{RFC7523}} serves if the document specifies one.

A DPoP proof on the exchange constrains any resulting deferral, and it protects the subject token only if that token is bound to the proof's key ({{te-considerations}}).

The authorization server MUST validate the subject token before retrieving client-controlled metadata or enqueueing any processing. A subject token MUST result in the `invalid_request` error code (Section 2.2.2 of {{RFC8693}}) if any of the following holds:

* It is invalid, revoked, or, for a statement, one that {{renewal}} forbids accepting.
* It has expired, unless it is a prior software statement accepted under {{renewal}}.
* It does not authorize issuance for the presented `client_id` parameter.

An error response to a token exchange MUST NOT contain the subject token.

The exchange results in a software statement token response ({{software-statement-response}}), a deferral at a deferring issuer ({{deferred-processing}}), or the terminal denial of {{terminal-denial}}.

## Initial Access Tokens {#initial-access-tokens}

An initial access token presented under this profile:

* MUST be time limited.
* MUST be limited to the issuer.
* MUST be bound to an exact client identifier URL or an explicitly authorized client identifier namespace.
* MUST have at least 128 bits of entropy if it is opaque.
* MUST be sender-constrained to a client key if it is reusable, for example through DPoP or mutual TLS, and the authorization server MUST verify that binding against the key the exchange request proves.
* MUST be single-use if it is a bearer token.

The authorization server MUST enforce every restriction the token carries, such as on audiences or metadata, and MUST prevent replay beyond its permitted number of uses. A use is consumed when the authorization server commits to an outcome: issuing a statement, creating a deferral, or denying issuance. Concurrent presentations of a single-use token MUST NOT both be committed.

Issuance policy determines whether an initial access token authorizes the client only to request a statement or also authorizes its issuance.

## Renewal {#renewal}

A client renews by presenting its current or most recent software statement as the subject token. The authorization server MUST verify the following:

* The authorization server issued the statement.
* The statement's `sub` claim equals the request's `client_id` parameter.
* The client authenticated with a key carried both by the document the statement's `cimd_digest` claim names and by the current document.

That authentication is the holder binding: without it, whoever held a copy of the statement could renew. To check the reviewed document's keys, an issuer offering renewal needs the octets of each document it issues a statement over ({{audit-logging}}).

A client rotating keys inline renews while its document carries both the old and the new key. For a document naming `jwks_uri`, the binding is to whatever that location serves. A public client holds no such key and obtains a new statement through the redirect flow ({{authorization-request}}).

A renewed statement MUST NOT be broader than the subject statement: its `aud` is the requested audience narrowed to the subject statement's `aud`, or that `aud` if the request names none, it carries any `aud_tenant` claim of the subject statement, and its `statement_uses` permits no use the subject statement does not. It is matched to the subject statement by `iss` and `sub` ({{LIFECYCLE}}, {{REGISTRATION}}).

An issuer MUST NOT accept as the subject token a statement it has withdrawn, or one whose status it publishes as other than `VALID` ({{LIFECYCLE}}). Renewing such a statement would reissue the decision that withdrawal ended.

The authorization server MAY accept a statement that has expired, and SHOULD bound how long after expiry it will do so. A renewal after an extended absence amounts to a new issuance request.

If the current document's digest equals the subject statement's `cimd_digest`, whether renewal requires fresh review is issuer policy. If the digests differ, the issuer MUST apply the decision it would apply to a first issuance for that document, and MUST NOT renew on the strength of the prior statement alone. Without this rule, whoever can change the document could obtain the issuer's signature over an unreviewed change.

# Software Statement Authorization Request {#authorization-request}

The client sends an authorization request as described in Section 4.1.1 of {{RFC6749}}, with the following parameters:

`response_type`:
: REQUIRED. The value MUST be `software_statement_code`.

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`redirect_uri`:
: REQUIRED. The value MUST exactly match one of the redirection URIs in the CIMD, subject to the redirection URI rules in Section 4.2 of {{CIMD}}. Whatever authentication method the document declares, the value MUST NOT be a redirection URI that another application on the same device could claim, as Section 5.6.3 of {{STATEMENT}} defines ({{authorization-response-security}}).

`state`:
: OPTIONAL, as in Section 4.1.1 of {{RFC6749}}.

`code_challenge`:
: REQUIRED. A PKCE challenge as defined by {{RFC7636}}.

`code_challenge_method`:
: REQUIRED. The value MUST be `S256`.

`response_mode`:
: OPTIONAL. The response mode, as defined by {{OAUTH-MRT}}. The default for `response_type=software_statement_code` is `query`. {{software-statement-code-response}} gives response mode considerations.

`audience`:
: OPTIONAL. The requested audience ({{audience}}).

`completion_mode`:
: OPTIONAL. A value that includes `deferred`, sent as the advance hint of Section 4.2 of {{DTR}}.

`dpop_jkt`:
: REQUIRED for a public client and OPTIONAL otherwise. The parameter has the semantics of Section 10 of {{RFC9449}}.

The authorization server MUST reject with the `invalid_request` error code a request that omits a required PKCE parameter or a required `dpop_jkt` parameter. The prohibitions of {{prohibited-parameters}} apply.

Hybrid response types that combine `software_statement_code` with `code`, `token`, `id_token`, or any other response type are not defined. An authorization server MUST reject such a request with the `unsupported_response_type` error code.

The following is a non-normative example of an authorization request from a confidential client (line breaks for display purposes only):

~~~ http
GET /authorize?response_type=software_statement_code
  &client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
  &state=4L7xQ2mN9pR6sT1vW8yZ3aB5cD0fG2hJ
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256 HTTP/1.1
Host: server.example.com
~~~

# Authorization Response {#authorization-response}

After validating the request and performing any immediate interaction, the authorization server returns the software statement code response or an error. The authorization server MUST NOT place the software statement or approval-sensitive information in any response from the authorization endpoint.

A denial of issuance is never signaled in the authorization response: the authorization server returns a software statement code whether the issuance decision is approved, pending, or already denied, and delivers any denial at redemption ({{terminal-denial}}).

## Software Statement Code Response {#software-statement-code-response}

The software statement code response applies the authorization response protections of {{RFC9700}}. The authorization server returns the following parameters to the client's redirection endpoint, using the selected response mode:

`software_statement_code`:
: REQUIRED. A short-lived, single-use artifact redeemed at the token endpoint ({{software-statement-code-redemption}}). It MUST be bound to the client identifier, redirection URI, PKCE challenge, metadata snapshot, requested audience, and, when the `dpop_jkt` parameter was present, that JWK thumbprint.

  The value MUST contain at least 128 bits of entropy from a cryptographically secure random source, be opaque to the client, expire shortly after issuance, and not be accepted more than once.

`state`:
: REQUIRED if the authorization request contained the `state` parameter. The exact value received in the authorization request.

`iss`:
: REQUIRED. The authorization server issuer identification parameter defined by {{RFC9207}}.

A software statement code is not an authorization code and MUST NOT be redeemable as one.

Before redeeming a software statement code, the client MUST verify the `state` parameter, if it sent one, and MUST validate the authorization response `iss` parameter according to {{RFC9207}}.

Because scripts at the redirection endpoint can access the fragment, the `fragment` response mode SHOULD NOT be used. A client MAY request the `form_post` response mode {{FORM-POST}}.

The following is an example of a software statement code response using the default `query` response mode (line breaks for display purposes only):

~~~ http
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
  software_statement_code=V7e1gP8zT2mN4qR6sW9xY3aB5cD7fH0jK2pL4uQ6vX&
  state=4L7xQ2mN9pR6sT1vW8yZ3aB5cD0fG2hJ&
  iss=https%3A%2F%2Fserver.example.com
~~~

Request validation errors follow Section 4.1.2.1 of {{RFC6749}}.

# Software Statement Code Redemption {#software-statement-code-redemption}

The client redeems a software statement code by sending an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format with the following parameters:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:software-statement`.

`software_statement_code`:
: REQUIRED. The software statement code returned by the authorization endpoint.

`redirect_uri`:
: REQUIRED. The same redirection URI used in the authorization request.

`client_id`:
: REQUIRED. The same client identifier URL used in the authorization request.

`code_verifier`:
: REQUIRED. The PKCE verifier corresponding to the `code_challenge` in the authorization request.

`completion_mode`:
: REQUIRED at a deferring issuer, with a value that MUST include `deferred`. A request without it is rejected with `invalid_request`.

The prohibitions of {{prohibited-parameters}} apply. The request MUST NOT contain the `audience` parameter, which was bound at the authorization endpoint. A request containing it is rejected with `invalid_request`.

The client authenticates according to {{client-identity}}. When the `dpop_jkt` parameter was included in the authorization request, the client MUST send a DPoP proof for the token endpoint using the same key.

The authorization server MUST validate the software statement code and all of its bindings before processing the request. An invalid, expired, previously used, or incorrectly bound code MUST result in an `invalid_grant` error. A PKCE or DPoP binding failure is handled according to {{RFC7636}} or {{RFC9449}}, respectively.

A redemption attempt consumes the software statement code whenever the presented code value is one the authorization server issued that has neither expired nor been consumed, including when its PKCE, DPoP, or client-authentication bindings fail. A DPoP nonce challenge {{RFC9449}} is not a binding failure: an authorization server requiring a nonce issues the `use_dpop_nonce` challenge before evaluating the code, which remains unconsumed.

When a previously consumed code is presented again, the authorization server SHOULD revoke any deferral derived from it ({{RFC9700}}). To keep anyone who observed the code in a URL from triggering that revocation, an authorization server MAY limit revocation to replays that pass the PKCE check.

For a valid, unconsumed code, the authorization server returns the following, according to the issuance decision:

* **Approved.** The software statement token response ({{software-statement-response}}).
* **Denied.** The terminal denial of {{terminal-denial}}.
* **Pending.** A deferred token response ({{deferred-processing}}).

An issuer that does not defer completes the decision before responding.

The following is a non-normative example of a redemption request from a confidential client at a deferring issuer (line breaks for display purposes only):

~~~ http
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3A
  software-statement
  &software_statement_code=V7e1gP8zT2mN4qR6sW9xY3aB5cD7fH0jK2pL4uQ6vX
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
  &client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
  &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
  &completion_mode=deferred
  &client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
  client-assertion-type%3Ajwt-bearer
  &client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6ImNsaWVudC0xIn0...
~~~

# Deferred Processing {#deferred-processing}

A deferring issuer MAY answer a redemption ({{software-statement-code-redemption}}) or token exchange ({{token-exchange-profile}}) with a deferred token response, which follows {{DTR}} except that the final response MUST be delivered only by polling and the statement does not inherit the deferral's DPoP binding. A client MUST NOT send `client_notification_token`, which the authorization server rejects with `invalid_request`, and the server MUST NOT deliver a callback, even to a `deferred_client_notification_endpoint` in the document.

The following is a non-normative example of a first polling request for a deferral created by a token exchange, from a confidential client authenticating with the same `private_key_jwt` method as the exchange (line breaks for display purposes only):

~~~ http
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred
&deferral_code=8xLOxBtZp8
&client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6ImNsaWVudC0xIn0...
~~~

# Software Statement Token Response {#software-statement-response}

A successful response has HTTP status code 200, a media type of `application/json`, and the following members:

`access_token`:
: REQUIRED. The software statement issued by the authorization server. It MUST conform to {{STATEMENT}}, with the `cimd_digest` claim set to the digest of the bound metadata snapshot, the `aud` claim set to the selected audience if one is restricted, and the `statement_uses` claim set to the uses issuance policy permits.

`issued_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`token_type`:
: REQUIRED. The value MUST be `N_A` (Section 2.2.1 of {{RFC8693}}).

`expires_in`:
: RECOMMENDED. The remaining lifetime of the software statement in seconds. If present, it MUST be consistent with the statement's `exp` claim.

The response MUST NOT contain the `refresh_token` or `scope` members. The authorization server MUST include the `Cache-Control` HTTP header field with the value `no-store`.

The following is an example of a software statement token response:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":
    "eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1lbnQrand0...",
  "issued_token_type":
    "urn:ietf:params:oauth:token-type:software-statement",
  "token_type": "N_A",
  "expires_in": 3600
}
~~~

The `access_token` member is a security-token container ({{RFC8693}}), not an OAuth access token. The software statement it carries is consumed only as {{STATEMENT}} and {{REGISTRATION}} provide, and MUST NOT be attached to a request as an `Authorization: Bearer` credential. A protected resource cannot validate one (Section 4 of {{STATEMENT}}).

## Terminal Denial {#terminal-denial}

When the authorization server decides not to issue the requested software statement, it returns a token error response as defined in Section 5.2 of {{RFC6749}} with the `access_denied` error code (Section 3.5 of {{RFC8628}}), and MUST include the `Cache-Control: no-store` response header field. This applies to code redemption and token exchange alike, including when the denial answers a polling request ({{deferred-processing}}).

The denial is terminal for the request. A denied deferral behaves as {{DTR}} specifies for a request resolved with an error, and a client that receives the `access_denied` error code in response to a polling request MUST NOT send another polling request with that `deferral_code`. A denial does not preclude a later issuance request. Whether to accept one is issuance policy.

# Issuer as Trusting Authorization Server {#same-server}

An authorization server that issued a statement already holds the decision that statement records. If it is also a trusting authorization server, it MAY admit the client on that record rather than require the statement to be presented back to it, at registration or at runtime.

Because the record of a decision is not a fresh retrieval of the document, a server admitting a client on the record of a statement it issued MUST apply every check Sections 3.2 and 5.1 of {{STATEMENT}} apply to a presented statement, and at registration every check {{REGISTRATION}} applies, taking the statement's claims from its record of the decision. The record replaces only the signature: the server still obtains the document at `sub` and compares its digest with the recorded one.

At runtime, the recorded decision replaces the presented statement, not the proof of who is making the request. The request MUST meet the sender constraint of Section 5.6 of {{STATEMENT}} against the reviewed document, as a presentation would, and a successful admission creates an establishment.

# Authorization Server Metadata {#authorization-server-metadata}

An authorization server that issues software statements under this specification advertises `true` for `client_id_metadata_document_supported` ({{CIMD}}) and publishes the authorization server metadata that Section 8 of {{STATEMENT}} requires of an authorization server that issues statements.

A deferring issuer publishes the metadata {{DTR}} requires, including `true` for `deferred_token_response_supported`. An issuer that does not defer advertises neither that value nor the deferral-code token type in the `revocation_endpoint_token_type_values_supported` metadata value, and need not implement {{DTR}}.

An authorization server supporting the redirect flow advertises `urn:ietf:params:oauth:grant-type:software-statement` in `grant_types_supported`, `software_statement_code` in `response_types_supported`, and `true` for `authorization_response_iss_parameter_supported` as defined by {{RFC9207}}.

An authorization server supporting the token exchange profile ({{token-exchange-profile}}) advertises `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported` and publishes the `software_statement_subject_token_types_supported` metadata value defined in this section. An implementation supporting only that profile advertises neither the software statement grant nor the `software_statement_code` response type.

This specification defines the following additional authorization server metadata {{RFC8414}} value:

`software_statement_subject_token_types_supported`:
: REQUIRED for an authorization server that supports the token exchange profile ({{token-exchange-profile}}), and absent otherwise. A JSON array of the `subject_token_type` values the authorization server accepts when `requested_token_type` is `urn:ietf:params:oauth:token-type:software-statement`. Publishing this member signals support for the profile. Listing `urn:ietf:params:oauth:token-type:software-statement` among its values advertises renewal by prior statement ({{renewal}}).

# Implementation Considerations {#implementation-considerations}

This section gives guidance to implementers. It adds no requirement.

## Retrieval and Approval-Queue Bounds {#processing-bounds}

An authorization server can rate-limit token exchanges, cache successful retrievals and back off after failures as Section 11.1 of {{STATEMENT}} describes, and bound pending deferrals per client identifier and requester. Because the redirect flow reaches the approval queue before any client-authenticated step, the same rate limits and pending-approval bounds per client identifier can apply there.

## Handling the Issued Statement {#issued-statement-handling}

A client ought to key the statement on `issued_token_type` and keep it out of logs.

## Approval Interface {#approval-interface}

An approval interface ought to state that the decision vouches for the client's CIMD, ought not to imply that the approver is granting the client access to resources, and ought to present the following:

* The client identifier URL, with its origin shown as the client's identity, and with display metadata such as the `client_name` and `logo_uri` members marked as asserted by the client (a look-alike document can carry another vendor's name).
* The document content the statement will vouch for, identified by its metadata digest.
* The audience the issuer intends to place in the statement.
* The tenant the decision is confined to, if the statement will carry the `aud_tenant` claim ({{STATEMENT}}).
* The intended lifetime.

If the client requested a different or broader audience, the interface ought to make narrowing visible.

## Audit and Logging {#audit-logging}

An authorization server ought to keep an audit record that binds each decision, whether approval or denial, to the following:

* The metadata digest ({{metadata-snapshot}}) of the document the deciding party evaluated.
* The policy under which the decision was made.
* The identity of the deciding party.
* The time of decision.

An authorization server ought to retain the exact retrieved octets of the approved document for audit, because a re-serialized copy cannot reproduce the digest. A deployment that requires approver attribution can retain it in an audit record or define a statement claim and its privacy semantics.
# Deployment Considerations {#deployment-considerations}

This section gives guidance to operators and issuers. It adds no requirement.

## Review and Approval Practice {#review-practice}

Because the client authors the metadata document, an issuer ought to corroborate security-relevant metadata through evidence beyond the document itself. Issuance policy can relate the redirection URI's origin to the client identifier URL or the CIMD's `client_uri` member.

The policy governing who can approve issuance ought to be at least as restrictive as the policy under which the issuer establishes a client manually. A token exchange carries no in-band evidence of user participation, so an authorization server ought not to treat one as implying prior user consent. An initial access token that authorizes issuance ({{initial-access-tokens}}) records a decision made, when the token was issued, under the policy that governs approval in the redirect flow.

To keep a look-alike document from reaching approval, an issuer ought to limit what an approver can approve to client identifiers under publisher namespaces it has enrolled. A document naming attesters in `client_attesters` ({{ATTESTERS}}) endorses them for the software under review and deserves particular scrutiny.

## Deferral Lifetimes {#lifetime-practice}

Issuers ought to set deferral code lifetimes that reflect their actual approval latency, which for a review involving human judgment can be hours or days ({{deferred-processing}}).

# Security Considerations {#security-considerations}

The security considerations of Section 13 of {{STATEMENT}}, {{CIMD}}, and Section 5 of {{RFC8693}} apply.

## Authorization Response Security {#authorization-response-security}

The software statement is a signed credential and can contain sensitive deployment information, so it is never returned in an authorization response ({{authorization-response}}). The client's `iss` check ({{software-statement-code-response}}) counters mix-up.

{{authorization-request}} refuses a redirection URI another application can claim for the reason Section 5.6.3 of {{STATEMENT}} gives. A native public client can host an `https` endpoint or use the token exchange profile ({{token-exchange-profile}}).

Endpoint or key control informs issuance policy but does not determine issuance ({{review-practice}}).

A software statement code in a URL is visible to browser history, referrer fields, logs, and other observers. It is short lived, single use, and unredeemable without the PKCE verifier and any bound `dpop_jkt` key. The `form_post` response mode ({{software-statement-code-response}}) keeps the code out of URLs.

## Token Exchange Considerations {#te-considerations}

Validating the subject token before metadata retrieval or enqueueing ({{token-exchange-profile}}) limits resource consumption by unauthorized requesters. Other bounds are local policy ({{processing-bounds}}).

A subject token is an authorization credential, not a client identifier. If the CIMD declares a method, a subject token does not substitute for client authentication.

A subject token appears in a form body, so any component recording request bodies can expose it. Clients and authorization servers MUST protect a subject token as a bearer credential unless its format provides proof of possession. An authorization server ought to keep subject tokens out of logs, traces, and audit records, and an initial access token ought to be integrity protected and kept confidential in transit and at rest.

## Renewal by Prior Statement

If the reviewed and current documents name a `jwks_uri`, whoever controls that location can supply a key that satisfies the holder binding of {{renewal}}. Binding renewal to the client's key spares automated renewal a long-lived reusable initial access token, a standing credential to mint statements.

## Approver Authority

A software statement does not identify the human or system that approved issuance ({{audit-logging}}). A look-alike document, or one naming attesters in `client_attesters` ({{ATTESTERS}}), can reach an approver ({{review-practice}}).

An approved statement is accepted at every authorization server in its audience, not only within the approver's own scope. Approval by a party authorized only for a personal or organizational scope MUST NOT produce a statement whose audience exceeds that scope, or whose `aud_tenant` claim names a tenant outside it.

# Privacy Considerations

The privacy considerations of {{STATEMENT}} apply.

The authorization server learns the client identifier URL, the client's CIMD, and information about the party interacting with the authorization endpoint. It ought to collect and retain only the information required for issuance, security monitoring, and audit obligations.

An `aud` claim reveals which authorization servers the client intends to use. Section 14 of {{STATEMENT}} weighs that disclosure against omitting the claim, and Section 12.2 of {{STATEMENT}} advises naming an audience, which an issuer does by requiring one in the request ({{authorization-request}}).

If the client's relationship with the authorization servers its `audience` parameter names is sensitive, a redirect-flow client SHOULD use pushed authorization requests {{RFC9126}}.

Approval records can link a person to a client and deployment. Such records ought to be access-controlled and retained only as long as required.

A central issuer also learns from renewals which of its statements are in active use, so issuance and renewal logs deserve the same care as the statements.

# IANA Considerations {#iana}

## OAuth Authorization Endpoint Response Types Registry

This specification requests registration of the following value in the IANA "OAuth Authorization Endpoint Response Types" registry established by {{RFC6749}}:

Response Type Name:
: `software_statement_code`

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-request}} and {{authorization-response}}

## OAuth URI Registry

This specification requests registration of the following values in the IANA "OAuth URI" registry:

URN:
: `urn:ietf:params:oauth:grant-type:software-statement`

Common Name:
: OAuth Software Statement Grant Type

Change Controller:
: IETF

Specification Document(s):
: This specification, {{software-statement-code-redemption}}

URN:
: `urn:ietf:params:oauth:token-type:software-statement`

Common Name:
: OAuth Software Statement Token Type

Change Controller:
: IETF

Specification Document(s):
: This specification, {{software-statement-response}} and {{token-exchange-profile}}

## OAuth Parameters Registry

This specification requests that IANA add this specification, {{audience}}, as an additional reference for the existing `audience` parameter registered by {{RFC8693}} and extend its usage location to include authorization requests. The parameter name and change controller are unchanged.

This specification likewise requests that IANA add it as an additional reference for the `completion_mode` parameter ({{authorization-request}}). The parameter is defined by {{DTR}}, whose registration already includes authorization requests. The name, usage location, and change controller are unchanged.

This specification also requests registration of the following value in the IANA "OAuth Parameters" registry established by {{RFC6749}}:

Parameter Name:
: `software_statement_code`

Parameter Usage Location:
: authorization response, token request

Change Controller:
: IETF

Specification Document(s):
: This specification, {{software-statement-code-response}} and {{software-statement-code-redemption}}

## OAuth Extensions Error Registry

The `access_denied` and `invalid_target` error codes are already registered. This specification requests that IANA add this specification as a reference to the existing `access_denied` entry, and update the existing `invalid_target` entry, adding authorization error responses to its usage locations and this specification as a reference:

Error Name:
: `access_denied`

Error Usage Location:
: token error response

Related Protocol Extension:
: RFC 8628

Change Controller:
: IETF

Specification Document(s):
: {{RFC8628}}, this specification ({{terminal-denial}})

Error Name:
: `invalid_target`

Error Usage Location:
: authorization error response, in addition to the locations already registered

Related Protocol Extension:
: resource parameter

Change Controller:
: IESG

Specification Document(s):
: {{RFC8707}}, this specification ({{authorization-request}})

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following values in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_subject_token_types_supported`

Metadata Description:
: JSON array of subject token type values accepted for exchanging into software statements; signals support for the token exchange profile.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-server-metadata}} and {{token-exchange-profile}}

--- back

# Design Rationale {#design-rationale}

**Why the software statement code is not an authorization code.** Redeeming an authorization code issues an access token under {{RFC6749}}. This flow issues an artifact that grants nothing. A distinct code keeps implementations that treat any code as redeemable from confusing the two.

**Why not the device authorization grant.** {{RFC8628}} fits a human decision that outlives a request, and an issuer whose approval is always out of band can use it. It assumes a user co-present with a constrained device who enters a user code elsewhere. Issuance approval is made by an administrator or reviewer who does not operate the client and is often not present, and the client already has a browser. The redirect flow covers an approver reachable through that browser. The token exchange profile covers the case with no browser.

**Why not client attestation or OpenID Federation.** Section 10 of {{STATEMENT}} relates statements to client attestation and OpenID Federation. Federation standardizes trust resolution, not enrollment, approval, or deferred completion.

# Acknowledgments
{:numbered="false"}

This specification builds on {{DTR}}, {{CIMD}}, and {{APPROVAL-DCR}}. The author thanks the authors and contributors to those specifications and the OAuth working group participants whose discussions informed this work.
