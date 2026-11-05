---
title: "Client ID Metadata Document Digests in Identity Assertion JWT Authorization Grants"
abbrev: oauth-cimd-sw-stmt-idjag
docname: draft-mcguinness-oauth-cimd-sw-stmt-idjag-latest
category: std
submissiontype: IETF

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Client ID Metadata Document
 - Identity Assertion JWT Authorization Grant
 - ID-JAG
 - Software Statement

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
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  IDJAG: I-D.ietf-oauth-identity-assertion-authz-grant
  STATEMENT: I-D.mcguinness-oauth-cimd-sw-stmt

informative:
  RFC7523:

--- abstract

This specification defines how an identity provider that issues an Identity Assertion JWT Authorization Grant (ID-JAG) binds the grant to the exact octets of the client's Client ID Metadata Document (CIMD), using the `cimd_digest` claim that a companion software statement specification defines. A resource authorization server that trusts the identity provider for this purpose can then admit a client it holds no registration for, on the identity provider's decision, with the client's metadata taken from the document whose digest matches. The admission covers only the token request that redeems the grant, and it creates no registration and no refresh token.

--- middle

# Introduction

An Identity Assertion JWT Authorization Grant {{IDJAG}} lets a client obtain, from an identity provider that two trust domains share, an assertion of a user's identity and a delegation to a resource authorization server. The grant names the client in its `client_id` claim. A client identified by a Client ID Metadata Document (CIMD) {{CIMD}} uses its client identifier URL at both servers, so no mapping between them is needed (Section 5.1 of {{IDJAG}}).

The grant tells the Resource Authorization Server that the identity provider issued it for that client. It does not say which client metadata that decision covered. The Resource Authorization Server retrieves the client's document itself, and that document can differ from the one the identity provider relied on. A Resource Authorization Server that admits CIMD clients only when a party it trusts has evaluated them has no such party in the request.

{{STATEMENT}} defines a software statement that binds an issuer's review to the octets of a CIMD through the `cimd_digest` claim. A statement serves when the reviewer is not in the request path (Section 10.2 of {{STATEMENT}}). When the reviewer is the identity provider that issues the grant, the identity provider can carry the same claim in the grant it already signs.

This specification defines the claim's meaning in an ID-JAG and when an identity provider includes it ({{claim}}), and how a Resource Authorization Server admits a client on it ({{admission}}). At a minimum, the claim shows that the identity provider issued the grant with these octets as the client's metadata. The claim does not by itself show that the identity provider reviewed the configuration. Any further evaluation is part of the agreement under which the Resource Authorization Server designates the identity provider ({{issuer-designation}}). An ordinary ID-JAG redemption can rely on a sender-constrained ID-JAG instead of client authentication (Section 4.4 of {{IDJAG}}), but a vouched admission requires the client to authenticate at the Resource Authorization Server with a key its document carries, as it did at the identity provider ({{client-authentication}}).

This specification extends {{STATEMENT}} and {{IDJAG}} and changes neither. A server that does not implement it ignores the claim (Section 4 of {{RFC7519}}).

## Protocol Overview

The following non-normative sequence summarizes a vouched admission:

1. The client authenticates to the IdP Authorization Server in a Token Exchange request (Section 4.3 of {{IDJAG}}), using its client identifier URL and a key its document carries.
2. If its policy permits issuing an ID-JAG for the client, the IdP Authorization Server includes the `cimd_digest` claim, computed over the document from which it took that key ({{claim}}).
3. The client presents the ID-JAG at the Resource Authorization Server's token endpoint (Section 4.4 of {{IDJAG}}), authenticating with a key the document carries.
4. The Resource Authorization Server validates the ID-JAG, retrieves the document at the URL in the `client_id` claim, and compares its digest with the claim. If the digests match and its trust configuration designates the issuer, it admits the client with metadata taken from that document ({{admission}}).
5. The admission covers that request only. A re-submission of the ID-JAG repeats step 4, and a later ID-JAG carries the digest of the document served at that time.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology from {{RFC6749}} (OAuth), {{CIMD}} (Client ID Metadata Documents), and {{IDJAG}} (IdP Authorization Server, Resource Authorization Server, and ID-JAG), and the term Permitted Method defined in Section 2 of {{STATEMENT}}.

This specification defines the following terms.

Vouched Document:
: The CIMD whose metadata digest (Section 3.1 of {{STATEMENT}}) an ID-JAG carries in its `cimd_digest` claim.

Designated Issuer:
: An IdP Authorization Server, and for a multi-tenant issuer one of its tenants, that a Resource Authorization Server's trust configuration designates for vouched admission in one of the server's tenants ({{trust-configuration}}).

Vouched Admission:
: A Resource Authorization Server's acceptance of a client for a token request on the `cimd_digest` claim of an ID-JAG from a designated issuer ({{admission}}).

# Metadata Digest Claim {#claim}

An ID-JAG MAY contain the `cimd_digest` claim (Section 3 of {{STATEMENT}}), whose value is the metadata digest of the client's CIMD. The claim asserts that the IdP Authorization Server's decision to issue the ID-JAG for the client covered the document with those octets. This specification does not define how the IdP Authorization Server reaches that decision ({{issuer-designation}}).

An IdP Authorization Server MUST NOT include the claim unless all of the following hold:

* The ID-JAG's `client_id` claim is a client identifier URL (Section 3 of {{CIMD}}) and is the client identifier under which the client authenticated in the Token Exchange request (Section 4.3.3 of {{IDJAG}}), not one mapped from it (Section 5.1 of {{IDJAG}}).
* In that request, the client authenticated with a method that proves possession of a key the document carries (Section 5.6.1 of {{STATEMENT}}), such as `private_key_jwt`.
* The claim's value is the metadata digest of the octets of the document that carries that key, in its `jwks` member or at its `jwks_uri`, as the IdP Authorization Server obtained them for that authentication, not of octets from another retrieval.

The client therefore uses its client identifier URL at the IdP Authorization Server as well, including for the identity assertion or refresh token it exchanges (Section 4.3.3 of {{IDJAG}}).

A Resource Authorization Server compares the digest with the document as Section 5.2 of {{STATEMENT}} obtains it: by retrieval, or from octets it holds within the bounds of Section 5.3 of {{STATEMENT}}, retrieving the document once more if held octets do not match. A server that requires a vouched admission rejects a digest that still does not match ({{errors}}), so a digest of an earlier document fails once the server's copy reflects the change. When an IdP Authorization Server authenticates a client for an ID-JAG that will carry the claim, it SHOULD therefore use the document as currently served, revalidating any octets it holds (Section 3.1 of {{STATEMENT}}), including octets held from pre-registering the client identifier URL (Section 9.2 of {{CIMD}}).

The claim changes no other processing of the ID-JAG at either server.

# Vouched Admission {#admission}

## Trust Configuration {#trust-configuration}

A Resource Authorization Server MUST NOT rely on the claim unless its trust configuration, established out of band, designates the ID-JAG's issuer for vouched admission in the tenant in which the server accepts the ID-JAG (Section 4.4.1 of {{IDJAG}}). For a multi-tenant issuer, the designation names the issuer and its tenant together (Section 6.1 of {{IDJAG}}). The server MUST NOT derive the designation from the ID-JAG or from the client's document. Trusting an issuer for ID-JAGs does not by itself designate it.

Vouched admission applies to a CIMD client the server holds no registration for. For a registered client, the registration governs and the server ignores the claim, as does a server that does not rely on it. A server's policy can require a vouched admission for a CIMD client it holds no registration for ({{errors}}).

## Processing Rules {#processing}

A Resource Authorization Server that relies on the claim processes a token request redeeming an ID-JAG, for a CIMD client it holds no registration for, as follows, rejecting at the first failure ({{errors}}):

1. Validate the ID-JAG as Section 4.4 of {{IDJAG}} requires, including the verification of any `cnf` claim, except for the checks that depend on client authentication: the client continuity check of Section 4.4.1 of {{IDJAG}}, and the rejection in Section 4.4 of {{IDJAG}} of a request that has neither client authentication nor a sender-constrained ID-JAG. The server completes these deferred checks before it issues an access token on any path.
2. Determine whether the ID-JAG carries the claim and its issuer is a designated issuer for the tenant ({{trust-configuration}}). If not, proceed without the claim.
3. Obtain the document at the URL in the `client_id` claim and validate it as {{CIMD}} requires. The server MUST reject a document containing duplicate member names, because parsers interpret such names differently despite an identical digest (Section 5.2 of {{STATEMENT}}). Then compare its digest with the claim, as Section 5.2 of {{STATEMENT}} requires for a statement's `sub` and `cimd_digest` claims. If the digests do not match, proceed without the claim.
4. If the vouched document permits no method other than `none` that the server supports and its policy allows, proceed without the claim. Otherwise, the server MUST require the client to authenticate with such a method, proving possession of a key the vouched document carries (Section 5.6.1 of {{STATEMENT}}; {{client-authentication}}), and then completes the deferred checks. For `private_key_jwt`, Section 5 of {{CIMD}} already requires this authentication.
5. The server MUST reject the request unless its grant type appears in the vouched document's `grant_types` member, including the grant type that Section 4.4 of {{IDJAG}} requires for an ID-JAG bound to a DPoP key. A document that omits the member permits only `authorization_code` (Section 2 of {{RFC7591}}), so it permits no redemption of an ID-JAG.
6. If the vouched document has a `scope` member, the server MUST NOT issue an access token whose scope falls outside it. The server narrows the scope it would otherwise grant (Section 4.4.1 of {{IDJAG}}) to meet this rule. Authorization details that the server derives from a scope this rule removes (Section 4.4.1 of {{IDJAG}}) are removed with it. Other authorization details the ID-JAG carries are granted as Section 4.4.1 of {{IDJAG}} and the server's policy decide, so a grant expressed only through authorization details, or one that grants no scope, is unaffected. If the server would otherwise grant some scope, narrowing removes all of it, and no authorization details remain to grant, the server MUST reject the request.

To proceed without the claim, the server obtains the document if it has not already done so and applies its policy for CIMD clients it holds no registration for, as if the ID-JAG carried no claim. The remaining steps and the refresh-token rule below then do not apply, but the deferred checks of step 1 still do. If that policy requires a vouched admission, the server rejects the request ({{errors}}).

The server MUST NOT issue a refresh token in response to a request it admits on the claim, even if the request also presents a statement ({{relationship}}). A vouched admission does not admit a later request, and a record the server keeps of it, such as for audit, does not change that. A later request, including a re-submission of the same ID-JAG (Section 4.4.4 of {{IDJAG}}), repeats this processing, and can use document octets the server holds within the bounds of Section 5.3 of {{STATEMENT}}.

# Error Responses {#errors}

A rejection uses the error response of Section 5.2 of {{RFC6749}}. A failure of the validation that Section 4.4 or Section 9.11 of {{IDJAG}} requires uses the error code that {{IDJAG}} assigns.

For a rejection this specification defines, the server uses `temporarily_unavailable` if a retrieval of the document, or of the key set at its `jwks_uri`, did not complete (Section 5.3 of {{STATEMENT}}), and responds with HTTP status code 503, as Section 6 of {{STATEMENT}} requires. Otherwise, it uses the following error codes:

`invalid_client`:
: Any of the following:

  * The document cannot be obtained or fails validation as {{CIMD}} requires (Section 6.1 of {{CIMD}}), or contains duplicate member names ({{processing}}).
  * The client does not authenticate as step 4 of {{processing}} requires.

`invalid_grant`:
: The server's policy requires a vouched admission, the ID-JAG's issuer is a designated issuer for the tenant, and the ID-JAG carries no claim or a claim whose digest does not match. A re-issued ID-JAG may resolve the error (Section 4.4.3 of {{IDJAG}}), because the IdP Authorization Server can include the claim, or compute it over the document it then holds.

`access_denied`:
: The server's policy requires a vouched admission, and either the ID-JAG's issuer is not a designated issuer for the tenant ({{trust-configuration}}) or the vouched document permits no method that step 4 of {{processing}} allows. A re-issued ID-JAG will not resolve the error.

`unauthorized_client`:
: The request's grant type is not in the vouched document's `grant_types` member.

`invalid_scope`:
: The server would otherwise grant some scope, narrowing under step 6 of {{processing}} removes all of it, and no authorization details remain to grant.

# Examples {#example}

## Vouched Document

The following is an example of a client's document, served at `https://agent.example/client` exactly as shown, with two-space indentation, LF line endings, and a single final newline, and without any indentation added for display (657 octets):

~~~ json
{
  "client_id": "https://agent.example/client",
  "client_name": "Example Agent",
  "grant_types": [
    "urn:ietf:params:oauth:grant-type:token-exchange",
    "urn:ietf:params:oauth:grant-type:jwt-bearer"
  ],
  "token_endpoint_auth_methods_supported": ["private_key_jwt"],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "scope": "chat.read chat.history",
  "jwks": {
    "keys": [
      {
        "kty": "EC",
        "crv": "P-256",
        "kid": "2026-11",
        "x": "MKBCTNIcKUSDii11ySs3526iDZ8AiTo7Tu6KPAqv7D4",
        "y": "4Etl6SRW2YiLUrN5vfvVHuhp7x8PxltmWWlbbM4IFyM"
      }
    ]
  }
}
~~~

## ID-JAG Payload

The following is an example of the payload of an ID-JAG that the IdP Authorization Server issues after the client authenticates with `private_key_jwt`, using the key in the document's `jwks` member. The `cimd_digest` claim is the metadata digest of the document in the previous example.

~~~ json
{
  "jti": "9e43f81b64a33f20116179",
  "iss": "https://acme.idp.example/",
  "sub": "U019488227",
  "aud": "https://acme.chat.example/",
  "client_id": "https://agent.example/client",
  "exp": 1793404800,
  "iat": 1793404500,
  "resource": "https://api.chat.example/",
  "scope": "chat.read chat.history",
  "cimd_digest": "hoLujUutrork06uug7i0WF3pAiM9IHIUwepB_X_TOMQ"
}
~~~

## Token Request

The following is an example of the client redeeming that ID-JAG with the JWT bearer grant (Section 2.1 of {{RFC7523}}) at a Resource Authorization Server that holds no registration for the client (line breaks for display purposes only):

~~~ http
POST /oauth2/token HTTP/1.1
Host: acme.chat.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=eyJ0eXAiOiJvYXV0aC1pZC1qYWcrand0IiwiYWxnIjoiRVMyNTYi...
&client_id=https%3A%2F%2Fagent.example%2Fclient
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ajwt-bearer
&client_assertion=eyJ0eXAiOiJjbGllbnQtYXV0aGVudGljYXRpb24rand0Ii...
~~~

The Resource Authorization Server validates the ID-JAG, retrieves the document, and finds that its digest matches the claim. It verifies the client assertion against the key in the document's `jwks` member, checks that the document's grant types include the JWT bearer grant, and issues an access token whose scope falls within the document's `scope`. It issues no refresh token and keeps no registration.

# Relationship to Software Statements {#relationship}

A software statement {{STATEMENT}} and the claim this specification defines both bind a decision to the octets of a client's document. They differ in who carries the decision and how often. A statement is issued once for software and presented or pulled at many servers, so it serves a reviewer that is not in the request path, such as a client catalog. The claim rides a grant that the IdP Authorization Server issues per user and request, so it serves when the reviewer is the identity provider that issues the grant. Such an identity provider could instead issue statements confined to a tenant with `aud_tenant`, but the claim needs no second artifact and no separate issuance step.

A Resource Authorization Server MAY require both. A token request that redeems an ID-JAG can also present a statement (Section 5.4.1 of {{STATEMENT}}). Each is then processed under its own specification, neither substitutes for the other, and the refresh-token rule of {{processing}} still applies. A vouched admission does not satisfy a required issuer (Section 4 of {{STATEMENT}}), because it is not a statement.

# Security Considerations

## Assurance Provided by a Vouched Admission

The following table states what a vouched admission does and does not establish:

| Property | Established |
| --- | --- |
| A designated issuer decided to issue the ID-JAG for this client with this document | Yes |
| The document the server retrieved is the one that decision covered, octet for octet | Yes |
| The presenter controls a key the document carries | Yes, before any token is issued. A `jwks_uri` attests only the keys' location ({{key-locations}}) |
| The issuer reviewed the configuration beyond deciding to issue the grant | Only as the designation agreement provides ({{issuer-designation}}) |
| The running software is the software the issuer evaluated | No |
| The client may access a particular resource | No; the ID-JAG's authorization and the server's policy decide |
| The document's redirection URIs are approved for authorization requests | No; a vouched admission covers only the token request that redeems the ID-JAG |

## Client Authentication at Both Servers {#client-authentication}

Possession of the key bound to a sender-constrained ID-JAG shows that the presenter holds the key the IdP Authorization Server bound to the ID-JAG for the client (Section 9.1 of {{IDJAG}}), and that key need not appear in the client's document. A vouched admission takes the client's metadata from the vouched document, so it requires the presenter to prove a key that same document carries ({{processing}}). The Resource Authorization Server then verifies for itself that the presenter controls the client the document describes, instead of relying on the IdP Authorization Server's earlier authentication. The requirement does not duplicate the IdP Authorization Server's check. For `private_key_jwt`, Section 5 of {{CIMD}} already requires the same check of a server that supports the method when a document declares it. For other methods, such as `self_signed_tls_client_auth`, this specification adds it.

## Issuer Designation {#issuer-designation}

The claim does not say how the IdP Authorization Server evaluated the document. At a minimum, it shows that the IdP Authorization Server's policy let this client, with this document as its metadata, receive this grant. An IdP Authorization Server that approves clients by identifier and digests whatever document is served at the time gives less assurance than one that pins its approval to a document it reviewed. A Resource Authorization Server learns which applies from the agreement under which it designates the issuer, as the assurance of a statement rests on its issuer's review process (Section 13.1 of {{STATEMENT}}).

Designating an issuer extends to client software the trust a Resource Authorization Server already places in it for users. A designated issuer can admit any CIMD client in the tenant. A server that wants to confine it to particular software does so by policy, for example by refusing documents outside client identifier namespaces it configures, as Section 4 of {{STATEMENT}} does for statement issuers.

## Changed and Split Documents

The digest comparison detects a document that changed between the two servers' retrievals. It also detects a document served differently to the two, which Section 4.5 of {{CIMD}} forbids. A changed document fails the comparison until the IdP Authorization Server issues an ID-JAG over the current one.

If the IdP Authorization Server keeps digesting octets it held before the change, its ID-JAGs keep failing at a server that requires a vouched admission. The client requests one new ID-JAG and then stops (Section 4.4.4 of {{IDJAG}}), so the failure lasts as long as the IdP Authorization Server reuses the earlier octets ({{claim}}).

An ID-JAG's lifetime bounds how long it can admit the client: after its `exp`, it admits no further request. A Resource Authorization Server can shorten that bound by rejecting an ID-JAG whose `exp` is later than its policy allows (Section 4.4.4 of {{IDJAG}}). An IdP Authorization Server ends its vouching for a document by issuing no further ID-JAG with that document's digest, and an ID-JAG already issued remains usable until it expires or the Resource Authorization Server refuses its re-submission.

Neither expiry nor the end of vouching invalidates an access token already issued, which remains usable for its own lifetime. The prohibition of refresh tokens does not change that. A server that needs access to end when the ID-JAG expires limits the lifetime of the access tokens it issues on the claim.

## Key Locations {#key-locations}

A document's `jwks_uri` member is attested as a location, not as contents (Section 13.2 of {{STATEMENT}}). The IdP Authorization Server and the Resource Authorization Server can obtain different key sets from the same location without any change to the digest. Requiring both servers to take keys from the vouched document does not prevent this. A document that carries its keys inline in `jwks`, as in {{example}}, does.

## Retrieval

Every retrieval this specification causes inherits the protections of Section 5.3 of {{STATEMENT}}.

# Privacy Considerations

The claim is the digest of a document that is already public. It tells the Resource Authorization Server nothing about the user, and nothing about the client beyond the document the server can retrieve itself. The server's retrieval of the document is the same as for any CIMD client and has the same side channel (Section 11.1 of {{CIMD}}).

# IANA Considerations

This specification requests that IANA update the `cimd_digest` entry that {{STATEMENT}} registers in the "JSON Web Token Claims" registry established by {{RFC7519}}, so that its description covers a digest carried in a JWT other than a software statement. The digest algorithm and encoding are unchanged.

Claim Name:
: `cimd_digest`

Claim Description:
: Unpadded base64url-encoded SHA-256 digest of the retrieved octets of a Client ID Metadata Document that the JWT's issuer vouches for

Change Controller:
: IETF

Specification Document(s):
: Section 3 of {{STATEMENT}}, and {{claim}} of this specification

--- back

# Acknowledgments
{:numbered="false"}

This specification follows working-group discussion of how an identity provider issuing an ID-JAG could vouch for a client's metadata.

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-00

* Initial version.
