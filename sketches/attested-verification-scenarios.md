# Verification Scenarios for Attested Instance Keys

An implementation checklist for [the attested draft](../draft-mcguinness-oauth-cimd-sw-stmt-attested.md), moved out of the draft before -00. Expected results use the error codes that draft assigns, and list each alternative it permits, so that a conforming implementation choosing one does not fail the check; where this list and the draft differ, the draft governs.

## Prerequisites

Every scenario starts from this setup unless it says otherwise:

* An authorization server implementing the attested draft, with PAR, PKCE using `S256`, DPoP with server-issued nonces, and the `attest_jwt_client_auth_dpop` method.
* A statement issuer the server trusts, and an attester and evidence policy it accepts for the client identifier and tenant.
* A reviewed Client ID Metadata Document that selects the profile, declares `attest_jwt_client_auth_dpop`, lists `authorization_code` and `refresh_token` in `grant_types`, carries no `jwks` or `jwks_uri`, and lists only loopback redirection URIs the profile allows.
* A current statement over that document's digest, presented by the client or published for the server to pull.
* A client instance holding a key the attester has attested for that client identifier and digest.

## Scenarios

| # | Scenario | Setup | Expected result |
| --- | --- | --- | --- |
| 1 | Admission | PAR with a current attestation and a DPoP proof under the attested key; redemption with the PKCE verifier and the same key | PAR response, then DPoP-bound tokens; an establishment records the key, attester, policy, and tenant |
| 2 | Refresh | Refresh with a renewed attestation for the same key, attester, and policy while the review is current | New tokens bound to the same key |
| 3 | Copied statement, no attestation | A PAR for a document that selects the profile, carrying a valid statement but no attestation for the presenter's key | `invalid_client`; no establishment |
| 4 | Copied attestation | A valid attestation with a DPoP proof under a different key | `invalid_client`, or `invalid_client_attestation`; no establishment |
| 5 | Mismatched attestation | At the PAR endpoint: an attestation for another client identifier, or from an attester or under a policy not accepted for this client identifier or tenant | `invalid_client`, or `invalid_client_attestation` |
| 6 | Attestation over another digest | An attestation naming an earlier digest while the served document matches the statement | `use_fresh_attestation`; the client retries with a fresh attestation |
| 7 | Intercepted loopback code | Another process tries to redeem the code | Instance authentication comes first: `invalid_client`, or `invalid_client_attestation`, if it cannot authenticate under the attested key; `invalid_grant` if it authenticates under an attested key of its own, or lacks the PKCE verifier; no tokens |
| 8 | Replayed or misdirected proof | A DPoP proof reused, or made for another endpoint or method | `invalid_dpop_proof` |
| 9 | Nonce retry | A request without the current server nonce | `use_dpop_nonce` with a `DPoP-Nonce` header; no code consumed, no refresh token rotated, no establishment created, `iat` floor unchanged |
| 10 | Replacement without a matching attestation | Refresh with a replacement over a new digest and an attestation naming the old one | `use_fresh_attestation`; the establishment and refresh token are unchanged |
| 11 | Outstanding code after the document changed | Two PAR transactions for the same client create two establishments. The first is redeemed. The publisher changes the document, and the first grant refreshes onto a replacement over the new digest. The second establishment's code is then redeemed | `invalid_grant`: the replacement applies only to the first grant, so the second establishment still records the old digest, and redemption finds that the live document no longer matches it. The client restarts at the PAR endpoint |
| 12 | Control for 11 | Two establishments as in 11, but the document is unchanged and a newer statement has only raised the `iat` floor | Redemption succeeds: the recorded statement is unexpired and not refused, the digest still matches, and a raised floor alone does not invalidate the code |
| 13 | Takeover of a grant | A refresh or redemption by an instance that authenticates successfully, under an attester and policy the server accepts for the client identifier and tenant, but whose key, attester, policy, or tenant differs from the establishment's | `invalid_grant`: it does not match a fixed establishment value. An attester or policy the server does not accept fails authentication first, with `invalid_client` or `invalid_client_attestation` |
| 14 | Withdrawal or trust removal | The statement is withdrawn, its issuer is no longer trusted, or the attester or policy is no longer accepted, then the client refreshes with a fresh proof | `statement_required` for a withdrawn statement or an untrusted issuer; `invalid_client`, or `invalid_client_attestation`, for an attester or policy no longer accepted |
| 15 | Required binding | Server policy requires the binding for the client namespace or tenant, and a request uses another authentication method, skips PAR with a direct authorization request, or uses another grant type | Rejected rather than admitted under the policy for unreviewed clients: `invalid_client` for another authentication method, `invalid_request` at the authorization endpoint, `unauthorized_client` for another grant type |
| 16 | Superseded statement | At the PAR endpoint, a statement over an earlier document whose `iat` is below the retained `iat` floor | `statement_required`: the floor is checked before any retrieval, so the digest is never compared |
| 17 | Statement over a changed document | At the PAR endpoint, a statement the `iat` floor does not supersede, over an earlier version of the document than the one served | `invalid_client`: the digest does not match, and under the required binding the server does not proceed without the statement |
| 18 | Local process through the launcher | An unapproved local process asks the trusted launcher to sign or to run a transaction | Deployment check, not a protocol result: the launcher refuses, and no proof or token reaches the process |
