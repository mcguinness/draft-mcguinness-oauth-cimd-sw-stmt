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
  RFC6749:
  RFC8935:
  RFC8936:
  RFC8414:
  RFC8417:
  RFC9493:
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
    author:
      - ins: A. Tulshibagwale
        org: SGNL
      - ins: T. Cappalli
        org: Okta
      - ins: M. Scurtescu
        org: Coinbase
      - ins: A. Backman
        org: Amazon
      - ins: J. Bradley
        org: Yubico
      - ins: S. Miel
        org: Cisco
    date: 2025-08-29
  CAEP:
    target: https://openid.net/specs/openid-caep-1_0.html
    title: "OpenID Continuous Access Evaluation Profile 1.0"
    author:
      - ins: T. Cappalli
        org: Okta
      - ins: A. Tulshibagwale
        org: SGNL
    date: 2025-08-29

informative:
  CIMD: I-D.ietf-oauth-client-id-metadata-document
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

A companion specification defines a software statement by which a trusted issuer records its review of the exact octets of a Client ID Metadata Document (CIMD). That specification ends no grant at a statement's expiry, keeps an older unexpired statement admissible after a newer one is issued, and evaluates no statement on refresh. This specification lets an issuer withdraw a statement, and an authorization server that learns of the withdrawal keeps a refusal record. An `iat` floor supersedes an issuer's earlier statements about the same software once a server accepts a later one. Refresh is refused for a grant whose statement is withdrawn or, by recommendation, expired, unless a later statement replaces it. A profile of Token Status List is the withdrawal mechanism defined here, and an optional Shared Signals event prompts servers to check status promptly.

--- middle

# Introduction

{{STATEMENT}} defines a software statement by which a configured issuer records its review of the exact octets of a Client ID Metadata Document (CIMD) {{CIMD}}, and how an authorization server admits a client and binds a grant on that review. It leaves out what follows admission: expiry ends no grant already opened, an issuer cannot withdraw a statement before it expires, an older unexpired statement remains admissible after a newer one is issued, and refresh evaluates no statement.

This specification adds withdrawal and refusal records ({{withdrawal}}), supersession of earlier statements through an `iat` floor ({{supersession}}), and refresh rules ({{refresh-lifecycle}}) under which a grant whose recorded statement is withdrawn, or by recommendation expired, continues only if a later statement replaces it.

A trusting authorization server that implements this specification applies all of {{withdrawal}} through {{refresh-lifecycle}} and {{lifecycle-errors}}, whatever withdrawal mechanism it uses, if any. {{withdrawal}} recommends that an issuer withdraw an earlier statement when it issues a later, narrower one. Publishing status and sending events are optional for an issuer, and supersession and refresh replacement work without them. A client can offer a later statement on refresh ({{refresh-replacement}}).

{{token-status-list}} profiles Token Status List {{STATUSLIST}} as the withdrawal mechanism of this specification: an issuer withdraws a statement by setting its status, and a trusting authorization server learns of it when it next resolves the list. {{shared-signals}} defines an optional Shared Signals Framework {{SSF}} event, not itself a withdrawal mechanism, that prompts the trusting authorization servers that have configured an issuer to resolve a changed status promptly.

Registrations under {{REGISTRATION}} advance the `iat` floor, and a refusal record ends a registration's validity at the server that holds it. This specification defines no new endpoint. It extends {{STATEMENT}}, which does not depend on it.

## Overview {#overview}

The rules of this specification act at two points: when a statement is accepted to open something new, such as an admission, a registration, or a refresh replacement, and when a grant or registration it governs continues. The `iat` floor ({{supersession}}) limits only the first. A refusal record ({{withdrawal}}), and expiry where the server requires a current statement ({{currency}}), also limit the second. {{tab-standing}} shows what a statement's standing at one authorization server allows.

| Standing at the server | New admission or registration | Pulled statement selection | Refresh of a grant that records it | Use as a refresh replacement |
| --- | --- | --- | --- | --- |
| Current, not superseded | Accepted | Considered | Continues | Accepted if later than the recorded statement |
| Superseded by the `iat` floor | Rejected | Ignored | Continues | Rejected |
| Covered by a refusal record | Rejected | Ignored | Needs a replacement | Rejected |
| Expired | Rejected | Ignored | Continues unless {{currency}} requires a replacement | Rejected |
{: #tab-standing title="What a Statement's Standing Allows"}

A registration ({{REGISTRATION}}) follows the same pattern: a refusal record covering its governing statement ends its validity, and a superseded governing statement leaves it valid until that statement expires.

A server implementing this specification keeps the following state:

* a refusal record for each statement it has learned is withdrawn, or that its operator refused ({{withdrawal}});
* an `iat` floor for each `iss` and `sub`, per tenant on a multi-tenant server ({{supersession}});
* for each establishment, its recorded statement and what step 4 of {{refresh-replacement}} records with it;
* with Token Status List, the Status List Tokens it holds ({{status-resolution}}); and
* with status change events, its streams and the SET `jti` values it has applied on each ({{processing}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses the terms transmitter, receiver, stream, and transmitter configuration metadata from {{SSF}}, Security Event Token (SET) from {{RFC8417}}, Status List Token from {{STATUSLIST}}, and the terms defined in Section 2 of {{STATEMENT}}.

This specification defines the following terms.

Withdrawal Mechanism:
: The means by which a trusting authorization server learns that an issuer has withdrawn a statement before its expiry ({{withdrawal}}), such as the Token Status List profile of {{token-status-list}}.

Refusal Record:
: State an authorization server holds for a statement, identified by its `iss` and `jti`, once it has learned that the statement ceased to be acceptable before its expiry, whether from a withdrawal mechanism it uses for the issuer ({{withdrawal}}) or from its own operator.

Recorded Statement:
: The statement an establishment records: the one presented or pulled to create it or, after a refresh accepts a replacement, the replacement most recently accepted for it ({{refresh-replacement}}).

Refresh Replacement:
: A later statement from the recorded statement's issuer about the same software, offered or pulled on refresh to take the recorded statement's place ({{refresh-replacement}}).

`iat` Floor:
: For a trusting authorization server, the greatest `iat` claim value among the statements it has accepted for a given `iss` and `sub`, per tenant on a multi-tenant server ({{supersession}}).

This specification also extends the following term of {{STATEMENT}}.

Current Statement:
: As Section 2 of {{STATEMENT}} defines it. Under this specification, a refusal record ends a statement, so a statement that a refusal record covers is not current ({{withdrawal}}). The `iat` floor ({{supersession}}) separately limits which current statements a server accepts.

# Withdrawal and Refusal Records {#withdrawal}

An issuer can withdraw a statement before its `exp`. Unlike OAuth token revocation ({{RFC7009}}), a withdrawal revokes no issued token: a server holding a refusal record refuses new admissions and refresh under the statement ({{currency}}, {{open-grants}}).

How a trusting authorization server learns of a withdrawal is a withdrawal mechanism. This specification requires no withdrawal mechanism. It defines one, the Token Status List profile of {{token-status-list}}, and other specifications can define how another mechanism applies to these statements.

In addition to the checks of Section 3.2 of {{STATEMENT}}, a trusting authorization server MUST reject a statement for which it holds a refusal record. A server that uses a withdrawal mechanism for a statement's issuer SHOULD consult it, within the caching and freshness rules that mechanism defines, before accepting the statement for a new admission, at each refresh of a grant that records it, and at each token request under a registration the statement governs ({{REGISTRATION}}).

A refusal derived from a withdrawal mechanism lasts until a later authoritative result from that mechanism replaces it, or until an expiry the mechanism itself defines for the withdrawal. A server MUST NOT retain such a refusal once it is replaced or expired in that way. A failure to consult the mechanism, or cached evidence that has lapsed, neither replaces nor expires a refusal.

A withdrawal constrains and never relaxes. A statement for which no withdrawal mechanism reports a withdrawal, or for which the server cannot consult the mechanism it uses ({{withdrawal-practice}}), is bounded by its effective expiry as it would be otherwise, and a server MUST NOT treat any withdrawal mechanism as grounds to accept a statement past its effective expiry.

An issuer that uses a withdrawal mechanism and issues a later statement narrower than the one it replaces SHOULD withdraw the earlier statement ({{replacement-security}}). A later statement is narrower if the document it names no longer carries a key, redirection URI, or scope the earlier document carried, if its `aud`, `aud_tenant`, or `statement_uses` is narrower, or if its remaining lifetime is shorter.

Rejections under this section use the error responses of {{lifecycle-errors}}.

# Supersession {#supersession}

Because an issuer's `iat` values for a given `iss` and `sub` increase strictly (Section 3 of {{STATEMENT}}), a later `iat` marks a later review, and a statement whose `iat` is earlier than the server's `iat` floor is superseded.

A trusting authorization server MUST retain each `iat` floor for at least the maximum statement lifetime it honors for the issuer after the floor's value. A statement is accepted for the floor once the server admits a client on it, applies it as a replacement ({{refresh-replacement}}), or registers a client or renews a registration with it ({{REGISTRATION}}). The server MUST reject a statement whose `iat` claim is earlier than the retained `iat` floor, whether it arrives as a replacement on refresh, a new registration or a replacement delivered under {{REGISTRATION}}, or a runtime presentation. A repeated registration request that Section 3.3 of {{REGISTRATION}} answers with the existing registration is not a new registration. A client holding a superseded, broader statement therefore cannot defer a narrowing by opening a fresh registration or establishment with it.

The floor needs one `iat` value for each `iss` and `sub` and, on a multi-tenant server, each tenant, not the statements themselves. Every statement the floor refuses has an earlier `iat`, so its effective expiry passes within the maximum statement lifetime the server honors for the issuer, counted from the floor's value. Once that period and any clock-skew allowance have passed, the floor refuses nothing that expiry would not refuse. A server that raises the lifetime it honors for an issuer extends that period for the floors it already holds. A floor lost before then leaves the statements it superseded acceptable until they expire.

At admission, the server verifies the `iat` floor in step 3 of Section 5.1 of {{STATEMENT}}, along with the `client_id` rule, before it resolves the reviewed document.

In selecting a pulled statement (Section 5.5.1 of {{STATEMENT}}), the server also ignores any statement that a refusal record covers ({{withdrawal}}) or that the `iat` floor supersedes. A pulled statement the server does not apply does not advance the `iat` floor. Nor does a review-only presentation or pulled statement (Section 5.6.3 of {{STATEMENT}}).

An instance holding only a superseded statement can open no new grant at the server, which is why instances need the most recently issued statement (Section 12.3 of {{STATEMENT}}). Grants it already opened continue under {{refresh-lifecycle}}.

# Refresh {#refresh-lifecycle}

Under {{STATEMENT}}, refresh evaluates no statement: it checks the establishment's proven key (Section 5.7.1 of {{STATEMENT}}). This section extends refresh so that a grant can depend on a current statement.

## Requiring a Current Statement {#currency}

When the establishment's recorded statement has expired, the authorization server SHOULD require a current statement on refresh-token use ({{open-grants}}). A refresh under an establishment needs a replacement ({{refresh-replacement}}) if a refusal record covers the recorded statement, or if the recorded statement has expired and either the server requires a current statement or a required issuer (Section 4 of {{STATEMENT}}) covers the establishment's `sub` in the establishment's tenant. The server MUST reject a refresh that needs a replacement unless it accepts one, even if it does not otherwise require a current statement. If a required issuer covers that `sub` and tenant and the recorded statement is from another issuer, the server MUST reject the refresh, and the client opens a new grant by runtime presentation. A raised `iat` floor ({{supersession}}) does not by itself require a replacement.

## Refresh Replacement {#refresh-replacement}

A replacement is a `presentation` use of its statement (Section 3 of {{STATEMENT}}).

On refresh-token use under an establishment, the authorization server first completes the checks of Section 5.7.1 of {{STATEMENT}} on the establishment's proven key, unchanged, before it considers any replacement. A server that implements this specification then proceeds as follows. A refresh rejected at any step leaves the grant's state unchanged.

{{fig-refresh}} summarizes these steps and the error each exit returns. {{lifecycle-errors}} lists every code.

~~~ ascii-art
refresh-token request
  |
proven key checks (Section 5.7.1 of [STATEMENT])
  |     fail ............................. invalid_grant
step 1: is a replacement needed (Section 5.1)?
  |     recorded statement is not from
  |     the required issuer .............. statement_required
  |     no, and none offered ............. go to step 4
step 2: obtain it, offered or pulled
  |     offered without client_id ........ invalid_request
  |     pull did not complete ............ temporarily_unavailable
  |     none obtained .................... statement_required
step 3: validate it
  |     fails ............................ statement_required
step 4: bound the access, apply an accepted replacement
  |     exceeds the metadata ............. unauthorized_client
  |                                        or invalid_scope
tokens issued
~~~
{: #fig-refresh title="Refresh Processing and Its Errors"}

1. Decide whether {{currency}} requires a replacement or rejects the refresh, having consulted any withdrawal mechanism as {{withdrawal}} recommends.

2. Obtain a replacement. A client can offer one on any refresh in the `software_statement` parameter. A request carrying a replacement MUST meet the `client_id` rule that Section 5.4 of {{STATEMENT}} sets for a request making a runtime presentation. A statement with the recorded `iss` and `jti` is not a replacement, and a request carrying it is processed as one offering none. When a replacement is needed and the request carries none, a server that supports pulled statements pulls, however the establishment was created, and takes as the replacement the most recently issued statement with the recorded `iss` and `sub` among those it considers under Section 5.5.1 of {{STATEMENT}}, {{withdrawal}}, and {{supersession}}. A refresh that needs a replacement and obtains none is rejected: a reviewed grant never continues under the policy for clients the server has not reviewed.

   The pull follows Sections 5.3 and 5.5 of {{STATEMENT}}, and a refresh whose pull does not complete is rejected.

3. Validate the replacement. A refresh whose replacement fails any of the following requirements is rejected, even when no replacement was needed. The replacement:

   * MUST validate under Section 3.2 of {{STATEMENT}}, {{withdrawal}}, and {{supersession}}.
   * MUST have the recorded statement's `iss` and `sub`.
   * MUST have an `iat` later than the recorded statement's.
   * MUST name in `cimd_digest` the document currently served at that `sub`, which the server confirms by retrieval or by revalidating octets it holds.
   * MUST name a document that carries the establishment's proven key, in its `jwks` or at its `jwks_uri` (Section 5.6.1 of {{STATEMENT}}), or, for an establishment created under Section 5.6.2 of {{STATEMENT}}, a document that still satisfies that section.

   A replacement that meets these requirements is accepted.

4. Bound access and update. The refreshed access MUST fall within the metadata of the document an accepted replacement names or, without one, the establishment's reviewed metadata. If a replacement is accepted, the refreshed grant's statement identity (`iss` and `jti`), `iat`, effective expiry, any claim an extension uses to locate the statement, reviewed metadata and the digest it matched, and issuer trust decision become the replacement's, in one atomic update. Concurrent replacements resolve to the most recently issued statement.

A replacement accepted on refresh MUST apply only to the grant refreshed.

If the server bounds establishments (Section 11.3 of {{STATEMENT}}), a count persists across replacements: a replacement accepted on refresh does not add to it.

# Token Status List {#token-status-list}

This section defines the withdrawal mechanism ({{withdrawal}}) of this specification, a profile of Token Status List {{STATUSLIST}} through which an issuer publishes the authoritative status of its statements.

## Issuer Key Sets {#key-sets}

An issuer's Status List key set is the JWK Set at the `jwks_uri` of its authorization server metadata {{RFC8414}}, discovered from the configured `iss`. For an issuer that is not an authorization server (Section 4 of {{STATEMENT}}), it is the JWK Set at a Status List key set URL in the trust configuration. The `https` and certificate requirements of Section 4 of {{STATEMENT}} for a statement key set URL apply to that URL. An issuer MUST serve its statement key set, its Status List key set, and the key set at the `jwks_uri` of any transmitter configuration metadata it publishes ({{relationship}}) at three different locations, with no key in more than one of them.

| Key set | Location | Verifies |
| --- | --- | --- |
| Statement key set | Section 4 of {{STATEMENT}} | Software statements |
| Status List key set | The `jwks_uri` of the issuer's authorization server metadata or, for an issuer that is not an authorization server, a configured URL | Status List Tokens ({{status-publication}}) |
| Transmitter key set | The `jwks_uri` of the issuer's transmitter configuration metadata | Status change events ({{shared-signals}}) |
{: #tab-key-sets title="Issuer Key Sets"}

## Status Publication {#status-publication}

An issuer that uses Token Status List publishes the status of its statements as {{STATUSLIST}} defines. A statement locates its entry in the issuer's Status List Token through the `status` claim, whose `status_list` member gives the status list URI and index (Section 6.2 of {{STATUSLIST}}). It is the claim by which this specification locates a statement, which an establishment records (Section 5.7 of {{STATEMENT}}). An issuer that publishes status:

* MUST publish status, and carry the `status` claim, for every statement it issues under a given `iss` from the point it begins publishing. Once statements issued before that point have expired, a trusting authorization server can read an absent claim as meaning the issuer publishes no status.
* MUST NOT assign the same combination of status list URI and index to more than one statement, including across successive Status List Tokens published at that URI. Withdrawing a statement then affects no other.
* MUST sign each Status List Token with a key in its Status List key set ({{key-sets}}).
* MUST include the `exp` claim in every Status List Token it publishes. An older token then cannot stand in for a newer one indefinitely.
* MUST give each Status List Token it publishes at a given list URI an `iat` claim value later than that of any token it published there before ({{status-resolution}}).

## Status Resolution {#status-resolution}

If a statement carries the `status` claim and the server resolves statuses for its issuer, a status of `INVALID` is a withdrawal, as is any other status except `VALID` that the server's policy for that issuer refuses. On a withdrawal, the server holds a refusal record ({{withdrawal}}).

Resolution follows {{STATUSLIST}}. A statement whose status index lies outside the list is rejected, as {{STATUSLIST}} requires. A server SHOULD retrieve the list on the list's own schedule, within the expiration and caching bounds its `exp` and `ttl` claims set (Section 13.7 of {{STATUSLIST}}), rather than once per request ({{schedule-practice}}, {{privacy-considerations}}).

A server MUST NOT replace a Status List Token it holds with one whose `iat` claim is earlier, and MUST treat one whose `iat` claim is equal but whose contents differ as a resolution failure. A host serving an older token then cannot restore a status the issuer has withdrawn.

A server MUST verify a Status List Token only with keys from the issuer's Status List key set ({{key-sets}}), never with keys from the token itself. It reuses a retrieved key set only as Section 5.3 of {{STATEMENT}} permits for a statement key set.

# Status Change Events {#shared-signals}

This section defines an optional Shared Signals Framework {{SSF}} event that tells a trusting authorization server that a status published under {{token-status-list}} has changed. The requirements of {{configuration}} through {{processing}} apply only to implementations that support the events.

For the events defined here, the issuer acts as a transmitter, and a trusting authorization server that has configured it acts as a receiver.

{{fig-withdrawal}} shows a withdrawal that an event signals.

~~~ ascii-art
  Issuer        Status list host    Authorization server    Client
    |                  |                     |                 |
    | (1) token with   |                     |                 |
    |  changed status  |                     |                 |
    |----------------->|                     |                 |
    | (2) status changed event (SET)         |                 |
    |--------------------------------------->|                 |
    |               (3) acknowledgement      |                 |
    |<---------------------------------------|                 |
    |                  | (4) fetch the list  |                 |
    |                  |<--------------------|                 |
    |                  | (5) Status List     |                 |
    |                  |     Token           |                 |
    |                  |-------------------->|                 |
    |                  |     (6) refusal record                |
    |                  |                     | (7) refresh     |
    |                  |                     |<----------------|
    |                  |                     | (8) replacement |
    |                  |                     |     needed      |
    |                  |                     |---------------->|
~~~
{: #fig-withdrawal title="Withdrawal Signaled by an Event"}

The issuer publishes a Status List Token reflecting the change (1) before it sends the event (2), which the receiver acknowledges (3). The receiver then retrieves the list (4, 5) and, if the status is `INVALID`, holds a refusal record (6). The next refresh of a grant that records the statement (7) needs a replacement (8), or is rejected with `statement_required`. Without the event, steps 4 to 6 happen on the server's own schedule ({{status-resolution}}), so a suppressed event delays the refusal but does not prevent it.

## Stream Configuration {#configuration}

An issuer that supports the events defined here publishes transmitter configuration metadata as {{SSF}} defines, discoverable from its issuer identifier. Stream creation, subject management, verification, and delivery follow {{SSF}}.

A trusting authorization server that supports these events SHOULD create one stream per configured issuer that offers them ({{stream-practice}}).

The subjects appropriate to a stream are those of the issuer's statements whose `aud` claim is absent or names the receiving authorization server. A transmitter SHOULD scope each stream to those subjects.

A receiver SHOULD request the event this specification defines, and SHOULD use the stream verification facility of {{SSF}} periodically.

## Status Changed Event {#status-changed}

The subject of every event defined here is client software. Events carry it in the `sub_id` claim of {{SSF}} using the `uri` subject identifier format (Section 3.2.7 of {{RFC9493}}). The `uri` member carries the client's CIMD URL exactly as it appears in the `sub` claim of the statements the event concerns.

A trusting authorization server MUST match the subject by exact comparison of the `uri` member against a statement's `sub` claim.

The event type identifier is `urn:ietf:params:oauth:event-type:software-statement-status-changed` ({{iana-event-type}}).

The event reports that the issuer has changed the published status of one or more statements for the subject. A transmitter MUST NOT transmit the event until a Status List Token reflecting the change is retrievable at the status list URI, including through any caching layer it operates. A receiver resolving before then would fetch the status from before the change.

The event does not say what the new status is, and a receiver MUST NOT infer one from it. Resolving a status of `VALID` after an event is not an error.

The payload ({{RFC8417}}) of an event defined here has the following claims:

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

## Event Issuer Trust and Keys {#relationship}

A trusting authorization server MUST NOT accept an event from an issuer it has not configured. It MUST verify the SET using keys from the `jwks_uri` member of the issuer's transmitter configuration metadata {{SSF}}, discovered from the issuer identifier it has configured, and never from a key-location value carried in the event. The SET's `iss` claim is that issuer identifier, which is also the `iss` claim value of the statements the event concerns.

A receiver MUST NOT create or keep a stream whose transmitter configuration metadata gives as its `jwks_uri` the location of the issuer's statement key set or Status List key set ({{key-sets}}).

A SET carrying an event defined here MUST have an `aud` claim containing the audience of the stream that delivers it, which {{SSF}} lets the transmitter and receiver agree on. A receiver MUST NOT create or keep a stream whose audience does not identify it.

## Receiver Processing {#processing}

The affected statements are those the transmitting issuer made about the subject, never statements another issuer made about the same software. They are the statement the event's `software_statement_jti` claim names or, if the event names none, those the receiver holds or has cached a status for. An event about a subject for which the receiver holds no state requires nothing of it. A receiver ignores events for subjects outside the client identifier namespaces for which it accepts the transmitting issuer (Section 4 of {{STATEMENT}}).

A trusting authorization server that receives an event defined here MUST process it as follows:

1. Verify the SET as {{RFC8417}}, {{SSF}}, and {{relationship}} require.
2. Reject an event whose `aud` claim does not contain the audience of the stream that delivered it ({{relationship}}).
3. Resolve the subject ({{status-changed}}).
4. Retrieve the Status List Token each affected statement's `status` claim references, as {{STATUSLIST}} defines, using a copy that the status list URI served, or confirmed by revalidation, in response to a request begun after the event arrived, without waiting for the schedule that would otherwise apply.
5. Apply the resolved status as {{status-resolution}} defines.

If step 1 or 2 fails, the receiver reports a SET error as {{RFC8935}} or {{RFC8936}} provides. Otherwise, it acknowledges the SET, whatever steps 3 to 5 produce.

One request begun after several events arrived serves every one of them that concerns the list it retrieves. An event is never grounds for refusal. Until the request succeeds, the copy the receiver holds remains in effect within its validity. If resolution does not complete, the receiver proceeds as it would had no event arrived.

A receiver MUST treat an event it has already applied as successfully delivered and acknowledge it as {{RFC8935}} or {{RFC8936}} requires, rather than reporting a delivery error. Rejecting a retry can stall or disable a stream carrying later events. A receiver recognizes a repeated delivery by the SET's `jti` claim, which is unique within an event feed (Section 2.2 of {{RFC8417}}), so it detects duplicates per stream.

Two constraints bound every event:

* A receiver MUST NOT let an event by itself create standing, extend a statement's lifetime, or otherwise increase what a client can do. A receiver MUST ignore any payload claim that would have such an effect.
* A receiver MUST continue to resolve status as {{status-resolution}} provides, independently of this mechanism. Stream loss, transmitter unavailability, or delivery failure then leaves resolution in force.

# Error Responses {#lifecycle-errors}

This section extends Section 6 of {{STATEMENT}}, whose error responses and HTTP status codes apply at each endpoint.

A presentation is rejected with `statement_required`, in place of the `invalid_client` that Section 6.2 of {{STATEMENT}} assigns, if a refusal record covers its statement ({{withdrawal}}), the statement's status index lies outside the list ({{status-resolution}}), or the `iat` floor supersedes the statement ({{supersession}}).

A refresh-token request that {{currency}} rejects, or whose replacement is rejected ({{refresh-replacement}}), uses `statement_required`, subject to the order of Section 6 of {{STATEMENT}}. A request that carries a replacement but omits `client_id` ({{refresh-replacement}}) is rejected with `invalid_request`. One whose `client_id` is not the client the refresh token was issued to fails the grant binding and is rejected with `invalid_grant` (Section 5.2 of {{RFC6749}}), and one whose replacement names a `sub` other than that `client_id` is rejected with `statement_required`. A refresh whose access exceeds the metadata of the document an accepted replacement names is rejected with `unauthorized_client` or `invalid_scope`, as Section 6.3 of {{STATEMENT}} provides for the reviewed metadata.

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} value:

`software_statement_replacement_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server applies refresh replacements ({{refresh-replacement}}). If omitted, the default value is `false`.

An authorization server implementing this specification MUST publish `software_statement_replacement_supported` with the value `true`.

On refresh-token use under an establishment, a server that does not apply refresh replacements ignores an offered statement (Section 5.4 of {{STATEMENT}}), so a successful refresh does not show that the server applied the statement. At a server that does not advertise `true`, a client that needs a later review to govern a grant under an establishment opens a new grant by runtime presentation. This value does not describe the renewal of a registration, which a statement delivered on a refresh-token request can perform at a server that implements {{REGISTRATION}}.

# Implementation Considerations {#implementation-considerations}

This section gives guidance to implementers. It adds no requirement.

## Resolution in Response to Events {#event-resolution}

A receiver can bound the rate of the retrievals that events cause by deferring a retrieval briefly and coalescing events that concern the same status list URI ({{processing}}). Its scheduled resolution continues regardless.

# Deployment Considerations {#deployment-considerations}

This section gives guidance to operators and issuers. It adds no requirement.

## Withdrawal Practice {#withdrawal-practice}

What a server does when it cannot consult a withdrawal mechanism it relies on is local policy ({{withdrawal}}). For Token Status List, Section 8.3 of {{STATUSLIST}} recommends rejecting a statement whose status the server cannot resolve. Refusing makes issuer availability a precondition for every request the statement governs. Proceeding leaves a withdrawn statement acceptable until it expires.

A refusal that the server's operator entered, rather than one derived from a withdrawal mechanism, persists on the operator's own terms.

Withdrawal does not replace lifetime: an issuer chooses `exp` assuming no trusting authorization server uses a withdrawal mechanism.

## Resolution Schedule and Lifetimes {#schedule-practice}

Retrieving the list on its own schedule ({{status-resolution}}) keeps each request a statement governs independent of issuer availability. Because events can be suppressed, a deployment ought to choose a resolution schedule and statement lifetimes it would accept with no event stream ({{processing}}).

## Stream Coverage and Monitoring {#stream-practice}

A stream can cover every subject appropriate to it ({{configuration}}), or the receiver can add each subject through the subject management of {{SSF}} when it first accepts a statement about that software. A withdrawal whose event was transmitted before the subject was added to the stream is found at a later resolution ({{processing}}).

A receiver ought to alert on stream loss rather than assume quiescence.

# Security Considerations

The security considerations of Section 13 of {{STATEMENT}}, {{STATUSLIST}}, {{SSF}}, and {{RFC8417}} apply.

## Withdrawal and Open Grants {#open-grants}

A withdrawal revokes no issued token ({{withdrawal}}). Beyond the cases of Section 13.7 of {{STATEMENT}}, {{currency}} can prevent further refresh of a grant until the server accepts a replacement ({{refresh-replacement}}), and a server that removes an issuer from trust can end the grants opened and registrations created on that issuer's statements by entering refusal records for them.

## Supersession and Refresh Replacement {#replacement-security}

The `iat` floor ({{supersession}}) exists because one statement is consumable more than once (Section 11.3 of {{STATEMENT}}). Without it, a client holding an older, broader statement could keep opening registrations and establishments with it after its issuer narrows the review, until that statement expires. The floor closes this at a server only once the server accepts the later statement. A server that has not seen the later statement still accepts the earlier one, which is why {{withdrawal}} recommends that an issuer withdraw an earlier statement when it issues a later, narrower one.

Refresh verifies possession of the establishment's proven key before it considers a replacement ({{refresh-replacement}}), so holding a statement gives a party nothing at refresh without the key, and a replacement cannot move a grant to another key.

Because a replacement must name the document currently served (step 3 of {{refresh-replacement}}), a client cannot move a grant to a reviewed document it no longer serves.

## Forged and Replayed Events

An event names no status, so a forged, replayed, delayed, or reordered event cannot change whether any statement is acceptable. Its only effect is a resolution the receiver would otherwise have performed on its own schedule.

The remaining exposure is resource cost: an attacker holding the issuer's SET signing key, or a misbehaving transmitter, can trigger resolutions ({{event-resolution}}) and nothing more, because the key set that verifies SETs shares no location or key with the statement key set or the Status List key set ({{token-status-list}}).

## Event Suppression

Because a receiver resolves status independently ({{processing}}), an attacker who suppresses events delays a withdrawal at most until the receiver's next scheduled resolution and, for new admissions, never beyond the affected statements' effective expiry ({{schedule-practice}}).

The absence of events looks the same on a healthy stream and a suppressed one. Periodic stream verification ({{configuration}}) makes the difference observable ({{stream-practice}}).

# Privacy Considerations {#privacy-considerations}

The privacy considerations of Section 14 of {{STATEMENT}}, {{STATUSLIST}}, {{SSF}}, and {{RFC8417}} apply.

Because {{STATUSLIST}} aggregates many statements into one list, a retrieval tells the issuer only that some trusting authorization server is checking. Retrieving the list on its own schedule ({{status-resolution}}) keeps the server's request timing from disclosing the client population it serves. A retrieval that an event prompts shows the issuer that the receiver holds a statement or a cached status for the event's subject.

Subject identifiers in these events name reviewed software and, in aggregate, describe an organization's approved software estate. For this reason, a stream is scoped as {{configuration}} provides. A receiver ought to treat event logs as sensitive at rest, as Section 11.4 of {{STATEMENT}} treats statements. A log of events is a record about client identifiers that can outlive the issuer's own published status, so a receiver ought to bound its retention.

Refusal records ({{withdrawal}}) and `iat` floors ({{supersession}}) are durable state that an authorization server keeps per client identifier. They show which software an issuer has withdrawn or reviewed again, so a server ought to protect them as it protects statements (Section 11.4 of {{STATEMENT}}) and ought not to keep an `iat` floor longer than its purpose requires ({{supersession}}).

# IANA Considerations

The `software_statement_jti` payload claim of {{status-changed}} needs no registration, because SET payload claims need not be registered as JWT claims (Section 2 of {{RFC8417}}) and no IANA registry of SET payload claims exists.

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

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_replacement_supported`

Metadata Description:
: Boolean value indicating whether the authorization server applies Client ID Metadata Document software statement refresh replacements.

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-server-metadata}}

--- back
