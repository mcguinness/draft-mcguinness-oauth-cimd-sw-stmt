---
title: "CIMD Software Statement Registration"
abbrev: oauth-cimd-sw-stmt-registration
docname: draft-mcguinness-oauth-cimd-sw-stmt-registration-latest
category: std

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
  RFC7636:
  RFC9126:
  RFC9449:
  RFC9700:
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "CIMD Software Statement"
    author:
      - fullname: Karl McGuinness
  CIMD: I-D.ietf-oauth-client-id-metadata-document

informative:
  RFC7592:

--- abstract

The CIMD Software Statement specification defines a software statement for a Client ID Metadata Document that carries no client metadata: it binds an issuer's review to the exact bytes of the client's own document. This specification defines how an authorization server consumes such a statement in an RFC 7591 registration request, taking every metadata value from the reviewed document rather than from the request. It also defines how the statement's expiry can bound the resulting registration, which RFC 7591 leaves unbounded, and how a client renews the registration by delivering a replacement statement, which also updates the registration's metadata.

--- middle

# Introduction

A software statement (Section 2.3 of {{RFC7591}}) carries a third party's approval of client software into a dynamic client registration request. In {{RFC7591}}, values in the statement take precedence over those in the request. A statement under {{STATEMENT}} carries no client metadata: it names a Client ID Metadata Document by its URL and binds the review to that document's exact octets. An {{RFC7591}} server that verified such a statement without this specification would take every metadata value from the request, attaching the issuer's approval to redirection URIs and keys nobody reviewed.

{{RFC7591}} also defines no expiry or renewal for a registration. A registration based on a statement can outlive it, and nothing keeps the registration's validity or metadata in line with the issuer's decision once the issuer narrows, withdraws, or stops renewing it.

{{STATEMENT}} admits a client at request time without registering it. This specification is for deployments that register clients through {{RFC7591}} instead, for example to assign their own client identifiers or to manage registrations under {{RFC7592}}. It defines the following:

* Consumption: the registration takes every metadata value from the reviewed document, never from the request ({{dcr-presentation}}).
* Validity: an authorization server can bound the registration by the statement's effective expiry, and advertise that it does ({{registration-validity}}).
* Renewal: the client extends the registration by delivering a replacement statement, which also brings the registration's metadata up to date with the new review ({{revalidation}}).

This specification never registers software whose document lists a redirection URI another application could claim ({{dcr-presentation}}). Such a document is review-only under {{STATEMENT}} and cannot satisfy a policy requiring reviewed software. A server can register a public client, whose document declares `token_endpoint_auth_method` of `none`, only under rules that bind it as {{STATEMENT}} binds a public client at runtime ({{public-client-registration}}). This specification depends on {{STATEMENT}}, which does not depend on it.

## Protocol Overview

The following non-normative sequence summarizes a statement-governed registration:

1. The client registers through {{RFC7591}}, carrying a statement in the `software_statement` member.
2. The authorization server derives the registration's metadata from the reviewed document and records the statement's identity. The registration is valid until the statement's effective expiry ({{registration-validity}}).
3. The client delivers a replacement statement in a token or pushed authorization request under the registration or, if supported, an {{RFC7592}} update request, extending the registration's validity to the replacement's effective expiry.
4. If no replacement arrives, the registration expires and requests under it fail.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology from {{RFC6749}} (OAuth), {{RFC7591}} (client metadata and software statements), and {{CIMD}} (Client ID Metadata Documents), and the terms Issuer, Runtime Presentation, Effective Expiry, Refusal Record, Current Statement, and Review-Only from {{STATEMENT}}.

This specification defines the following terms.

Delivery:
: Carrying a statement in a request under a statement-governed registration, in one of the locations {{revalidation}} lists, so that it can renew the registration. A delivery is not a runtime presentation.

Governing Statement:
: The statement that bounds a statement-governed registration: the one consumed at registration or, after a renewal, the replacement that renewed it most recently ({{revalidation}}).

Statement-Governed Registration:
: An {{RFC7591}} client registration to which the server applies the validity model of {{registration-validity}}: its validity is bound to a software statement's effective expiry, and replacement statements renew it.

# Consumption at Registration {#dcr-presentation}

The statement is consumed in the `software_statement` member of an {{RFC7591}} registration request. The authorization server validates it under the validation rules of {{STATEMENT}} and registers the client under its ordinary registration policy.

In {{RFC7591}}, values in a software statement take precedence over those in the request. A statement under {{STATEMENT}} carries no client metadata, so the reviewed document takes that role. An authorization server consuming a statement under this specification:

* MUST resolve the Client ID Metadata Document at the statement's `sub`, rejecting the registration if retrieval does not complete.
* MUST validate the document as {{CIMD}} requires, including that its `client_id` member matches the client identifier URL it is held for, which is the statement's `sub`.
* MUST verify that the digest of the retrieved representation equals `cimd_digest`, rejecting the registration if it does not, and MUST derive the registered metadata by parsing the same octets it digested, not a second retrieval or a cached copy it has not digested.
* MUST take every client metadata value from that document, and MUST NOT take any client metadata value from the registration request, or from any {{RFC7592}} update request for the registration it creates, whether or not the document carries that member:
  * The registration's metadata changes only by re-derivation from the document a replacement statement reviews ({{revalidation}}).
  * A request MAY carry metadata, as {{RFC7591}} clients do, but it does not contribute to the registration.
  * The registered `client_id` is assigned as {{RFC7591}} provides, and the document's own `client_id` member is the URL recorded as `sub`.
* MUST reject the registration with `invalid_client_metadata` if the document omits `token_endpoint_auth_method`, which {{STATEMENT}} requires a reviewed document to declare. Registering under the {{RFC7591}} default, a shared-secret method that {{CIMD}} forbids, would issue a client secret to any holder of a copy of the statement.
* MUST reject the registration with `invalid_client_metadata` if the document declares `token_endpoint_auth_method` of `none`, unless it registers the client under {{public-client-registration}}. Such software can otherwise use runtime presentation if it meets the public-client binding requirements of {{STATEMENT}}.
* MUST reject the registration with `invalid_client_metadata` if any of the document's redirection URIs could be claimed by another application on the same device (see the public-client presentation rules of {{STATEMENT}}), whatever authentication method the document declares.

If the statement's decision is confined to one of the authorization server's tenants, by its `aud_tenant` claim or by the server's trust configuration for its issuer ({{STATEMENT}}), the authorization server MUST record that tenant with the registration, and MUST treat a request under the registration in any other tenant, including a delivery ({{revalidation}}), as a request from a client it has not registered. The restriction persists across renewals, and a renewal by a replacement whose decision is confined to a tenant restricts the registration to that tenant. A registration created from a statement whose decision is not confined to a tenant carries no tenant restriction under this specification until such a renewal.

Unlike runtime presentation ({{STATEMENT}}), registration has no review-only form: the registration is the admission.

Taking every value from the document prevents metadata substitution. At registration a statement is a bearer artifact, so a request-supplied value for a member the document omits, such as a `jwks` beside a document naming only `jwks_uri`, would give any holder of a copy a registration with reviewed branding and its own key.

The external-retrieval protections and resource bounds of {{STATEMENT}} apply to retrieving the document as to a runtime presentation. The authorization server completes every check of {{STATEMENT}} that needs no client-controlled retrieval before resolving the document.

## Public Clients {#public-client-registration}

An authorization server MAY register a client whose reviewed document declares `token_endpoint_auth_method` of `none`. Such a registration has no client credential. Every installation of the software registers and receives the same registration and `client_id`, and so can any holder of a copy of the statement. The registration is bound instead as {{STATEMENT}} binds a public client at runtime: authorization codes reach only the reviewed redirection URIs, and each grant is bound to a key the installation holds. A server that registers such a client:

* MUST reject the registration with `invalid_client_metadata` if the document lists no redirection URI. {{dcr-presentation}} already rejects a document listing one that another application could claim.
* MUST issue tokens under the registration only through the authorization code grant and refresh of the grants it opens.
* MUST require PKCE {{RFC7636}} with the `S256` method on every authorization request under the registration.
* MUST require every authorization request under the registration to bind a key the client holds, through the `dpop_jkt` parameter or, at the pushed authorization request endpoint, a DPoP proof (Section 10.1 of {{RFC9449}}), and MUST require code redemption and every refresh of the resulting grant to prove that key.
* MUST apply the validity model of {{registration-validity}} to the registration, whether or not it advertises the model.
* MUST NOT issue a registration access token {{RFC7592}} for the registration.
* MUST answer a later registration request with the existing registration if the request is in a tenant where the registration can be used and its statement has the same `iss` and `sub`, validates, is not superseded under the `iat` floor of {{STATEMENT}}, and is confined to no tenant other than one the registration is already restricted to. If that statement's `iat` is later than the governing statement's, the server evaluates it as {{revalidation}} provides for a delivery, renewing the registration with it or rejecting the request with the codes of {{errors}}.

Such a registration has no {{RFC7592}} management, and the deliveries that {{revalidation}} lists at the token and pushed authorization request endpoints need no client authentication under it, since it has none. A delivery renews it only with a valid replacement, so an unauthenticated delivery can apply only a statement the issuer signed. A statement whose decision is confined to a tenant is not a replacement for such a registration if the registration carries no tenant restriction. Without a registration access token, the registration ends only at its effective expiry, on withdrawal ({{registration-validity}}), or by the server's own policy.

## Registration Validity {#registration-validity}

To let a client rely on the signal ({{authorization-server-metadata}}) before it registers, an authorization server that advertises `software_statement_registration_validity_supported` as `true` MUST apply the following validity model to every registration it creates from a statement validated under {{STATEMENT}}, whichever issuer signed it:

* The authorization server MUST record, with the registration, the governing statement's `iss`, `jti`, `sub`, `iat`, and any claim by which a withdrawal mechanism locates that statement, such as `status`, together with the registration's effective expiry. That effective expiry is the governing statement's effective expiry ({{STATEMENT}}), what a renewal extends, and what `registration_expires_at` reports.
* The registration is valid until that effective expiry.
* Once the effective expiry passes without a replacement ({{revalidation}}), the authorization server MUST reject authorization, token, and pushed authorization requests and {{RFC7592}} update requests under the registration, other than the revalidation requests {{revalidation}} permits:
  * They are rejected with `statement_required` ({{STATEMENT}}), except that an {{RFC7592}} update request is rejected with `invalid_software_statement`, since the governing statement is expired or refused.
  * An authorization request carrying the `software_statement` parameter is rejected with `invalid_request` instead, as {{STATEMENT}} provides.
  * {{RFC7592}} read and delete requests remain available for a client to learn its registration's state and end it.
  * At the token and pushed authorization request endpoints, the client is first authenticated against the retained registration, and a request that fails that authentication is rejected with `invalid_client`, as for an unknown client.
* The authorization server SHOULD retain the expired record for a later authenticated revalidation ({{oracle-considerations}}). A valid replacement restores the registration ({{revalidation}}), so no grace period is needed.

At the authorization endpoint, the rejection is returned in the authorization error response {{RFC6749}}. The authorization server redirects with it only after validating the request's `redirect_uri` against the retained registration. If it cannot do so, it MUST NOT redirect and reports the error to the resource owner instead. The client then delivers a replacement in a pushed authorization request ({{revalidation}}).

If the server holds a refusal record for the governing statement ({{STATEMENT}}), the registration ceases to be valid as it does at its effective expiry, and {{revalidation}} is the recovery path. The client learns of the withdrawal only from the rejection. The disposition of outstanding grants is local policy.

A server applying this model that uses a withdrawal mechanism for the governing statement's issuer SHOULD consult it for that statement at each token request under the registration, as {{STATEMENT}} provides for refresh.

An authorization server applying this model MUST publish `pushed_authorization_request_endpoint`. Without that endpoint, {{revalidation}} leaves a client whose only grant type is the authorization code, and which holds no refresh token, with no way to renew.

A server applying this model MUST return a `registration_expires_at` member, a NumericDate (Section 2 of {{RFC7519}}) giving the latest time the registration remains valid, in the {{RFC7591}} registration response and in the response to any request that renews the registration or carries its governing statement ({{revalidation}}), and SHOULD return it in any {{RFC7592}} read or update response it supports. For a registration that has ceased to be valid, the value is the time it ceased. The server can cap the lifetime it honors for the issuer ({{STATEMENT}}), so a client cannot compute this boundary from the statement alone.

A server that omits the signal or advertises `false` does not bound registrations by statement expiry, except as {{public-client-registration}} and the next paragraph require, but still consumes the statement as {{dcr-presentation}} requires.

A server that records a required issuer for a tenant ({{STATEMENT}}) MUST apply this validity model to every registration it creates in that tenant for software in a covered namespace, whether or not it advertises the model. It MUST reject a request in that tenant under any registration for software in a covered namespace, whatever tenant the registration was created in and whenever the requirement was configured, unless a current statement from the required issuer governs the registration once any replacement the request delivers has been evaluated ({{revalidation}}). The rejection is `statement_required` ({{STATEMENT}}), returned as for an expired registration. Without these rules, withholding the required issuer's statement would keep such a registration in use after the issuer withdrew or stopped renewing its decision. A registration governed by another issuer's statement cannot satisfy the requirement, since a replacement keeps the governing statement's `iss`.

## Revalidation {#revalidation}

A client renews a statement-governed registration by delivering a replacement statement in one of the following:

* The `software_statement` parameter of a token request under the registration, authenticated as the registered client under the registration's own method.
* The `software_statement` parameter of an authenticated pushed authorization request {{RFC9126}} under the registration.
* The `software_statement` member of an authenticated {{RFC7592}} update request, if the deployment offers registration management.

The update request carries the complete client metadata that {{RFC7592}} requires, which the server does not take ({{dcr-presentation}}).

A statement with the governing statement's `iss` and `sub` and an `iat` no later than the governing statement's `iat` is not a replacement. The authorization server leaves the registration unchanged and treats a request as carrying no replacement in either of the following cases, so that the request proceeds while the registration is valid and is rejected as {{registration-validity}} provides once it is not:

* The request carries the governing statement itself, identified by its `iss` and `jti`. A client sends it, for example, after losing the response to the request that renewed the registration, or by attaching its current statement to later requests.
* The request is an {{RFC7592}} update request, which repeats, unmodified, the statement that a registration, read, or update response returned (Section 3.2.1 of {{RFC7591}}; Section 2.2 of {{RFC7592}}). After a token or pushed authorization request has renewed the registration, that statement can be older than the governing statement.

Any other statement older than the governing statement is superseded under the `iat` floor of {{STATEMENT}}, and a request carrying it at the token or pushed authorization request endpoint is rejected with `statement_required`.

An {{RFC7592}} read or update response returns the governing statement as its `software_statement` member.

The replacement MUST satisfy the following:

* It validates under the validation rules of {{STATEMENT}}, including its audience if it carries one.
* The `iss` and `sub` claims equal those of the governing statement.
* The `iat` claim is later than that of the governing statement.
* Its document declares `token_endpoint_auth_method` of `none` if and only if the governing statement's document does.

For a replacement, the authorization server MUST do the following:

* Obtain the Client ID Metadata Document at the replacement's `sub` by retrieval, as {{dcr-presentation}} requires.
* Verify that the digest of those octets equals the replacement's `cimd_digest`.
* Re-derive the registration's metadata from those same octets, under the rules of {{dcr-presentation}}.

Renewing on the statement alone would leave in the registration a key, redirection URI, or scope that the new review removed.

On success, the replacement becomes the governing statement: the authorization server MUST replace the recorded identity, `iat`, effective expiry, and derived metadata with the replacement's in a single atomic update, and concurrent deliveries resolve to the most recently issued statement.

Access issued under the registration after that update, including on refresh of a grant issued before it, MUST fall within the re-derived metadata, as {{STATEMENT}} requires of a refreshed grant.

When a request under an expired registration contains a replacement, the authorization server MUST authenticate the client against the retained registration and evaluate the replacement before applying the expiry rejection. A valid replacement therefore restores the registration. An omitted or invalid replacement does not.

The rejection of a request under an expired registration that carries no replacement ({{registration-validity}}) MUST NOT by itself cause the authorization server to revoke refresh tokens as it does on detecting refresh-token replay (Section 4.14.2 of {{RFC9700}}). The client recovers by delivering a valid replacement.

A request whose replacement fails the rules above is rejected whether or not the registration has expired. The client then learns of the failure at once and can tell a bad replacement from a missing one. Such a request is rejected with `statement_required` at the token and pushed authorization request endpoints, or with `temporarily_unavailable` there if a retrieval needed to check the replacement did not complete for a reason other than the size bound of {{STATEMENT}}, and with the codes of {{errors}} at the registration management endpoint. A rejected delivery leaves the governing statement unchanged.

A registration request without an {{RFC7592}} registration access token creates a new registration and never renews an existing one, except for a public-client registration ({{public-client-registration}}).

# Statements from a Registered or Admitted Client {#registered-delivery}

A client registered through {{RFC7591}}, or admitted under its Client ID Metadata Document URL as its `client_id`, can still carry a statement. Runtime presentation ({{STATEMENT}}) does not alter a registration's metadata. At the token and pushed authorization request endpoints, a statement carried under a registration that is neither a runtime presentation nor a delivery ({{revalidation}}) is rejected with `invalid_request` ({{STATEMENT}}). At the registration management endpoint, a statement from an issuer the server does not accept for the client is rejected with `unapproved_software_statement` ({{errors}}).

For a statement-governed registration, a statement carried in a location {{revalidation}} lists is a delivery: one from the governing issuer can renew the registration, and one from another issuer fails the replacement rules of {{revalidation}}. While the registration is valid, its governing statement is current, even after a later statement has raised the `iat` floor of {{STATEMENT}}, and satisfies any requirement the server has for a current statement on refresh.

If the registration is not statement-governed, a statement neither renews nor alters it and is not a delivery. For such a registration, the server applies a reviewed change, if at all, through its own registration policy, deriving metadata only from a reviewed document ({{dcr-presentation}}).

# Repeated Registration {#repeated-registration}

One unexpired statement can be consumed more than once, within the bounds of {{STATEMENT}}. The safe default is one registration per (`iss`, `sub`) at one authorization server. On repeated consumption, local policy can reject the request, treat it as idempotent, or create another registration. A repeated public-client registration is answered with the existing registration as {{public-client-registration}} provides. An idempotent response MUST NOT return the existing registration's credentials, such as its `registration_access_token`. A repeated request can come from any holder of a copy of the statement.

# Error Responses {#errors}

A rejection this specification defines, at a registration endpoint or at the registration management endpoint of {{RFC7592}}, uses the error response of Section 3.2.2 of {{RFC7591}}, with the HTTP status code that section specifies. An authorization server SHOULD respond with HTTP status code 503 to a request it rejects with `temporarily_unavailable`.

Rejections of a registration management request for other reasons use the responses {{RFC7592}} specifies. Rejections at other endpoints use the error responses of {{STATEMENT}}, and registration expiry is reported as {{registration-validity}} defines.

An authorization server rejecting a registration request, or a registration management request carrying a failing replacement, uses the following error codes:

`invalid_software_statement`:
: The statement is malformed, fails signature or claim validation, or is expired, refused by a refusal record, or superseded under the `iat` floor of {{STATEMENT}}. The code also applies when the digest of the retrieved document does not match the statement's `cimd_digest`.

`unapproved_software_statement`:
: The statement is valid but not acceptable here: its issuer is not configured, its `aud` excludes this server, its `sub` is outside the issuer's scope, its `aud_tenant` is not this request's tenant or is absent when required, or its `statement_uses` excludes this use. This code is also used if a statement this server requires is absent.

`invalid_client_metadata`:
: Any of the following holds for the reviewed document:

  * It contains duplicate member names ({{STATEMENT}}).
  * Its `client_id` member does not match the statement's `sub` ({{dcr-presentation}}).
  * It carries metadata this server's policy refuses.
  * It exceeds the server's size bound ({{STATEMENT}}), so that retrying cannot succeed.
  * It omits `token_endpoint_auth_method` ({{dcr-presentation}}).
  * It declares `token_endpoint_auth_method` of `none` and the server does not register it under {{public-client-registration}}.
  * It declares `token_endpoint_auth_method` of `none` and lists no redirection URI ({{public-client-registration}}).
  * It carries a redirection URI another application could claim ({{dcr-presentation}}).

`temporarily_unavailable`:
: A retrieval the request needs did not complete, for a reason other than the size bound.

# Example {#example}

## Revalidating a Statement-Governed Registration

The following non-normative example shows a registered client, `client_id` `s6BhdRkqt3`, renewing its registration by delivering a replacement statement on an ordinary refresh, authenticated under its registered `private_key_jwt` method (line breaks are for display purposes only):

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

The authorization server validates the replacement and re-derives the registration's metadata ({{revalidation}}), returning the new validity boundary in the `registration_expires_at` member of the token response. After expiry, the same request without a replacement would fail with `statement_required` ({{registration-validity}}).

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} value:

`software_statement_registration_validity_supported`:
: OPTIONAL. Boolean value indicating whether every registration the authorization server creates from a statement validated under {{STATEMENT}} is governed by the validity and revalidation rules of {{registration-validity}} and {{revalidation}}. If omitted, the default value is `false`. A value of `true` does not imply support for runtime presentation.

# Security Considerations {#security-considerations}

## Statements at Registration {#statement-bearer}

A statement consumed at registration is a reusable bearer artifact until it expires. Its audience, lifetime, and `statement_uses` claim ({{STATEMENT}}) and the bounds of {{repeated-registration}} limit what a stolen statement can create.

Under the one-registration default ({{repeated-registration}}), whoever first registers with a copy of a statement holds that registration and its registration access token, and can keep the software's publisher from registering at that server. Such a holder cannot authenticate as the client if the reviewed document's authentication method requires a key the document carries.

If the publisher's statement permits presentation and the server supports it, the publisher's software can still be admitted through runtime presentation ({{STATEMENT}}). Otherwise, recovery requires the server's operator to intervene or a registration policy other than the one-registration default. This specification defines no way to displace such a registration.

A public-client registration ({{public-client-registration}}) has no credential to take and is answered idempotently, so its first registrant gains nothing over later ones. Any holder of a copy can use it, as any party can use a public client's identifier. Its codes reach only the reviewed redirection URIs and its grants are bound to keys that installations hold, so a holder of a copy gains what it would gain at runtime, including the exposure the copied-statement considerations of {{STATEMENT}} describe for a holder that controls the user agent.

## Renewal Authenticates the Credential

Renewal proves possession of the registration's credential and the currency of a statement sharing the governing `iss` and `sub`, not that the renewing party is the reviewed software. An attacker holding a stolen client credential can renew indefinitely with any current statement for that software. Such statements circulate to every deployment of it. A public-client registration has no credential, so its renewal proves only that the replacement is current. This loses nothing, since any party can already use that registration.

Deployments should pair statement-governed registrations with credential rotation, sender-constrained client authentication, and the repeated-consumption limits of {{STATEMENT}} and {{repeated-registration}}, and should treat a credential compromise as requiring re-registration rather than renewal. Runtime presentation ({{STATEMENT}}) binds every presenter to the reviewed document and so does not have this limitation.

## Registration Fraud and Impersonation {#registration-fraud}

Open registration permits `client_name`, `logo_uri`, and `client_uri` values that imitate trusted software on consent screens (Section 5 of {{RFC7591}}). Requiring a statement replaces self-asserted branding with issuer-reviewed values. Servers that render such values on consent screens should prefer those from a reviewed document and should apply heightened scrutiny to registrations that claim user-visible branding without a statement.

Requiring a statement also means each new client identity needs another issuer decision, so a discarded client cannot return at no cost. A statement does not make metadata true: a client that misleads review can obtain a genuine one for fraudulent metadata, so a statement is only as reliable as the issuer's verification.

## Observable State {#oracle-considerations}

A retained expired registration is distinguishable from an unknown client ({{revalidation}}). At the token and pushed authorization request endpoints, only a requester that authenticates as the registration's client can observe this, and what it learns is the state it must act on. A public-client registration has no credential, so anyone can observe its state there, as anyone can at the authorization endpoint.

At the authorization endpoint, anyone who can send an authorization request naming the client and one of its registered redirection URIs receives the `statement_required` redirection ({{registration-validity}}), and so learns that the registration has lapsed. If an observer knows the governing statement's `exp`, a lapse before it shows that the server either holds a refusal record for the statement or caps the lifetime it honors for the issuer.

The disclosures above are accepted for the reason {{STATEMENT}} gives for its own `statement_required` refusal: a client that cannot learn why it was refused cannot act. Servers publishing `software_statement_registration_validity_supported` also disclose that statement-derived registrations expire there, which a client needs to know before registering. Beyond a lapse and its timing, these disclosures reveal nothing of the server's issuer trust configuration, which {{STATEMENT}} keeps from unauthenticated requesters.

# Privacy Considerations

A registration request reveals to the authorization server what a runtime presentation does ({{STATEMENT}}).

# IANA Considerations {#iana}

## OAuth Dynamic Client Registration Metadata Registry

This specification requests registration of the following client metadata member, returned by an authorization server in a registration response:

Client Metadata Name:
: `registration_expires_at`

Client Metadata Description:
: Latest time at which a statement-governed registration remains valid, as a NumericDate. A withdrawal can end it earlier. Returned in a client registration response, in a registration management response, and in the token or pushed authorization request response to a request that renews the registration or carries its governing statement.

Change Controller:
: IESG

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

This specification requests registration of the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

Metadata Name:
: `software_statement_registration_validity_supported`

Metadata Description:
: Boolean value indicating whether every registration created from a validated software statement is governed by the statement validity and revalidation rules of this specification.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

## OAuth Extensions Error Registry

This specification requests that IANA add this specification as a reference for the `temporarily_unavailable` error and extend its usage location to include the client registration error response:

Error Name:
: `temporarily_unavailable`

Existing Registration:
: {{RFC6749}}

Error Usage Location:
: client registration error response, in addition to the locations already registered

Change Controller:
: IETF

Specification Document(s):
: {{RFC6749}}, this specification ({{errors}})

--- back

# Acknowledgments
{:numbered="false"}

This specification was separated from {{STATEMENT}}, where registration consumption was first defined, so that runtime admission could be implemented without it.
