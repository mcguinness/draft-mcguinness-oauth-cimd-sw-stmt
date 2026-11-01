---
title: "Client ID Metadata Document Software Statement Registration"
abbrev: oauth-cimd-sw-stmt-registration
docname: draft-mcguinness-oauth-cimd-sw-stmt-registration-latest
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
 - Registration Validity

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
  RFC7519:
  RFC7591:
  RFC8414:
  RFC9126:
  RFC9700:
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "Client ID Metadata Document Software Statement"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-00
  CIMD: I-D.ietf-oauth-client-id-metadata-document

informative:
  RFC7592:

--- abstract

A companion specification defines a software statement for a Client ID Metadata Document (CIMD). The statement carries no client metadata: it binds an issuer's review to the exact octets of the client's own document. This specification defines how an authorization server consumes such a statement in an RFC 7591 registration request, taking every metadata value from the reviewed document rather than from the request. It also defines how the statement's expiry bounds the resulting registration, which RFC 7591 leaves unbounded, and how a client renews the registration by delivering a replacement statement, which also updates the registration's metadata.

--- middle

# Introduction

A software statement (Section 2.3 of {{RFC7591}}) carries a third party's approval of client software into a dynamic client registration request. In {{RFC7591}}, values in the statement take precedence over those in the request. A statement under Section 3 of {{STATEMENT}} carries no client metadata: it names a Client ID Metadata Document (CIMD) by its URL and binds the review to that document's exact octets. An {{RFC7591}} server that verified such a statement without implementing this specification would take every metadata value from the request, attaching the issuer's approval to redirection URIs and keys nobody reviewed.

{{RFC7591}} also defines no expiry or renewal for a registration. A registration based on a statement can outlive the statement, and nothing keeps the registration's validity or metadata in line with the issuer's decision once the issuer narrows or withdraws its review or stops renewing the statement.

{{STATEMENT}} admits a client at request time without registering it. This specification is for deployments that register clients through {{RFC7591}} instead, for example to assign their own client identifiers or to manage registrations under {{RFC7592}}. It defines consumption of the statement at registration ({{dcr-presentation}}), the registration's validity ({{registration-validity}}), and its renewal by a replacement statement ({{revalidation}}). This specification depends on {{STATEMENT}}, which does not depend on it.

## Protocol Overview

The following non-normative sequence summarizes a statement-governed registration:

1. The client registers through {{RFC7591}}, carrying a statement in the `software_statement` member.
2. The authorization server derives the registration's metadata from the reviewed document and records the statement's identity. The registration is valid until the statement ceases to be current, at the latest its effective expiry ({{registration-validity}}).
3. The client delivers a replacement statement in a token or pushed authorization request under the registration or, if supported, an {{RFC7592}} update request. A valid replacement replaces the registration's metadata with metadata derived from the document it reviews, and sets the registration's validity to its effective expiry.
4. If no replacement arrives, the registration ceases to be valid and requests under it are rejected.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology from {{RFC6749}} (OAuth), {{RFC7591}} (client metadata and software statements), and {{CIMD}} (Client ID Metadata Documents), and the terms defined in Section 2 of {{STATEMENT}}.

This specification defines the following terms.

Delivery:
: Carrying a statement in a request under a statement-governed registration, in one of the locations {{revalidation}} lists, so that it can renew the registration. A delivery is not a runtime presentation.

Governing Statement:
: The statement that bounds a statement-governed registration: the one consumed at registration or, after a renewal, the replacement that renewed it most recently ({{revalidation}}). The authorization server identifies it by its `iss` and `jti` claims.

Replacement:
: A delivered statement with the governing statement's `iss` and `sub` claims and a later `iat` ({{revalidation}}).

Statement-Governed Registration:
: An {{RFC7591}} client registration created from a statement under this specification ({{dcr-presentation}}).

# Consumption at Registration {#dcr-presentation}

The statement is consumed in the `software_statement` member of an {{RFC7591}} registration request. The authorization server validates it under Section 3.2 of {{STATEMENT}} and registers the client under its ordinary registration policy. If a required issuer covers the statement's `sub` in the request's tenant (Section 4 of {{STATEMENT}}), the authorization server MUST NOT create a registration on a statement from another issuer.

A required issuer constrains only registrations made on statements, because a registration made without a statement names no client identifier URL for a required issuer to cover. A deployment that needs every registration in a tenant reviewed requires a statement for every registration there and rejects a request that carries none with `unapproved_software_statement` ({{errors}}).

The retrieval requirements of Sections 3.1 and 5.3 of {{STATEMENT}} apply to retrieving the document as they apply to a runtime presentation, and a server can apply the bounds of Section 11.2 of {{STATEMENT}}. Following the order of Section 5.1 of {{STATEMENT}}, the authorization server completes every check that needs no client-controlled retrieval before resolving the document.

The reviewed document takes the role that {{RFC7591}} gives a statement's own values. In deriving a registration's metadata, at registration or for a replacement ({{revalidation}}), the authorization server:

* MUST obtain the CIMD at the statement's `sub` claim and compare its digest with the `cimd_digest` claim as Section 5.2 of {{STATEMENT}} requires, rejecting the request if the digests differ, and MUST derive the registered metadata from the octets whose digest matched.
* MUST validate the document as {{CIMD}} requires, including that its `client_id` member matches the statement's `sub` claim.
* MUST take every client metadata value from that document, except values the authorization server issues, such as `client_id` and `client_id_issued_at`.
* MUST NOT take any client metadata value from the request, or from any {{RFC7592}} update request for the registration, whether or not the document carries the corresponding member.

Client metadata in a registration or update request does not contribute to the registration. The registered `client_id` is assigned as {{RFC7591}} provides, and the document's own `client_id` member is the URL recorded as the governing statement's `sub`. A member the document omits, other than `token_endpoint_auth_method` (rejected below), takes its {{RFC7591}} default. The registration's metadata changes only when a replacement is applied ({{revalidation}}).

The authorization server MUST reject the registration, or a replacement ({{revalidation}}), if any of the following holds:

* The document omits the `token_endpoint_auth_method` member.
* The document declares `token_endpoint_auth_method` of `none`, unless the authorization server registers the client under {{public-client-registration}}.
* Any of the document's redirection URIs could be claimed by another application on the same device (Section 5.6.3 of {{STATEMENT}}), whatever authentication method the document declares.

Unlike runtime presentation (Section 5 of {{STATEMENT}}), registration has no review-only form: the rejections above cover the document conditions that make a presentation review-only.

Software whose document declares `token_endpoint_auth_method` of `none` and that the server does not register under {{public-client-registration}} can instead use runtime presentation if it meets the public-client binding requirements of Section 5.6.2 of {{STATEMENT}}.

## Tenant Restriction {#tenant-restriction}

If the statement's decision is confined to one of the authorization server's tenants, by its `aud_tenant` claim or by the server's trust configuration for its issuer (Section 4 of {{STATEMENT}}), the registration is restricted to that tenant. Otherwise it carries no tenant restriction under this specification. A replacement ({{revalidation}}) determines the restriction as follows:

| Registration before | Replacement's decision | After |
| --- | --- | --- |
| Restricted to a tenant | Confined to that tenant, or not confined | Restricted to that tenant |
| Restricted to a tenant | Confined to another tenant | Fails validation (Section 3.2 of {{STATEMENT}}) |
| No restriction, confidential client | Confined to a tenant | Restricted to that tenant |
| No restriction, public client | Confined to a tenant | Not a valid replacement |
| No restriction | Not confined to a tenant | No restriction |

The tenant rules of steps 1 and 4 of {{registration-requests}} also apply to any other request under the registration, including {{RFC7592}} read and delete requests.

## Public Clients {#public-client-registration}

An authorization server MAY register a client whose reviewed document declares `token_endpoint_auth_method` of `none`, creating a public-client registration. Such a registration has no client credential and is shared among installations of the software. A server that registers such a client:

* MUST reject the registration if the document lists no redirection URI.
* MUST issue tokens under the registration only through the authorization code grant and the refresh of grants it opens.
* MUST apply Section 5.6.2 of {{STATEMENT}} to every authorization request under the registration, and MUST require code redemption and every refresh of the resulting grant to prove the key bound there.
* MUST NOT issue a registration access token {{RFC7592}} for the registration.

## Repeated Registration {#repeated-registration}

A registration request is repeated if its statement has the `iss` and `sub` claims of the governing statement of an existing statement-governed registration, valid or not, that:

* is not restricted to another tenant ({{tenant-restriction}}), and
* is restricted to the request's tenant, if that tenant requires a tenant-scoped decision (Section 3.2 of {{STATEMENT}}).

One current statement can be consumed more than once, subject to the bounds of Sections 11.2 and 11.3 of {{STATEMENT}} and the server's registration policy. A server can count registrations toward the bound on establishments of Section 11.3 of {{STATEMENT}}, but the bound stays effective only if a registration under which no access token has been issued does not count toward it, since a holder of a copy of the statement can create one.

If the registration a request repeats is a confidential-client registration, local policy can reject the request, treat it as idempotent, or create another registration. An idempotent response MUST NOT return the existing registration's credentials, such as its `registration_access_token` member. A common default is one registration per `iss` and `sub`, per tenant on a multi-tenant server, at the cost {{statement-bearer}} describes.

If the registration a request repeats is a public-client registration ({{public-client-registration}}), the authorization server MUST NOT create another registration and handles the request by its statement, once the statement validates:

| Statement in the repeated request | Outcome |
| --- | --- |
| The governing statement, identified by its `iss` and `jti` | The existing registration is returned |
| A later `iat` than the governing statement's | Evaluated as a replacement ({{revalidation}}); the renewed registration is returned or the request is rejected ({{errors}}) |
| Any other statement with an `iat` no later than the governing statement's | The request is rejected |

# Registration Validity {#registration-validity}

The following validity model applies to every statement-governed registration:

* The registration's effective expiry (Section 2 of {{STATEMENT}}) is that of its governing statement.
* The registration is valid until its governing statement ceases to be current (Section 2 of {{STATEMENT}}), and so at most until that effective expiry.
* Once the registration has ceased to be valid, the authorization server MUST reject, at step 4 of {{registration-requests}}, authorization, token, and pushed authorization requests and {{RFC7592}} update requests under it, unless a replacement the request delivers has restored it at step 3.
* The authorization server SHOULD retain a registration that has ceased to be valid, so that a replacement can restore it.
* Ceasing to be valid does not expire the registration's credentials, including its registration access token, and does not by itself revoke access tokens already issued under the registration.
* Removing the governing statement's issuer from trust configuration, or narrowing what it can attest, does not by itself end the registration. A server ends it by its own means, such as deleting it.

If a request that carries no replacement is rejected because the registration has ceased to be valid, the rejection MUST NOT by itself cause the authorization server to revoke refresh tokens as it does on detecting refresh-token replay (Section 4.14.2 of {{RFC9700}}).

If the authorization server records a required issuer for a tenant (Section 4 of {{STATEMENT}}), it MUST reject an authorization, token, or pushed authorization request, or an {{RFC7592}} update request, in that tenant under any statement-governed registration whose governing statement's `sub` falls within a covered namespace, unless a current statement from the required issuer governs the registration as step 3 of {{registration-requests}} leaves it. This rule applies whatever tenant the registration was created in and whenever the requirement was configured. Without this rule, withholding the required issuer's statement would keep such a registration in use after the issuer withdrew or stopped renewing its decision.

Neither the validity model nor the required-issuer rule rejects {{RFC7592}} read and delete requests.

The `registration_expires_at` member is a NumericDate (Section 2 of {{RFC7519}}) giving the latest time the registration remains valid or, for a registration that has ceased to be valid, the time it ceased. The authorization server:

* MUST return `registration_expires_at` in the {{RFC7591}} registration response and in the successful response to any request that renews the registration or carries its governing statement ({{revalidation}}).
* SHOULD return it in any other {{RFC7592}} read or update response it supports.

# Requests Under a Registration {#registration-requests}

The authorization server MUST process an authorization, token, or pushed authorization request, or an {{RFC7592}} update request, under a statement-governed registration in the following order, stopping at the first step that rejects the request or treats it as from a client the server has not registered:

1. If the registration is restricted to a tenant ({{tenant-restriction}}) and the request, including one that delivers a statement, is made in another tenant, it MUST treat the request as from a client it has not registered.
2. It authenticates a token or pushed authorization request under the registration's own client authentication method, and an {{RFC7592}} update request as {{RFC7592}} requires, against the retained registration even if the registration has ceased to be valid. A token or pushed authorization request under a public-client registration has no client credential to authenticate ({{public-client-registration}}).
3. It classifies any statement the request delivers by its claims, before validating it, and evaluates a replacement ({{revalidation}}). A replacement applied at this step stands even if a later step, or the request's grant, rejects the request.
4. It applies the rejections of {{registration-validity}} to the registration as step 3 leaves it. If the request is made in a tenant where the authorization server requires a tenant-scoped decision (Section 3.2 of {{STATEMENT}}) and the registration, as step 3 leaves it, carries no tenant restriction, it MUST treat the request as from a client it has not registered.
5. It evaluates the request against the registration's metadata.

# Renewal {#revalidation}

A client renews a statement-governed registration by delivering a replacement statement in one of the following, authenticated as step 2 of {{registration-requests}} provides:

* The `software_statement` parameter of a token request under the registration.
* The `software_statement` parameter of a pushed authorization request {{RFC9126}} under the registration.
* The `software_statement` member of an {{RFC7592}} update request, if the deployment offers registration management.

A delivery is a `registration` use of the statement (Section 3 of {{STATEMENT}}). A registration request is not a delivery, and renews an existing registration only as {{repeated-registration}} provides for a public-client registration. A statement neither renews nor alters a registration that is not statement-governed, and runtime presentation (Section 5 of {{STATEMENT}}) alters no registration's metadata.

The authorization server treats a delivered statement as the first matching row of the following table provides:

| Statement delivered | Treatment |
| --- | --- |
| The governing statement itself, identified by its `iss` and `jti` | No replacement |
| In an {{RFC7592}} update request, any statement with the governing statement's `iss` and `sub` and an `iat` no later than the governing statement's | No replacement |
| At the token or pushed authorization request endpoint, any other statement with the governing statement's `iss` and `sub` and an earlier `iat` | Earlier than the governing statement. The request is rejected |
| A statement with the governing statement's `iss` and `sub` and a later `iat` | A replacement, evaluated below |
| Any other statement, such as one with another `iss` or `sub` | Fails the replacement rules below |

An {{RFC7592}} update request repeats the statement a previous response returned (Section 3.2.1 of {{RFC7591}}; Section 2.2 of {{RFC7592}}), which can be older than the governing statement. An {{RFC7592}} read or update response returns the governing statement as its `software_statement` member.

The replacement MUST satisfy the following:

* It validates under Section 3.2 of {{STATEMENT}}.
* The `iss` and `sub` claims equal those of the governing statement.
* The `iat` claim is later than that of the governing statement.
* Its document declares `token_endpoint_auth_method` of `none` if and only if the governing statement's document does.
* For a public-client registration that carries no tenant restriction, its decision is not confined to a tenant ({{tenant-restriction}}).
* Its document satisfies the rules of {{dcr-presentation}}, from which the registration's metadata is derived.

Renewing on the statement alone would leave in the registration a key, redirection URI, or scope that the new review removed.

A request whose replacement fails the rules above is rejected ({{errors}}) whether or not the registration has ceased to be valid. A rejected delivery leaves the governing statement unchanged.

When a replacement satisfies these rules, the authorization server MUST, in a single atomic update, make it the governing statement and give the registration the replacement's effective expiry, the metadata derived from its document ({{dcr-presentation}}), and any tenant restriction it adds ({{tenant-restriction}}). Concurrent deliveries resolve to the most recently issued statement. A valid replacement restores a registration that has ceased to be valid.

Access issued under the registration after that update, including on refresh of a grant issued before it, MUST fall within the re-derived metadata.

If every grant a confidential-client registration permits needs an authorization request or a refresh token, and the authorization server does not support {{RFC7592}} update requests for the registration, the server MUST support pushed authorization requests {{RFC9126}} for the registration. Otherwise such a client could not restore a registration that has ceased to be valid without a refresh token. A public-client registration also renews through a repeated registration request ({{repeated-registration}}).

# Error Responses {#errors}

A rejection this specification defines, at a client registration endpoint or at the client configuration endpoint of {{RFC7592}}, uses the error response of Section 3.2.2 of {{RFC7591}}, with the HTTP status code that section specifies, except that an authorization server SHOULD respond with HTTP status code 503 to a request it rejects with the `temporarily_unavailable` error code. Such a rejection uses the following error codes:

`invalid_software_statement`:
: Any of the following:

  * The statement is malformed, fails signature or claim validation for a reason `unapproved_software_statement` does not cover, is expired or otherwise not current (Section 2 of {{STATEMENT}}), or is superseded under an extension the server implements.
  * The document's digest does not equal the statement's `cimd_digest` claim ({{dcr-presentation}}).
  * An {{RFC7592}} update request is rejected under {{registration-validity}}.
  * A statement delivered at the client configuration endpoint, or the statement of a repeated public-client registration request evaluated as a replacement ({{repeated-registration}}), fails the replacement rules of {{revalidation}} where this section assigns no other code, such as a change in whether the document declares `token_endpoint_auth_method` of `none`.
  * The statement of a repeated public-client registration request is not the governing statement and has an `iat` no later than the governing statement's ({{repeated-registration}}).

`unapproved_software_statement`:
: The statement is well formed but not acceptable here: its issuer is not configured, its `aud` excludes this server, its `sub` is outside the client identifier namespaces the issuer can attest, its `aud_tenant` is not this request's tenant or is absent when required, its issuer is confined to another tenant, or its `statement_uses` excludes this use. This code is also used if a statement this server requires is absent from a registration request, if {{dcr-presentation}} forbids creating the registration because a required issuer covers the statement's `sub` and the statement is from another issuer, or if the server's policy refuses a repeated registration request ({{repeated-registration}}).

`invalid_client_metadata`:
: Any of the following holds for the reviewed document:

  * It contains duplicate member names (Section 5.2 of {{STATEMENT}}).
  * It fails validation as {{CIMD}} requires ({{dcr-presentation}}).
  * It carries metadata this server's policy refuses.
  * It cannot be obtained, including over the server's size bound, for a reason other than a retrieval that did not complete (Section 5.3 of {{STATEMENT}}).
  * It omits `token_endpoint_auth_method` ({{dcr-presentation}}).
  * It declares `token_endpoint_auth_method` of `none` and the server registers no public clients under {{public-client-registration}}.

`invalid_redirect_uri`:
: The reviewed document carries a redirection URI another application could claim ({{dcr-presentation}}), or declares `token_endpoint_auth_method` of `none` and lists no redirection URI ({{public-client-registration}}).

`temporarily_unavailable`:
: A retrieval the request needs did not complete (Section 5.3 of {{STATEMENT}}).

Rejections at other endpoints use the error responses of Section 6 of {{STATEMENT}}. At the token and pushed authorization request endpoints, a request that fails the authentication of step 2 of {{registration-requests}} is rejected with `invalid_client`. Subject to the order of Section 6 of {{STATEMENT}}, a token, pushed authorization, or authorization request is rejected with `statement_required` (Section 6.1 of {{STATEMENT}}) if:

* its registration has ceased to be valid, or the required-issuer rule rejects it ({{registration-validity}}), or
* its delivered statement is earlier than the governing statement or fails the replacement rules ({{revalidation}}).

At the authorization endpoint, the `redirect_uri` is validated against the registration's redirection URIs before an error is redirected (Section 4.1.2.1 of {{RFC6749}}).

Rejections of a registration management request for other reasons use the responses {{RFC7592}} specifies.

# Example {#example}

The following is a non-normative example of a registered client, whose `client_id` is `s6BhdRkqt3`, renewing its registration by delivering a replacement statement on an ordinary refresh, authenticated under its registered `private_key_jwt` method (line breaks for display purposes only):

~~~ http
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&client_id=s6BhdRkqt3
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjYtMDgifQ...
&software_statement=eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1l...
~~~

The authorization server validates the replacement and re-derives the registration's metadata ({{revalidation}}), returning the registration's new expiry in the `registration_expires_at` member of the token response. Once the registration has ceased to be valid, the same request without a replacement is rejected with `statement_required` ({{errors}}).

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} value:

`software_statement_registration_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server supports this specification. If omitted, the default value is `false`. A value of `true` means every registration the server creates from a statement of the profile in Section 3 of {{STATEMENT}} is statement-governed, and does not imply support for runtime presentation.

# Deployment Considerations {#deployment-considerations}

This section gives guidance to operators. It adds no requirement.

An issuer outage longer than a statement-governed registration's remaining lifetime makes the registration cease to be valid ({{registration-validity}}) until the client delivers a replacement ({{revalidation}}). The lifetimes an issuer chooses set the renewal cadence (Section 12.3 of {{STATEMENT}}). A deployment ought to have clients renew well before effective expiry, and ought to weigh that cadence against how quickly a narrowed or withdrawn review needs to take effect.

# Security Considerations {#security-considerations}

The security considerations of Section 13 of {{STATEMENT}} and Section 5 of {{RFC7591}} apply.

## Statements at Registration {#statement-bearer}

A statement consumed at registration is a reusable bearer artifact until it expires. Its audience, lifetime, and `statement_uses` claim (Section 3 of {{STATEMENT}}) and the registration policy of {{repeated-registration}} limit what a copied statement can create.

Taking every value from the document prevents metadata substitution. A request-supplied value for a member the document omits, such as a `jwks` member beside a document naming only `jwks_uri`, would give any holder of a copy a registration with reviewed branding and the holder's own key. Registering a document that omits `token_endpoint_auth_method` under the {{RFC7591}} default would register a shared-secret method, which {{CIMD}} forbids (Section 4.1 of {{CIMD}}) and the issuer never reviewed. An issuer that omits `aud` from a statement permitting registration accepts that any server trusting the issuer can register the software on the strength of a copy.

Under the one-registration default ({{repeated-registration}}), whoever first registers with a copy of a statement holds that registration and its registration access token, and can keep the software's publisher from registering there. Such a holder cannot authenticate as the client if the reviewed document's authentication method requires a key the document carries.

If the publisher's statement permits presentation and the server supports it, the publisher's software can still be admitted through runtime presentation (Section 5 of {{STATEMENT}}). Otherwise, recovery requires intervention by the server's operator or a registration policy other than the one-registration default. This specification defines no way to displace such a registration.

A public-client registration ({{public-client-registration}}) has no credential to take, and a repeated request is answered with it ({{repeated-registration}}), so its first registrant gains nothing over later ones. Any holder of a copy can use it. Authorization codes under it reach only the reviewed redirection URIs, and its grants are bound to keys that installations hold. A holder of a copy therefore gains what it would gain at runtime, including the exposure Section 13.4 of {{STATEMENT}} describes for a holder that controls the user agent.

## Credential Theft and Renewal {#renewal-credential}

Renewal proves possession of the registration's credential and the currency of a statement sharing the governing statement's `iss` and `sub`, not that the renewing party is the reviewed software. An attacker holding a stolen client credential can renew indefinitely with the newer statements distributed to every deployment of the software. A public-client registration's renewal proves only that the replacement is current. No protection is lost, since any party can already use that registration.

A narrower review takes effect at a registration when the client delivers it or when the governing statement ceases to be current. Withdrawal before then requires an extension (Section 9 of {{STATEMENT}}).

Deployments ought to pair statement-governed registrations with credential rotation, sender-constrained client authentication, and the bounds that Sections 11.2 and 11.3 of {{STATEMENT}} and {{repeated-registration}} describe, and ought to treat a credential compromise as requiring re-registration rather than renewal. Runtime presentation binds every presenter to the reviewed document (Sections 5.6 and 13.4 of {{STATEMENT}}) and so does not have this limitation.

## Registration Fraud and Impersonation {#registration-fraud}

Open registration permits `client_name`, `logo_uri`, and `client_uri` values that imitate trusted software on consent screens (Section 5 of {{RFC7591}}). Requiring a statement replaces self-asserted branding with issuer-reviewed values, though content served at a URL the document carries, such as a logo, can change after review (Section 13.1 of {{STATEMENT}}). Servers that render such values on consent screens ought to prefer those from a reviewed document and to apply heightened scrutiny to registrations that claim user-visible branding without a statement.

Requiring a statement also means a removed client cannot register again under a new client identifier URL without another issuer decision.

## Observable State {#oracle-considerations}

A retained registration that has ceased to be valid is distinguishable from an unknown client ({{registration-requests}}, {{errors}}). At the token and pushed authorization request endpoints, only a requester that authenticates as the registration's client can observe this, except for a public-client registration. At the authorization endpoint, anyone who sends an authorization request naming the client and one of its registered redirection URIs learns from the `statement_required` redirection that the registration has ceased to be valid. If an observer knows the governing statement's `exp`, a lapse before that time shows that an extension ended the statement or that the server caps the lifetime it honors for the issuer.

As at runtime (Section 13.8 of {{STATEMENT}}), these disclosures are accepted, and they reveal nothing of the server's issuer trust configuration beyond the fact and time that the registration ceased to be valid (Section 6.6 of {{STATEMENT}}). Publishing `software_statement_registration_supported` discloses that statement-governed registrations expire there, which a client needs to know before registering.

# Privacy Considerations

The privacy considerations of Section 14 of {{STATEMENT}} and Section 6 of {{RFC7591}} apply. A registration request reveals to the authorization server the same information as a runtime presentation.

# IANA Considerations {#iana}

## OAuth Dynamic Client Registration Metadata Registry

This specification requests registration of the following value in the IANA "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}. An authorization server returns this member in a registration response:

Client Metadata Name:
: `registration_expires_at`

Client Metadata Description:
: Latest time at which a statement-governed registration remains valid, as a NumericDate.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{registration-validity}}

## OAuth Parameters Registry

This specification requests registration of the following parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}, for the responses to requests that renew a statement-governed registration or carry its governing statement:

Parameter Name:
: `registration_expires_at`

Parameter Usage Location:
: token response, pushed authorization request response

Change Controller:
: IETF

Specification Document(s):
: This specification, {{registration-validity}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_registration_supported`

Metadata Description:
: Boolean value indicating whether the authorization server supports Client ID Metadata Document software statement registration.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-server-metadata}}

## OAuth Extensions Error Registry

This specification requests that IANA update the existing `temporarily_unavailable` entry, adding the client registration and client configuration endpoint error responses to its usage locations and this specification as a reference:

Error Name:
: `temporarily_unavailable`

Error Usage Location:
: client registration error response and client configuration endpoint error response, in addition to the locations already registered

Related Protocol Extension:
: OAuth 2.0 Authorization Framework

Change Controller:
: IETF

Specification Document(s):
: {{RFC6749}}, this specification ({{errors}})
