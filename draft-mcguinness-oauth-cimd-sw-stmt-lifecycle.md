---
title: "Client ID Metadata Document Software Statement Lifecycle"
abbrev: oauth-cimd-sw-stmt-lifecycle
docname: draft-mcguinness-oauth-cimd-sw-stmt-lifecycle-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Withdrawal
 - Supersession
 - Refresh Token
 - Status List
 - Shared Signals
 - Security Event Token

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC8935:
  RFC8936:
  RFC8414:
  RFC8417:
  RFC9493:
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "Client ID Metadata Document Software Statement"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-00
  STATUSLIST: I-D.ietf-oauth-status-list
  SSF:
    target: https://openid.net/specs/openid-sharedsignals-framework-1_0.html
    title: "OpenID Shared Signals Framework Specification 1.0"
  CAEP:
    target: https://openid.net/specs/openid-caep-1_0.html
    title: "OpenID Continuous Access Evaluation Profile 1.0"

informative:
  RFC6755:
  RFC7009:
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "Client ID Metadata Document Software Statement Registration"
    author:
      - fullname: Karl McGuinness
    date: 2026-10
    seriesinfo:
      Internet-Draft: draft-mcguinness-oauth-cimd-sw-stmt-registration-00

--- abstract

A software statement for a Client ID Metadata Document (CIMD) records a trusted issuer's review of a client's metadata. The base specification ends no grant at a statement's expiry, keeps an older unexpired statement admissible after a newer one is issued, and evaluates no statement on refresh. This specification lets an issuer withdraw a statement, and an authorization server that learns of it keeps a refusal record. An `iat` floor supersedes an issuer's earlier statements about the same software once a server accepts a later one. Refresh is refused for a grant whose statement is withdrawn or, by recommendation, expired, unless a later statement replaces it. A profile of Token Status List is the withdrawal mechanism defined here, and an optional Shared Signals event prompts servers to check status promptly.

--- middle

# Introduction

{{STATEMENT}} defines a software statement by which a configured issuer records its review of the exact octets of a Client ID Metadata Document (CIMD) {{CIMD}}, and how an authorization server admits a client and binds a grant on that review. It leaves out what follows admission: expiry ends no grant already opened, an issuer cannot withdraw a statement before it expires, an older unexpired statement remains admissible after a newer one is issued, and refresh evaluates no statement.

This specification adds withdrawal and refusal records ({{withdrawal}}), supersession of earlier statements through an `iat` floor ({{supersession}}), and refresh rules ({{refresh-lifecycle}}) under which a grant whose recorded statement is withdrawn, or by recommendation expired, continues only if a later statement replaces it.

A trusting authorization server that implements this specification applies all of {{withdrawal}} through {{lifecycle-errors}}, whatever withdrawal mechanism it uses, if any. An issuer that uses a withdrawal mechanism withdraws statements through it, and {{withdrawal}} recommends that it withdraw an earlier statement when it issues a later, narrower one. Publishing status and sending events are optional for an issuer, and supersession and refresh replacement work without them. A client can offer a later statement on refresh ({{refresh-replacement}}).

{{token-status-list}} profiles Token Status List {{STATUSLIST}} as the withdrawal mechanism of this specification: an issuer withdraws a statement by setting its status, and a trusting authorization server learns of it when it next resolves the list. {{shared-signals}} defines an optional Shared Signals Framework {{SSF}} event, not itself a withdrawal mechanism, that prompts the trusting authorization servers that have configured an issuer to resolve a changed status promptly.

{{STATEMENT}} does not depend on this specification. Registrations under {{REGISTRATION}} advance the `iat` floor, and a refusal record ends a registration's validity at the server that holds it. This specification defines no new endpoint.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses the terms Transmitter, Receiver, and Stream from {{SSF}}, Security Event Token (SET) from {{RFC8417}}, Status List Token from {{STATUSLIST}}, and the terms defined in Section 2 of {{STATEMENT}}.

This specification defines the following terms.

Withdrawal Mechanism:
: The means by which a trusting authorization server learns that an issuer has withdrawn a statement before its expiry ({{withdrawal}}), such as the Token Status List profile of {{token-status-list}}.

Refusal Record:
: State an authorization server holds when it has learned that a statement ceased to be acceptable before its expiry, whether from a withdrawal mechanism it uses for the issuer ({{withdrawal}}) or from its own operator.

Recorded Statement:
: The statement an establishment records: the one presented or pulled to create it or, after a refresh accepts a replacement, the replacement most recently accepted for it ({{refresh-replacement}}).

`iat` Floor:
: For a trusting authorization server, the greatest `iat` claim value among the statements it has accepted for a given `iss` and `sub`, per tenant on a multi-tenant server ({{supersession}}).

This specification also extends the following term of {{STATEMENT}}.

Current Statement:
: As Section 2 of {{STATEMENT}} defines it. Under this specification, a refusal record ends a statement, so a statement that a refusal record covers is not current ({{withdrawal}}). The `iat` floor ({{supersession}}) separately limits which current statements a server accepts.

# Withdrawal and Refusal Records {#withdrawal}

An issuer can withdraw a statement before its `exp`. Unlike OAuth token revocation ({{RFC7009}}), a withdrawal revokes no issued token: a server holding a refusal record refuses new admissions and refresh under the statement ({{currency}}, {{open-grants}}).
{: #lifecycle}

How a trusting authorization server learns of a withdrawal is a withdrawal mechanism. This specification requires no withdrawal mechanism. It defines one, the Token Status List profile of {{token-status-list}}, and other specifications can define how another mechanism applies to these statements.

In addition to the checks of Section 3.2 of {{STATEMENT}}, a trusting authorization server MUST reject a statement for which it holds a refusal record. A server that uses a withdrawal mechanism for the statement's issuer SHOULD consult it before accepting the statement for a new admission, within the caching and freshness rules that mechanism defines.

A refusal derived from a withdrawal mechanism lasts until a later authoritative result from that mechanism replaces it, or until an expiry the mechanism itself defines for the withdrawal. A server MUST NOT retain such a refusal once it is replaced or expired in that way. A failure to consult the mechanism, or cached evidence that has lapsed, neither replaces nor expires a refusal.

A withdrawal constrains and never relaxes. A statement for which no withdrawal mechanism reports a withdrawal, or for which the server cannot consult the mechanism it uses ({{withdrawal-practice}}), is bounded by its effective expiry as it would be otherwise, and a server MUST NOT treat any withdrawal mechanism as grounds to accept a statement past its effective expiry.

An issuer that uses a withdrawal mechanism and issues a later statement narrower than the one it replaces SHOULD withdraw the earlier statement. A later statement is narrower if the document it names no longer carries a key, redirection URI, or scope the earlier document carried, if its `aud`, `aud_tenant`, or `statement_uses` is narrower, or if its remaining lifetime is shorter.

Without that withdrawal, a trusting authorization server that has not yet seen the later statement, or at which the later statement does not validate, accepts the earlier statement until it expires.

Rejections under this section use the error responses of {{lifecycle-errors}}.

# Supersession {#supersession}

Under {{STATEMENT}}, one unexpired statement is consumable more than once, and an older statement that has not expired remains admissible after a newer one is issued (Section 3.3 of {{STATEMENT}}). Because the `iat` claim values of an issuer's statements for a given `iss` and `sub` increase strictly (Section 3 of {{STATEMENT}}), a later `iat` marks a later review. Supersession ends the admissibility of earlier statements once a server accepts a later one: a statement whose `iat` claim is earlier than the server's `iat` floor is superseded.

A trusting authorization server MUST retain, per `iss` and `sub` claims and, on a multi-tenant server, per tenant, the greatest `iat` claim value among the statements it has accepted, as its `iat` floor, for at least the maximum statement lifetime it honors for the issuer after that `iat`. A statement is accepted for the floor once the server admits a client on it, applies it as a replacement ({{refresh-replacement}}), or registers a client with it ({{REGISTRATION}}). The server MUST reject a statement whose `iat` claim is earlier than the retained `iat`, whether it arrives as a replacement, a new registration, or a new presentation. A client holding a superseded, broader statement therefore cannot defer a narrowing by opening a fresh registration or establishment with it.

At admission, the server verifies the `iat` floor in step 3 of Section 5.1 of {{STATEMENT}}, along with the statement requirements and the `client_id` rule, before it resolves the reviewed document.

In selecting a pulled statement (Section 5.5 of {{STATEMENT}}), the server also ignores any statement that the `iat` floor supersedes. A pulled statement the server does not apply does not advance the `iat` floor. Nor does a review-only presentation or pulled statement (Section 5.6.3 of {{STATEMENT}}).

The first presentation of a later statement that establishes the client supersedes its predecessor for new admissions by every instance of the software at that server ({{instance-practice}}). Grants the predecessor already opened continue under {{refresh-lifecycle}}.

# Refresh {#refresh-lifecycle}

Under {{STATEMENT}}, refresh evaluates no statement: it checks the establishment's proven key (Section 5.7.1 of {{STATEMENT}}). This section extends refresh so that a grant can depend on a current statement.

## Requiring a Current Statement {#currency}

When the establishment's recorded statement has expired, the authorization server SHOULD require a current statement on refresh-token use ({{open-grants}}). The recorded statement satisfies a requirement for a current statement while it is current, even after a later statement has raised the `iat` floor ({{supersession}}). A server that uses a withdrawal mechanism for the recorded statement's issuer SHOULD consult it for that statement at each refresh.

The requirement for a current statement from a required issuer (Section 4 of {{STATEMENT}}) also applies on refresh.

If the authorization server holds a refusal record for the recorded statement, it MUST reject a refresh without an accepted replacement ({{refresh-replacement}}), even if it does not otherwise require a current statement. If a required issuer covers the establishment's `sub` in the establishment's tenant, the server MUST reject a refresh unless the recorded statement or an accepted replacement is a current statement from that issuer. A grant recorded under another issuer's statement cannot meet this, so its client opens a new grant by presentation.

## Refresh Replacement {#refresh-replacement}

A refresh replacement is a later statement from the recorded statement's issuer about the same software, offered or pulled on refresh to take the recorded statement's place. It is a `presentation` use of the statement (Section 3 of {{STATEMENT}}). A server that does not implement this specification ignores an offered later statement (Section 5.4 of {{STATEMENT}}).

A request carrying a refresh replacement in the `software_statement` parameter (Section 5.4 of {{STATEMENT}}) MUST include the `client_id` parameter, even if the client authenticates, and its value, the client's CIMD URL, MUST exactly equal the statement's `sub` claim. If the `client_id` parameter is absent, the request is rejected. A refresh replacement whose `sub` differs is rejected as step 3 below provides.

On refresh-token use under an establishment, the authorization server first completes the checks of Section 5.7.1 of {{STATEMENT}} on the establishment's proven key, unchanged, before it considers any replacement. A server that implements this specification then proceeds as follows. A refresh rejected at any step leaves the grant's state unchanged.

1. Decide whether a replacement is needed. One is needed in either of the following cases, and otherwise none is, though the client can still offer one:

   * A refusal record covers the recorded statement.
   * The recorded statement has expired, and the server requires a current statement under {{currency}} or because a required issuer (Section 4 of {{STATEMENT}}) covers the establishment's `sub` in the establishment's tenant.

2. Obtain a replacement. A client can offer one on any refresh in the `software_statement` parameter. A statement with the recorded `iss` and `jti` is not a replacement and is treated as the recorded statement. When a replacement is needed and the request carries none, a server that supports pulled statements pulls, however the establishment was created, and takes as the replacement the newest statement with the recorded `iss` and `sub` among those it considers under Section 5.5 of {{STATEMENT}} and {{supersession}}. A refresh that needs a replacement and obtains none is rejected: a reviewed grant never continues under the policy for clients the server has not reviewed.

   The pull follows the retrieval rules of Sections 5.3 and 5.5 of {{STATEMENT}}. Only a request opening a new grant proceeds after a pull that does not complete, so a refresh whose pull does not complete is rejected.

3. Validate the replacement. A refresh whose replacement fails any of the following requirements is rejected, even when no replacement was needed. The replacement:

   * MUST validate under Section 3.2 of {{STATEMENT}} and {{withdrawal}}, including its audience if it carries one.
   * MUST have the recorded statement's `iss` and `sub`.
   * MUST have an `iat` later than the recorded statement's.
   * MUST name in `cimd_digest` the document currently served at that `sub`, which the server confirms by retrieval or by revalidating octets it holds.
   * MUST name a document that carries the establishment's proven key, in its `jwks` or at its `jwks_uri` (Section 5.6.1 of {{STATEMENT}}), or, for an establishment created under Section 5.6.2 of {{STATEMENT}}, a document that still satisfies that section.

   A refresh whose replacement the `iat` floor ({{supersession}}) supersedes is also rejected. A replacement that meets these requirements and that the floor does not supersede is accepted.

4. Apply the requirements of {{currency}}.

5. Bound access and update. The refreshed access MUST fall within the metadata of the document an accepted replacement names or, without one, the establishment's reviewed metadata. If a replacement is accepted, the refreshed grant's statement identity (`iss` and `jti`), `iat`, effective expiry, any claim by which a withdrawal mechanism locates the statement, reviewed metadata and the digest it matched, and issuer trust decision become the replacement's, in one atomic update. Without one, the grant's state is unchanged. Concurrent replacements resolve to the most recently issued statement.

An accepted replacement is the only way refresh changes an establishment's recorded statement and reviewed metadata, which refresh under {{STATEMENT}} alone never changes. The bound that Section 5.7.1 of {{STATEMENT}} sets on refreshed access applies to the reviewed metadata as step 5 leaves it. A replacement never changes the proven key: a client that needs a different key opens a new grant by presentation.

A replacement accepted on refresh MUST apply only to the grant refreshed.

If the server bounds establishments (Section 11.3 of {{STATEMENT}}), a count persists across replacements: a replacement accepted on refresh does not add to it.

# Error Responses {#lifecycle-errors}

This section extends Section 7 of {{STATEMENT}}, whose error responses and HTTP status codes apply at each endpoint.

The `statement_required` error code (Section 7.1 of {{STATEMENT}}) also applies when the statement supplied or pulled is refused by a refusal record ({{withdrawal}}) or superseded under the `iat` floor ({{supersession}}), and when a statement offered as a replacement on refresh ({{refresh-replacement}}) does not satisfy the requirements for one.

A presentation whose statement is refused by a refusal record or superseded under the `iat` floor is rejected with `statement_required`, in place of the `invalid_client` error code that Section 7.2 of {{STATEMENT}} assigns to a statement that fails validation.

A statement whose `status` claim references an index outside the list ({{status-resolution}}) is rejected with the error code assigned to a statement refused by a refusal record, such as `statement_required`.

Under an establishment, a refresh-token request rejected for a reason this specification defines uses `temporarily_unavailable` if a retrieval it needs did not complete, including a pull of a replacement, as Section 7.3 of {{STATEMENT}} provides, and otherwise the following error codes:

`statement_required`:
: A replacement fails step 3 of {{refresh-replacement}} or the `iat` floor, a needed replacement is not obtained in step 2, including when a completed pull yields none, or the refresh fails step 4 ({{currency}}). These cases use `statement_required` in place of `invalid_grant`.

`unauthorized_client` or `invalid_scope`:
: The refreshed access exceeds the metadata of the document an accepted replacement names, as Section 7.3 of {{STATEMENT}} provides for the reviewed metadata.

`invalid_request`:
: The request carries a replacement but omits `client_id` ({{refresh-replacement}}).

If a statement is refused by a refusal record rather than merely expired, and the client is authenticated, the authorization server ought to indicate in the `error_description` parameter that a statement issued more recently is required (Section 7.6 of {{STATEMENT}}).

# Token Status List {#token-status-list}

This section defines the withdrawal mechanism ({{withdrawal}}) of this specification, a profile of Token Status List {{STATUSLIST}} through which an issuer publishes the authoritative status of its statements. A trusting authorization server can resolve those statuses for any issuer it has configured. An issuer can also withdraw a statement through another withdrawal mechanism. The events of {{shared-signals}} are not a withdrawal mechanism: they only prompt a trusting authorization server to resolve a status promptly.
{: #status-mechanisms}

For an issuer that is not an authorization server (Section 4 of {{STATEMENT}}), the trust configuration records a Status List key set URL in place of the `jwks_uri` metadata value ({{status-publication}}, {{status-resolution}}). A configured Status List key set URL MUST use the `https` scheme, and a trusting authorization server MUST validate the certificate of the host serving it. The issuer MUST keep that key set apart from its statement key set and from the `jwks_uri` of any Transmitter configuration it publishes ({{relationship}}), serving it at neither location and placing in it no key that appears at either.

## Status Publication {#status-publication}

An issuer that uses Token Status List publishes the status of its statements as {{STATUSLIST}} defines, and each statement's `status` claim (Section 3 of {{STATEMENT}}) locates its entry in the issuer's Status List Token. An issuer that publishes status:

* MUST publish status, and carry the `status` claim, for every statement it issues under a given `iss` from the point it begins publishing. Once statements issued before that point have expired, a trusting authorization server can read an absent claim as meaning the issuer publishes no status.
* MUST NOT assign the same combination of status list URI and index to more than one statement, including across successive Status List Tokens published at that URI. Withdrawing a statement then affects no other.
* MUST sign the Status List Token with a key published at the `jwks_uri` metadata value of its authorization server metadata {{RFC8414}}, which Sections 4 and 9 of {{STATEMENT}} keep apart from its statement signing keys, or, for an issuer that is not an authorization server, in its Status List key set ({{token-status-list}}).
* MUST include the `exp` claim in every Status List Token it publishes. An older token then cannot stand in for a newer one indefinitely.
* MUST give each Status List Token it publishes at a given list URI an `iat` claim value later than that of any token it published there before ({{status-resolution}}).

## Status Resolution {#status-resolution}

If a statement carries the `status` claim and the trusting authorization server resolves statuses for the statement's issuer, the server MUST reject the statement if its status is `INVALID`, and MUST apply its configured policy for that issuer to the statement if its status is `SUSPENDED` or any other value except `VALID` and `INVALID`.

A status of `INVALID` is a withdrawal, as is any other status except `VALID` when the policy configured for that issuer refuses the statement. On a withdrawal, the server holds a refusal record ({{withdrawal}}). Token Status List defines no expiry for a withdrawal, so the refusal ends only when a later resolution returns a status that no longer refuses the statement.

Resolution follows {{STATUSLIST}}. In particular:

* If the referenced index lies outside the list, the statement is rejected as {{STATUSLIST}} requires ({{lifecycle-errors}}).
* The caching rules of {{STATUSLIST}} apply to resolution, and also apply when the server checks a recorded statement at refresh ({{currency}}).
* A server SHOULD retrieve the list on the list's own schedule, within the expiration and caching bounds its `exp` and `ttl` claims set (Section 11.5 of {{STATUSLIST}}), rather than once per request ({{schedule-practice}}, {{resolution-schedule}}).

A server MUST NOT replace a Status List Token it holds with one whose `iat` claim is earlier, and MUST treat one whose `iat` claim is equal but whose contents differ as a resolution failure. A host serving an older token then cannot restore a status the issuer has withdrawn. Section 8.3 of {{STATUSLIST}} recommends rejecting a statement whose status the server cannot resolve. A statement the server accepts without a resolved status remains bounded by its effective expiry (Section 2 of {{STATEMENT}}).

A server MUST obtain a Status List Token's verification keys from the `jwks_uri` metadata value of the issuer's authorization server metadata {{RFC8414}}, discovered from the configured `iss`, or, for an issuer that is not an authorization server, from the Status List key set URL recorded for it in trust configuration ({{token-status-list}}), and never from the Status List Token itself. The server reuses a retrieved key set only as Section 5.3 of {{STATEMENT}} permits for a statement key set, so that a removed key stops verifying within the server's own bound.

# Status Change Events {#shared-signals}

This section defines an optional Shared Signals Framework {{SSF}} event that tells a trusting authorization server that a status published under {{token-status-list}} has changed. The requirements of {{relationship}} through {{configuration}} apply only to implementations that support the events.

For the events defined here, the issuer acts as a Transmitter, and a trusting authorization server that has configured it acts as a Receiver.

## Event Issuer Trust and Keys {#relationship}

A trusting authorization server MUST NOT accept an event from an issuer it has not configured. It MUST verify the SET using keys from the `jwks_uri` member of the issuer's Transmitter configuration {{SSF}}, discovered from the issuer identifier it has configured, and never from a key-location value carried in the event. The SET's `iss` claim is that issuer identifier, which is also the `iss` claim value of the statements the event concerns.

An issuer MUST ensure that its Transmitter configuration's `jwks_uri` member differs both from the issuer's statement key set location, its `software_statement_jwks_uri` (Sections 4 and 9 of {{STATEMENT}}) or, for an issuer that is not an authorization server, the statement key set URL recorded for it in trust configuration (Section 4 of {{STATEMENT}}), and from the location whose keys verify its Status List Tokens: the `jwks_uri` metadata value in its authorization server metadata {{RFC8414}} or, for an issuer that is not an authorization server, its Status List key set URL ({{token-status-list}}). A receiver MUST NOT create or keep a stream whose Transmitter configuration names either of those locations. An issuer MUST NOT publish at its Transmitter configuration's `jwks_uri` a key that also appears in the key set at either location.

A SET carrying an event defined here MUST use the explicit `typ` JOSE header parameter value `secevent+jwt` (Section 2.3 of {{RFC8417}}). Its `aud` claim MUST contain the audience of the stream that delivers it, which {{SSF}} lets the transmitter and receiver agree on. A receiver MUST NOT create or keep a stream whose audience does not identify it.

## Subject Identification {#subjects}

The subject of every event defined here is client software. Events carry it in the `sub_id` claim of {{SSF}} using the `uri` subject identifier format of {{RFC9493}}. The `uri` member carries the client's CIMD URL exactly as it appears in the `sub` claim of the statements the event concerns.

A trusting authorization server MUST match the subject by exact comparison of the `uri` member against a statement's `sub` claim.

## Status Changed Event {#status-changed}

The event type identifier is `urn:ietf:params:oauth:event-type:software-statement-status-changed` ({{iana-event-type}}).

The event reports that the issuer has changed the published status of one or more statements for the subject. A transmitter MUST NOT transmit the event until a Status List Token reflecting the change is retrievable at the status list URI, including through any caching layer it operates. A receiver resolving before then would fetch the status from before the change.

The event does not say what the new status is, and a receiver MUST NOT infer one from it. Resolving a status of `VALID` after an event is not an error.

The payload ({{RFC8417}}) of an event defined here has the following claims:
{: #events}

`event_timestamp`:
: OPTIONAL. The claim defined by {{CAEP}}, a NumericDate value that here gives the time at which the issuer changed the status the event reports. It is informational: a receiver does not use it to order, bound, or scope anything.

`software_statement_jti`:
: OPTIONAL. The `jti` claim value of a single statement whose status changed ({{processing}}).

`reason_admin`:
: OPTIONAL. An administrative message, as {{CAEP}} defines the claim.

The following is a non-normative example of the payload of a SET carrying this event for one statement:

~~~ json
{
  "iss": "https://issuer.example",
  "jti": "4d3559ec67504aaba65d40b0363faad8",
  "iat": 1767225600,
  "aud": "https://as.example",
  "sub_id": {
    "format": "uri",
    "uri": "https://client.example/app"
  },
  "events": {
"urn:ietf:params:oauth:event-type:software-statement-status-changed":
    {
      "event_timestamp": 1767225590,
      "software_statement_jti": "c2a907b34d28"
    }
  }
}
~~~

## Receiver Processing {#processing}

A trusting authorization server that receives an event defined here MUST process it as follows:

1. Verify the SET as {{RFC8417}} requires, including its `typ` JOSE header parameter, and verify that the SET's issuer is configured and that the keys verifying the SET were obtained as {{relationship}} requires.
2. Reject an event whose `aud` claim does not contain the audience of the stream that delivered it ({{relationship}}).
3. Resolve the subject ({{subjects}}).
4. Retrieve the Status List Token each affected statement's `status` claim references, as {{STATUSLIST}} defines, using a copy that the status list URI served, or confirmed by revalidation, in response to a request begun after the event arrived, without waiting for the schedule that would otherwise apply.
5. Apply the resolved status as {{status-resolution}} defines.

If step 1 or 2 fails, the receiver reports a SET error as {{RFC8935}} or {{RFC8936}} provides. Otherwise, it acknowledges the SET, whatever steps 3 to 5 produce. A receiver ignores any other member of the SET's `events` claim.

The affected statements are those the transmitting issuer made about the subject, never statements another issuer made about the same software. They are the statement the event's `software_statement_jti` claim names or, if the event names none, those the receiver holds or has cached a status for. An event about a subject for which the receiver holds no state requires nothing of it. A receiver ignores events for subjects outside the client identifier namespaces for which it accepts the transmitting issuer (Section 4 of {{STATEMENT}}).

Revalidation suffices because a transmitter sends the event only once a token reflecting the change is retrievable ({{status-changed}}). One request begun after several events arrived serves every one of them that concerns the list it retrieves. An event is never grounds for refusal. Until the request succeeds, the copy the receiver holds remains in effect within its validity. If resolution does not complete, the receiver proceeds as it would had no event arrived: a refusal record it holds still applies, and the statement remains bounded by its effective expiry (Section 2 of {{STATEMENT}}).

A receiver MUST treat an event it has already applied as successfully delivered and acknowledge it as {{RFC8935}} or {{RFC8936}} requires, rather than reporting a delivery error. Rejecting a retry can stall or disable a stream carrying later events. Duplicate detection is per transmitting issuer.

Two constraints bound every event:

* A receiver MUST NOT let an event by itself create standing, extend a statement's lifetime, or otherwise increase what a client can do. A receiver MUST ignore any payload claim that would have such an effect.
* A receiver MUST continue to resolve status as {{status-resolution}} provides, and to enforce statement expiry, independently of this mechanism. Stream loss, transmitter unavailability, or delivery failure then leaves both controls in force.

## Stream Configuration {#configuration}

An issuer that supports the events defined here publishes Transmitter configuration metadata as {{SSF}} defines, discoverable from its issuer identifier. Stream creation, subject management, verification, and delivery follow {{SSF}}.

A trusting authorization server that supports these events SHOULD create one stream per configured issuer that offers them ({{stream-practice}}).

The subjects appropriate to a stream are those of the issuer's statements whose `aud` claim is absent or names the receiving authorization server. A transmitter SHOULD scope each stream to those subjects.

A receiver SHOULD request the event this specification defines, and SHOULD use the stream verification facility of {{SSF}} periodically.

# Implementation Considerations {#implementation-considerations}

This section gives guidance to implementers. It adds no requirement.

## Supersession State {#supersession-state}

For the `iat` floor ({{supersession}}), a server keeps one `iat` value for each `iss` and `sub` and, on a multi-tenant server, each tenant, not the statements themselves. Every statement the floor refuses has an earlier `iat`, so its effective expiry passes within the maximum statement lifetime the server honors for the issuer, counted from the floor's value. Once that period and any clock-skew allowance have passed, the floor refuses nothing that expiry would not refuse. A server that raises the lifetime it honors for an issuer extends that period for the floors it already holds. A floor lost before then readmits the statements it superseded until they expire.

## Resolution in Response to Events {#event-resolution}

A receiver can bound the rate of the retrievals that events cause by deferring a retrieval briefly and coalescing events that concern the same status list URI, since one request begun after those events serves them all ({{processing}}). Its scheduled resolution continues regardless.

## Event Logs {#event-logs}

A receiver ought to treat event logs as sensitive at rest, as Section 11.4 of {{STATEMENT}} treats statements. A log of events is a record about client identifiers that can outlive the issuer's own published status, so a receiver ought to bound its retention.

# Deployment Considerations {#deployment-considerations}

This section gives guidance to operators and issuers. It adds no requirement.

## Withdrawal Practice {#withdrawal-practice}

What a server does when it cannot consult a withdrawal mechanism it relies on is local policy ({{withdrawal}}). Refusing makes issuer availability a precondition for every request the statement governs. Proceeding leaves a withdrawn statement acceptable until it expires.

A refusal that the server's operator entered, rather than one derived from a withdrawal mechanism, persists on the operator's own terms.

Withdrawal does not replace lifetime: an issuer chooses `exp` assuming no trusting authorization server uses a withdrawal mechanism.

## Instances and Supersession {#instance-practice}

Once a later statement supersedes its predecessor at a server ({{supersession}}), an instance holding only the predecessor can open no new grant there. Section 12.3 of {{STATEMENT}} describes how instances obtain the most recently issued statement.

## Resolution Schedule and Lifetimes {#schedule-practice}

Retrieving the list on its own schedule ({{status-resolution}}) keeps each request a statement governs independent of issuer availability. Because events can be suppressed, a deployment ought to choose a resolution schedule and statement lifetimes it would accept with no event stream ({{processing}}).

## Stream Coverage and Monitoring {#stream-practice}

A stream can cover every subject appropriate to it ({{configuration}}), or the receiver can add each subject through the subject management of {{SSF}} when it first admits that software. Admission checks and the receiver's own status resolution ({{status-resolution}}, {{processing}}) remain independent of the subjects a stream covers, so a withdrawal whose event was transmitted before the subject was added to the stream is found at a later resolution.

A receiver ought to alert on stream loss rather than assume quiescence.

# Security Considerations

The security considerations of {{STATEMENT}}, {{STATUSLIST}}, {{SSF}}, and {{RFC8417}} apply.

## Withdrawal and Open Grants {#open-grants}

Expiry and withdrawal ({{withdrawal}}) govern new admissions on a statement. Neither revokes an issued access token. A grant opened by a statement lasts for the life of its refresh tokens, whatever the statement's lifetime, unless the server requires a current statement on refresh ({{currency}}), holds a refusal record, or finds the proven key gone from the client's document (Section 5.7.1 of {{STATEMENT}}). Section 13.7 of {{STATEMENT}} describes what removing an issuer from trust configuration, or narrowing what it can attest, leaves in place. Under this specification, a server can also end those grants and registrations by entering refusal records for the issuer's statements.

## Supersession and Refresh Replacement {#replacement-security}

The `iat` floor ({{supersession}}) exists because one statement is consumable more than once (Section 3.3 of {{STATEMENT}}). Without it, a client holding an older, broader statement could keep opening registrations and establishments with it after its issuer narrows the review, until that statement expires. The floor closes this at a server only once the server accepts the later statement. A server that has not seen the later statement still accepts the earlier one, which is why {{withdrawal}} recommends that an issuer withdraw an earlier statement when it issues a later, narrower one.

A replacement never changes the establishment's proven key. Refresh verifies possession of that key before it considers a replacement ({{refresh-replacement}}), so holding a statement gives a party nothing at refresh without the key, and a replacement cannot move a grant to another key.

A replacement validates only against the document currently served at its `sub` (step 3 of {{refresh-replacement}}). A client cannot use a later statement over a document it no longer serves, so an accepted replacement brings the grant to the current reviewed document, and the refreshed access falls within that document's metadata.

## Forged and Replayed Events

An event names no status, so a forged, replayed, delayed, or reordered event cannot change whether any statement is acceptable. Its only effect is a resolution the receiver would otherwise have performed on its own schedule.

The remaining exposure is resource cost: an attacker holding the issuer's SET signing key, or a misbehaving transmitter, can trigger resolutions ({{event-resolution}}).

## Event Suppression

Because a receiver resolves status independently ({{processing}}), an attacker who suppresses events delays a withdrawal at most until the receiver's next scheduled resolution and, for new admissions, never beyond the affected statements' effective expiry ({{schedule-practice}}). Grants already open continue until a refresh at which the receiver holds a refusal record, which {{currency}} then rejects unless a replacement is accepted.

The absence of events looks the same on a healthy stream and a suppressed one. Periodic stream verification ({{configuration}}) makes the difference observable ({{stream-practice}}).

## Key Separation and Compromise

The distinct key sets of {{relationship}} separate SET keys from statement and Status List Token keys. With the keys separated, compromise of a SET key lets an attacker trigger resolutions and nothing more. Compromise of a statement signing key has wider effect, because statements grant standing and events do not.

# Privacy Considerations {#resolution-schedule}

The privacy considerations of {{STATEMENT}}, {{STATUSLIST}}, {{SSF}}, and {{RFC8417}} apply.

Because {{STATUSLIST}} aggregates many statements into one list, a retrieval tells the issuer only that some trusting authorization server is checking. Retrieving the list on its own schedule ({{status-resolution}}) keeps the server's request timing from disclosing the client population it serves. A retrieval that an event prompts shows the issuer that the receiver holds a statement or a cached status for the event's subject.

Subject identifiers in these events name reviewed software and, in aggregate, describe an organization's approved software estate. For this reason, a stream is scoped as {{configuration}} provides. Event logs deserve the same care as statements ({{event-logs}}).

Refusal records ({{withdrawal}}) and `iat` floors ({{supersession}}) are durable state that an authorization server keeps per client identifier. They show which software an issuer has withdrawn or reviewed again, so a server ought to protect them as it protects statements (Section 11.4 of {{STATEMENT}}) and ought not to keep an `iat` floor longer than its purpose requires ({{supersession-state}}).

The events this specification defines create no durable receiver-side record of their own, so they add no negative state about a client identifier that outlives the issuer's own published status. A receiver that logs events creates such a record ({{event-logs}}).

# IANA Considerations

## OAuth URI Registry {#iana-event-type}

{{RFC8417}} establishes no registry of event types, so this specification takes its identifier from the `oauth` sub-namespace {{RFC6755}} and requests registration of the following value in the IANA "OAuth URI" registry:

URN:
: `urn:ietf:params:oauth:event-type:software-statement-status-changed`

Common Name:
: Client ID Metadata Document Software Statement Status Changed Event Type

Change Controller:
: IETF

Specification Document(s):
: This specification, {{status-changed}}

## SET Payload Claims

Under Section 2 of {{RFC8417}}, the `software_statement_jti` payload claim of {{status-changed}} is defined by this specification and need not be registered as a JWT claim. No IANA registry of SET payload claims exists, so no registration is requested.

--- back

# Acknowledgments
{:numbered="false"}

This specification draws on the Shared Signals Framework and the Continuous Access Evaluation Profile for its transport and payload conventions. The lifecycle rules of {{withdrawal}} through {{lifecycle-errors}} began in {{STATEMENT}} and were separated from it so that its admission baseline stands alone.
