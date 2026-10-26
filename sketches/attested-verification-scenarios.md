# Verification Scenarios for Attested Instance Keys

Test cases for draft-mcguinness-oauth-cimd-sw-stmt-attested, moved out of the draft before -00.

An implementation exercise should include the following cases in addition to successful admission and refresh:

* A copied statement with an unattested key fails before creating an establishment.
* A copied attestation with a different proof key fails authentication.
* A valid attestation for another client, digest, tenant authorization, or policy fails its corresponding binding check.
* A loopback code intercepted by another process cannot be redeemed without both the verifier and recorded key.
* A repeated proof and a proof intended for another endpoint are rejected.
* A nonce retry neither consumes grant state nor advances the `iat` floor.
* A refresh with a renewed attestation for the original key, attester, and policy succeeds when the review remains current.
* A metadata replacement without a matching current attestation fails without changing the establishment.
* A code outstanding while another grant sharing its establishment refreshes onto a replacement over new metadata is rejected at redemption, and the client restarts at the PAR endpoint.
* A new key, attester, policy, or tenant cannot take over an existing grant.
* Withdrawal or removal of trust prevents continuation despite a fresh proof.
* Omitting attestations or using direct authorization cannot bypass a server requirement for this binding.
* An unapproved local process cannot obtain usable signatures or tokens through the trusted launcher.
