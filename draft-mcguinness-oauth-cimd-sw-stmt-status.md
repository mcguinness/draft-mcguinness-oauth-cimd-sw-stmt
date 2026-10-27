---
title: "Client ID Metadata Document Software Statement Status"
abbrev: oauth-cimd-sw-stmt-status
docname: draft-mcguinness-oauth-cimd-sw-stmt-status-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Shared Signals
 - Security Event Token
 - Status List
 - Withdrawal

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6755:
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
  STATUSLIST: I-D.ietf-oauth-status-list
  SSF:
    target: https://openid.net/specs/openid-sharedsignals-framework-1_0.html
    title: "OpenID Shared Signals Framework Specification 1.0"
  CAEP:
    target: https://openid.net/specs/openid-caep-1_0.html
    title: "OpenID Continuous Access Evaluation Profile 1.0"

--- abstract

This specification defines how the status of a software statement for a Client ID Metadata Document is published and checked. Token Status List supplies the authoritative status: an issuer publishes it, and a trusting authorization server resolves it on the list's own schedule and refuses a withdrawn statement. Optional Shared Signals events tell an issuer's trusting authorization servers that a status has changed, prompting them to check at once. An event indicates only when to check, so a server that misses every event reaches the same result on its ordinary schedule.

--- middle

# Introduction

{{STATEMENT}} lets an issuer withdraw a statement's decision before the statement's expiry but requires and defines no withdrawal mechanism. This specification defines one, and an optional event that prompts a trusting authorization server to consult it at once. {{token-status-list}} profiles Token Status List {{STATUSLIST}}: an issuer withdraws a decision by setting the statement's status, and a trusting authorization server learns of the change when it next resolves the list.

A withdrawal therefore takes effect only as quickly as trusting authorization servers resolve the list. As an optional addition, this specification defines a Shared Signals Framework {{SSF}} event by which an issuer tells the trusting authorization servers that have configured it ({{STATEMENT}}) that a status has changed, prompting them to resolve that status at once. An event carries no status, and the status list remains the authority on whether a statement stands ({{processing}}).

The requirements of {{relationship}} through {{configuration}} apply only to implementations that support the events. This specification defines no new endpoint, transport, subject identifier format, or trust establishment mechanism. {{STATEMENT}} does not depend on this specification.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Transmitter, Receiver, Stream, and the delivery and configuration mechanisms are defined by {{SSF}}. Security Event Token (SET) is defined by {{RFC8417}}. Subject identifier formats are defined by {{RFC9493}}. Status List Token, and the validation that resolves a status, are defined by {{STATUSLIST}}. The terms Issuer, Trusting Authorization Server, Runtime Presentation, Effective Expiry, and Refusal Record are defined by {{STATEMENT}}, as are the software statement, its claims, its validation, issuer trust configuration, and withdrawal.

For the events defined here, the issuer acts as a Transmitter, and a trusting authorization server that has configured it acts as a Receiver.

# Token Status List {#token-status-list}

An issuer can use Token Status List {{STATUSLIST}} as a withdrawal mechanism ({{STATEMENT}}) for its statements. A trusting authorization server can resolve those statuses for any issuer it has configured.

## Status Publication {#status-publication}

An issuer that uses Token Status List publishes the status of its statements as {{STATUSLIST}} defines, and each statement's `status` claim ({{STATEMENT}}) locates its entry in the issuer's Status List Token. An issuer that publishes status:

* MUST publish status, and carry the `status` claim, for every statement it issues under a given `iss` from the point it begins publishing. Once statements issued before that point have expired, a trusting authorization server can read an absent claim as meaning the issuer publishes no status.
* MUST NOT assign the same combination of status list URI and index to more than one statement, including across successive Status List Tokens published at that URI. Withdrawing a statement then affects no other: withdrawing a superseded statement does not withdraw its replacement, and issuing a replacement does not restore a withdrawn predecessor.
* MUST sign the Status List Token with a key published at the `jwks_uri` metadata value of its authorization server metadata {{RFC8414}}, which {{STATEMENT}} keeps apart from its statement signing keys.
* MUST include the `exp` claim in every Status List Token it publishes, to prevent an older token from standing in for a newer one indefinitely.
* MUST give each Status List Token it publishes at a given list URI an `iat` claim value later than that of any token it published there before ({{status-resolution}}).

## Status Resolution {#status-resolution}

If a statement carries the `status` claim and the trusting authorization server resolves statuses for the statement's issuer, the server MUST reject the statement if its status is `INVALID`, and MUST apply its configured policy for that issuer to the statement if its status is `SUSPENDED` or any other value except `VALID` and `INVALID`.

A status of `INVALID` is a withdrawal, as is any other status except `VALID` when the policy configured for that issuer refuses the statement. On a withdrawal, the server holds a refusal record ({{STATEMENT}}). Token Status List defines no expiry for a withdrawal, so the refusal ends only when a later resolution returns a status that no longer refuses the statement.

Resolution follows {{STATUSLIST}}. In particular:

* If the referenced index lies outside the list, the server rejects the statement as {{STATUSLIST}} requires, with the error code that {{STATEMENT}} assigns to a statement refused by a refusal record.
* The caching rules of {{STATUSLIST}} apply to resolution, and also apply when the server checks a recorded statement at refresh ({{STATEMENT}}).

To prevent a host serving an older token from restoring a status the issuer has withdrawn, a server MUST NOT replace a Status List Token it holds with one whose `iat` claim is earlier, and MUST treat one whose `iat` claim is equal but whose contents differ as a resolution failure. A statement whose status the server cannot resolve is bounded by its effective expiry, as {{STATEMENT}} provides.

A server MUST obtain a Status List Token's verification keys from the `jwks_uri` metadata value of the issuer's authorization server metadata {{RFC8414}}, discovered from the configured `iss`, and never from the Status List Token itself.

# Event Issuer Trust and Keys {#relationship}

A trusting authorization server MUST NOT accept an event from an issuer it has not configured. It MUST verify the SET using keys from the `jwks_uri` member of the issuer's Transmitter configuration {{SSF}}, discovered from the issuer identifier it has configured, and never from a key-location value carried in the event. The SET's `iss` claim is that issuer identifier, which is also the `iss` claim value of the statements the event concerns.

An issuer MUST ensure that its Transmitter configuration's `jwks_uri` member differs both from the issuer's `software_statement_jwks_uri` ({{STATEMENT}}) and from the `jwks_uri` metadata value in its authorization server metadata {{RFC8414}}, whose keys verify its Status List Tokens. A receiver MUST NOT create or keep a stream whose Transmitter configuration names either of those locations.

A SET carrying an event defined here MUST use the explicit `typ` JOSE header parameter value `secevent+jwt` (Section 2.3 of {{RFC8417}}). Its `aud` claim MUST contain the audience of the stream that delivers it, which {{SSF}} lets the transmitter and receiver agree on. A receiver MUST NOT create or keep a stream whose audience does not identify it.

# Subject Identification {#subjects}

The subject of every event defined here is client software. Events carry it in the `sub_id` claim of {{SSF}} using the `uri` subject identifier format of {{RFC9493}}. The `uri` member carries the Client ID Metadata Document URL {{CIMD}} exactly as it appears in the `sub` claim of the statements the event concerns.

A trusting authorization server MUST match the subject by exact comparison of the `uri` member against a statement's `sub` claim.

# Event Types {#events}

The payload ({{RFC8417}}) of an event defined here has the following claims:

`event_timestamp`:
: OPTIONAL. The claim defined by {{CAEP}}, a NumericDate value that here gives the time at which the issuer changed the status the event reports. It is informational: a receiver does not use it to order, bound, or scope anything.

`software_statement_jti`:
: OPTIONAL. The `jti` claim value of a single statement whose status changed ({{processing}}).

`reason_admin`:
: OPTIONAL. An administrative message, as {{CAEP}} defines the claim.

## Status Changed {#status-changed}

The event type identifier is `urn:ietf:params:oauth:event-type:software-statement-status-changed` ({{iana-event-type}}).

The event reports that the issuer has changed the published status of one or more statements for the subject. A transmitter MUST NOT transmit the event until a Status List Token reflecting the change is retrievable at the status list URI, including through any caching layer it operates. A receiver resolving before then would fetch the state the event exists to correct.

The event does not say what the new status is, and a receiver MUST NOT infer one from it. A receiver that resolves a status of `VALID` after receiving an event has applied that event correctly.

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

# Receiver Processing {#processing}

A trusting authorization server that receives an event defined here MUST process it as follows:

1. Verify the SET as {{RFC8417}} requires, including its `typ` JOSE header parameter, and verify that the SET's issuer is configured and that the keys verifying the SET were obtained as {{relationship}} requires.
2. Reject an event whose `aud` claim does not contain the audience of the stream that delivered it ({{relationship}}) or whose event type is not recognized.
3. Resolve the subject ({{subjects}}).
4. Resolve the status of the affected statements from the issuer's Status List Token as {{STATUSLIST}} defines, without waiting for the schedule that would otherwise apply, and apply the resolved status as {{status-resolution}} defines.

The affected statements are those the transmitting issuer made about the subject, never statements another issuer made about the same software. They are the statement the event's `software_statement_jti` claim names or, if the event names none, those the receiver holds or has cached a status for. An event about a subject for which the receiver holds no state requires nothing of it.

A receiver MUST fetch the Status List Token for the affected statements afresh rather than answer from a cached copy. A cached copy is what the event exists to correct. An event is never grounds for refusal. Until the fetch succeeds, the copy the receiver holds remains in effect within its validity. If resolution does not complete, the receiver applies the rules of {{STATEMENT}} as it would had no event arrived.

A receiver MUST treat an event it has already applied as successfully delivered and acknowledge it as {{RFC8935}} or {{RFC8936}} requires, rather than reporting a delivery error. Rejecting a retry can stall or disable a stream carrying later events. Duplicate detection is per transmitting issuer.

Two constraints bound every event:

* An event MUST NOT by itself create standing, extend a statement's lifetime, or otherwise increase what a client can do. A receiver MUST ignore any payload claim that would have such an effect.
* A receiver MUST continue to resolve status on its own schedule, and to enforce statement expiry, independently of this mechanism. Stream loss, transmitter unavailability, or delivery failure then leaves both controls in force.

# Stream Configuration {#configuration}

An issuer that supports the events defined here publishes Transmitter configuration metadata as {{SSF}} defines, discoverable from its issuer identifier. Stream creation, subject management, verification, and delivery follow {{SSF}}.

A trusting authorization server that supports these events SHOULD create one stream per configured issuer that offers them. The stream can cover every subject that issuer attests, or the server can add each subject through the subject management of {{SSF}} when it first admits that software. Admission checks and the receiver's own status resolution ({{status-resolution}}, {{processing}}) remain independent of the subjects a stream covers, so a withdrawal whose event was transmitted before the subject was added to the stream is found at a later resolution.

The subjects appropriate to a stream are those of the issuer's statements whose `aud` claim is absent or names the receiving authorization server. A transmitter SHOULD scope each stream to those subjects. A receiver discards events for subjects outside the identifier scope for which it accepts that issuer ({{STATEMENT}}).

A receiver SHOULD request the event this specification defines, and SHOULD use the stream verification facility of {{SSF}} periodically.

# Security Considerations

The security considerations of {{STATEMENT}}, {{STATUSLIST}}, {{SSF}}, and {{RFC8417}} apply.

## Status Resolution Schedule {#resolution-schedule}

Because {{STATUSLIST}} aggregates many statements into one list, a retrieval tells the issuer only that some trusting authorization server is checking. A server SHOULD retrieve the list on the list's own schedule, within the expiration and caching bounds its `exp` and `ttl` claims set (Section 11.5 of {{STATUSLIST}}), rather than once per request. The server's request timing then does not disclose the client population it serves, and each request a statement governs does not depend on issuer availability.

## What an Event Cannot Do

An event names no status, so a forged, replayed, delayed, or reordered event cannot change whether any statement stands. Its only effect is a resolution the receiver would otherwise have performed on its own schedule.

The remaining exposure is resource cost: an attacker holding the issuer's SET signing key, or a misbehaving transmitter, can trigger resolutions. A receiver SHOULD bound the rate at which it resolves in response to events, coalescing events for the same issuer.

## Status Remains the Authority

Because a receiver resolves status independently ({{processing}}), an attacker who suppresses events delays a withdrawal at most until the receiver's next scheduled resolution and, for new admissions, never beyond the affected statements' effective expiry. Grants already open continue as the refresh policy of {{STATEMENT}} allows. Deployments therefore choose a resolution schedule and statement lifetimes they would accept with no event stream.

A receiver SHOULD alert on stream loss rather than assume quiescence. The absence of events looks the same on a healthy stream and a suppressed one. Periodic stream verification ({{configuration}}) makes the difference observable.

## Key Separation and Compromise

The distinct key set locations of {{relationship}} separate SET keys from statement and Status List Token keys only while the transmitter publishes no SET key in either of the other key sets. With the keys separated, compromise of a SET key lets an attacker trigger resolutions and nothing more. Compromise of a statement signing key has wider effect, because statements grant standing and events do not.

# Privacy Considerations

The privacy considerations of {{STATEMENT}}, {{STATUSLIST}}, {{SSF}}, and {{RFC8417}} apply. Retrieving a Status List Token on its own schedule keeps request timing from disclosing the client population a server serves ({{resolution-schedule}}).

Subject identifiers in these events name reviewed software and, in aggregate, describe an organization's approved software estate. For this reason, a stream is scoped as {{configuration}} provides. A receiver SHOULD apply to event logs the handling it applies to statements ({{STATEMENT}}).

Because this specification defines no durable receiver-side record, it adds no negative state about a client identifier that outlives the issuer's own published status. A receiver that logs events creates such a record, and SHOULD bound its retention accordingly.

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

Under Section 2 of {{RFC8417}}, the `software_statement_jti` payload claim of {{events}} is defined by this specification and need not be registered as a JWT claim. No IANA registry of SET payload claims exists, so no registration is requested.

--- back

# Acknowledgments
{:numbered="false"}

This profile draws on the Shared Signals Framework and the Continuous Access Evaluation Profile for its transport and payload conventions.
