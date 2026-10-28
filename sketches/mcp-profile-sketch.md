# Design Rationale: Pulled Statements and the MCP Profile

The reasoning behind two things now specified elsewhere: pulled statements, which the statement draft defines, and the Model Context Protocol profile drafted as [Reviewed Client Software](../mcp/reviewed-client-software.mdx). It is not normative, and where it differs from those documents, they govern. It began because MCP prefers Client ID Metadata Documents for identifying clients, and the family as it then stood asked more of an MCP client than any MCP client will do.

Claims here are of three kinds, and the text says which where it matters: **defined by the drafts** (a draft in this repository specifies it), **deployment policy** (a choice the drafts leave to an operator), or **proposed extension** (something no draft defines yet).

## Why MCP, and why now

The MCP authorization specification of 2026-07-28 recommends identifying clients by Client ID Metadata Document, which servers and clients SHOULD support, and says plainly that "Dynamic Client Registration is deprecated and retained for backwards compatibility with authorization servers that do not support Client ID Metadata Documents." Its own example document is a public client, `token_endpoint_auth_method` of `none`, redirecting to `http://127.0.0.1:3000/callback` and `http://localhost:3000/callback`.

Three consequences follow for the family.

* **Registration is optional.** Consumption at registration, the registration-validity model, and renewal through the token endpoint serve servers that keep RFC 7591 registrations. MCP still permits Dynamic Client Registration for compatibility, so an MCP authorization server may keep registrations; the profile assumes one that keeps none, which is the profile's choice rather than MCP's. Under that assumption runtime admission is the whole of the family's relevance, and a statement without `statement_uses` is already usable only at request time.
* **Desktop clients redirect to loopback.** The statement draft now treats such a client's presentation as review-only: it admits nothing an unreviewed client would not get, never satisfies a policy that requires reviewed software, and creates no establishment. That is the honest outcome, since another application on the same device can claim the loopback port, and it means a desktop client presenting a statement is no worse off than one presenting nothing.
* **Nobody carries anything yet.** An MCP client today sends its document URL and, at most, a client assertion. Asking every client vendor to obtain a statement, store it, renew it before expiry, and attach it to pushed authorization requests is the step that will not happen.

The last of these is the one this sketch is about.

## What carrying costs

The client-carried statement drives much of the family's machinery, but not all of it.

| Machinery | Why it exists | Removed by pull |
| --- | --- | --- |
| The `software_statement` request parameter | It is how the statement arrives | Yes |
| Renewal by prior statement, and its holder binding | The client must keep a current copy | For clients; the publisher replaces what it publishes |
| The copied-statement analysis | A copy in the wrong hands is presentable | Narrowed; a published statement is still readable |
| The `iat` floor | Instances and servers may see different versions | No |
| Refresh replacement | A grant outlives the statement that opened it | No |

A server that fetches the review itself needs less of this, but not none. The artifact stays exactly as defined: the same JWT, the same validation, the same digest comparison against the document the server resolves, and the same establishment. Only the conveyance changes, and conveyance accounts for only some of the rows above.

## Pulling the review

There are two places a server could pull from.

### From a location the document names

The document names, in `software_statements_uri`, where its current statements are published. The server already fetches the document to compare its digest, so finding the statements costs one more fetch, cached like the document.

This is what the statement draft's Pulled Statements section now defines, and it avoids the circularity that rules out embedding: the document carries a location, not a statement over itself, and the statement's digest covers that location along with everything else. A different party cannot redirect the server to statements of its choosing without changing the document, since the location is part of the document the digest covers, and changing it changes the digest every statement names.

The publisher controls the location, so it can withhold. Withholding a review downgrades the client to unreviewed, which is the publisher's own loss, unless the server's trust configuration requires an issuer for that client, in which case withholding is a refusal. It cannot forge one, since every statement is issuer-signed, and an older statement it serves is still subject to `exp`, status, and the `iat` floor.

It also gives a working form to the suggestion in Section 4.3 of the CIMD draft, carrying a `software_statement` inside the document. The statement draft has consumers ignore an embedded statement of its profile, because a document cannot carry a review of itself; a reference to where reviews live can.

### From the issuer

The server asks a configured issuer, by client identifier, for its current decision. This is a proposed extension that no draft defines; it is the pull [the vendor integration sketch](saas-integration-sketch.md) explores with Federation's trust mark endpoint. The publisher cannot withhold, but the issuer sits on the request path, learns which servers ask about which software, and needs an authorization model for who may ask about whom, which does not exist yet.

### Which

Document-located pull is the smaller change, the one the statement draft specifies, and fits MCP, where the client publisher is the party with an incentive to be reviewed. Issuer-located pull fits enterprise deployments, where the customer's own issuer should decide regardless of what the publisher publishes. They compose: a server can configure which issuers it pulls from directly and accept document-located statements from the rest.

## What pull keeps, and what it removes

The presenter binding does not change. A confidential client proves a key its document carries, through `private_key_jwt` or DPoP, as runtime presentation already requires. A public client with `https` redirection URIs is bound by them. A loopback client is review-only. None of that depended on who delivered the statement.

What goes away is client-side carriage and distribution: the request parameter, the client keeping a current copy, and the client's part in renewal, which becomes the publisher replacing what it publishes. What remains is everything that exists because statements change over time. A grant still outlives the statement that opened it, so refresh replacement stays. A host can serve a stale copy or publish in the middle of a change, and two servers fetching at different moments can see different versions, so the `iat` floor stays. The copied-statement analysis narrows, since nothing travels with a client, but a published statement is readable by anyone and so still presentable where presentation is accepted.

## The MCP profile

[Reviewed Client Software](../mcp/reviewed-client-software.mdx) is the profile this reasoning led to, a proposed extension to MCP. Its requirements beyond base MCP are its own: a Client ID Metadata Document as `client_id`, no registration step, statements pulled from `software_statements_uri`, and loopback clients treated as review-only.

## Composing with Enterprise-Managed Authorization

MCP's Enterprise-Managed Authorization extension is a profile of the Identity Assertion JWT Authorization Grant. The client exchanges the user's ID token at the enterprise identity provider, which "evaluates administrator-defined policies" and returns an ID-JAG; the client then presents the ID-JAG to the MCP authorization server with the `jwt-bearer` grant, using its Client ID Metadata Document URL as `client_id` where it is not pre-registered.

That is the request runtime presentation already covers, since the statement draft accepts the RFC 7521 assertion grants. On one `jwt-bearer` request the two artifacts answer different questions, which is the split the statement draft's relationship to identity assertions describes:

| Question | Answered by |
| --- | --- |
| May this user use this client with this MCP server, now? | The ID-JAG: the enterprise identity provider's policy, evaluated per grant |
| Who reviewed this software, and against which bytes? | The statement, pulled or presented, verified against the document's digest |

Neither carries what the other does. Where the identity provider mediates every grant, a customer that only wants to allow or block agent software can do it there, and a statement adds nothing it needs. The statement earns its place where the reviewer is not the identity provider: a catalog vouching to many MCP servers, a server vendor's marketplace, or a client acting with no user at all, where there is no ID-JAG to carry the customer's decision.

## Open questions

* **The member name, and where it is defined.** The statement draft now defines `software_statements_uri` and how a server pulls from it, and the MCP profile requires it. Whether CIMD should define the member instead is the natural thing to raise with the CIMD editors alongside the embedded-statement question.
* **Byte-exact digests and generated documents.** Some MCP client documents are generated by build tooling, where key order and whitespace drift without any metadata change. Pull makes re-review cheaper but does not remove it. Whether MCP needs the canonicalized digest the drafts defer is a question for the profile.
* **When to propose the profile.** It cites the statement draft, so it waits for that draft's first submission in order to cite a pinned revision.
* **Issuer-located pull's authorization model.** Who may ask an issuer about which software is the same unsolved question the vendor integration sketch reaches, and it needs an answer before issuer-located pull is more than a mechanism.
