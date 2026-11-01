---
title: "Binding Agent Dialogs to WIMSE and OAuth Credentials"
abbrev: "Dialog Binding for WIMSE and OAuth"
docname: draft-reddy-wimse-oauth-dialog-binding-latest
category: std
ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Workload Identity in Multi System Environments"
submissiontype: IETF
keyword:
  - agent
  - dialog
  - WIMSE
  - transaction token
stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: T. Reddy
    fullname: Tirumaleswar Reddy
    organization: Nokia
    email: k.tirumaleswar_reddy@nokia.com
  -
    ins: Z. Sarker
    name: Zaheduzzaman Sarker
    organization: Nokia
    country: Sweden
    email: zaheduzzaman.sarker@nokia.com

normative:
  RFC2119:
  RFC8174:
  RFC7519:
  RFC8693:
  RFC9421:
  I-D.ietf-wimse-workload-creds:
  I-D.ietf-wimse-identifier:
  I-D.ietf-wimse-http-signature:
  I-D.ietf-wimse-wpt:
  I-D.ietf-oauth-transaction-tokens:
  I-D.ietf-oauth-identity-chaining:

informative:
  A2A:
    title: "Agent2Agent (A2A) Protocol Specification"
    target: https://a2a-protocol.org/latest/specification/
  I-D.agentic-ai-usecases-requirements:
  RFC7523:
  I-D.ietf-wimse-arch:
  I-D.ietf-wimse-aims:
  I-D.reddy-wimse-aggregate-signatures:
  I-D.manyfolks-agentproto-arch:

--- abstract

AI agents and tools that take part in a dialog carry a dialog context
with each request and response. This document specifies how the
dialog context is bound to WIMSE and OAuth credentials. It extends the
WIMSE profile of HTTP Message Signatures, for requests and responses,
and the WIMSE Proof Token, for requests, to cover the dialog context,
records the dialog identifier in OAuth transaction tokens, and
describes how a new dialog at a trust boundary aligns with OAuth
identity chaining. A receiver can then verify who sent a request or
response, which dialog it belongs to, and, for a request, what it may
do, and that the answers agree.

--- middle

# Introduction {#intro}

The use cases and requirements for agent dialogs
{{I-D.agentic-ai-usecases-requirements}} describe how users, agents,
and tools interact over the course of a task, in a dialog. Among
them, a receiver needs to verify who sent a request or response, who
initiated the dialog, and that no hop altered the dialog context it
carries. Authorization granted in one dialog also needs to be limited
to that dialog.

These requirements lead to an agent communication architecture
{{I-D.manyfolks-agentproto-arch}}, an agent communication protocol,
and security mechanisms such as those in this document. This document
meets them with existing WIMSE and OAuth credentials: the WIMSE HTTP
signature or WPT covers the dialog context, and the transaction token
records the dialog identifier. A participant can initiate a linked
dialog within a trust domain. At a trust boundary, it always initiates
one, and authorization moves to the other domain through identity
chaining.

An agent communication protocol carries a dialog context with each
request and response, which identifies the dialog and the participant
that initiated it. This document uses `Dialog-Context` as a
placeholder name for the HTTP field that carries the dialog context.
Its name and format will be aligned with the agent communication
protocol once that protocol is defined. The goal of this document is
to show how the dialog context is bound to WIMSE and OAuth
credentials, not to define the dialog context itself.

The agent communication protocol defines no credentials, signatures,
or authorization model of its own. It relies on WIMSE for workload
identity and message integrity {{I-D.ietf-wimse-arch}}, and on OAuth
for authorization. The WIMSE framework for AI agents
{{I-D.ietf-wimse-aims}} composes these for agent authentication and
authorization. Today none of these mechanisms covers the dialog
context:

* A WIMSE HTTP signature {{I-D.ietf-wimse-http-signature}} or WIMSE
  Proof Token (WPT) {{I-D.ietf-wimse-wpt}} does not cover the
  `Dialog-Context` field. A hop can change the dialog context without
  breaking the proof.

* A transaction token {{I-D.ietf-oauth-transaction-tokens}} does not
  record the dialog. A token obtained in one dialog can be presented
  in another.

TLS protects each hop between adjacent endpoints
{{I-D.ietf-wimse-arch}}. Where two participants connect directly with
mutual TLS, TLS already authenticates the peer and protects the
messages on that hop. An application-layer proof, made with the
sender's WIT key over the message, adds two properties:

Protection across intermediaries:
: The receiver verifies the proof itself, even when an intermediary on
  the path, such as a gateway, terminates TLS. It learns which
  workload sent the message, not only which intermediary it connected
  to.

Provenance:
: Each hop that forwards a request or response adds its proof,
  starting with the initiator, and the proofs travel together with the
  message. A receiver several hops away can then verify which
  participant initiated the dialog, and that no hop altered the dialog
  context.

Provenance matters for agents because a request or response can pass
through several agents before it reaches its destination. For example,
A1 initiates a dialog with A2, and A2 sends a new request to a tool.
TLS and A2's proof show the tool only that A2 sent the request, not
that A1 initiated the dialog. The proofs of earlier hops show that
({{multi-hop}}).

This document closes both gaps. A receiver can then answer three
questions about a request from three independent sources, and check
that the answers agree:

* Who sent it: the WIMSE proof.
* What it may do: the transaction token.
* Which dialog it belongs to: the dialog context, covered by the WIMSE
  proof and recorded in the transaction token.

Transaction tokens are optional. Without them, the WIMSE proof still
binds the dialog context to the request and its sender.

# Conventions and Terminology {#terms}

{::boilerplate bcp14-tagged}

This document uses the following terms:

* Workload Identity Token (WIT), defined in
  {{I-D.ietf-wimse-workload-creds}}.
* WIMSE Proof Token (WPT), defined in {{I-D.ietf-wimse-wpt}}.
* Transaction Token and Transaction Token Service (TTS), defined in
  {{I-D.ietf-oauth-transaction-tokens}}.
* Dialog and Dialog Context, defined in
  {{I-D.agentic-ai-usecases-requirements}}.
* Dialog Identifier, defined in {{I-D.manyfolks-agentproto-arch}} as
  the Agentic Dialog Identifier (ADID).

It also defines the following terms:

Participant:
: An agent or tool that takes part in a dialog. Each participant is a
  workload with its own WIT.

Initiator:
: The participant that proposes the dialog context in the request that
  establishes the dialog. It is the principal of the dialog
  {{I-D.manyfolks-agentproto-arch}}. The dialog context identifies it
  by its WIMSE workload identifier.

Linked Dialog:
: A new dialog, with a new dialog context, that a participant initiates
  to continue a dialog it received.

This document applies to participants that are WIMSE workloads. How a
client without a WIT, such as a user application, is identified in the
dialog context is left to the agent communication protocol.

# Security Architecture {#architecture}

~~~ aasvg
+------------+   +-----------------+   +----------------------+
| WIT issuer |   | Transaction     |   | Authorization server |
|            |   | Token Service   |   |                      |
+------------+   +-----------------+   +----------------------+
      |                  |                        |
      | WIT              | transaction token      | grant for
      |                  | (tctx.dlg = D1)        | another domain
      v                  v                        v
+--------------------------------------------------------------+
|                     Participant A2 (sender)                  |
+--------------------------------------------------------------+
      |
      | Request, over TLS
      v
+--------------------------------------------------------------+
| Dialog-Context:  dlg = D1 <--------------------------(1)--+  |
|                  initiator = wimse://a.example/agent/a1   |  |
|                   ^                                       |  |
|                  (2) equal                                |  |
|                   v                                       |  |
| Txn-Token:  tctx.dlg = D1 <--------------------------(3)--+  |
| Workload-Identity-Token: sub = wimse://a.example/agent/a2 |  |
| Signature or WPT, made with A2's WIT key -----------------+  |
+--------------------------------------------------------------+
      |
      v
+--------------------------------------------------------------+
|                   Next participant or tool                   |
+--------------------------------------------------------------+
~~~
{: #fig-arch title="Security architecture"}

The figure shows three bindings:

1. The signature or WPT covers the dialog context.

2. The transaction token records the same dialog identifier as the
   dialog context.

3. The signature or WPT covers the transaction token.

The agent communication protocol defines no credentials or signatures
of its own. It reuses WIMSE for workload identity and message
integrity, and OAuth for authorization. Each answers a different
question about a request: the WIMSE HTTP signature or WPT says who
sent it, the transaction token says what it may do, and the dialog
context says which dialog it belongs to.

Each participant holds a WIT issued by its domain's WIT issuer. Each
hop proves possession of its WIT key for the message it sends, using
either the WIMSE HTTP signature {{I-D.ietf-wimse-http-signature}}, for
requests and responses, or the WPT {{I-D.ietf-wimse-wpt}}, for
requests only. Because the signature or WPT covers the dialog context,
a receiver can verify that the dialog context it received is the one
the sender signed, and that it was not changed in transit. The dialog
context therefore needs no signature of its own.

Both proofs protect a single hop. For a receiver to verify which
participant initiated the dialog, the proofs of earlier hops need to
travel with the message ({{multi-hop}}).

Where a deployment uses transaction tokens, the Transaction Token
Service records the dialog identifier in the token. Participants
forward the token within the trust domain, and a receiver rejects a
token issued for another dialog.

At a trust boundary, a participant initiates a linked dialog, and
authorization moves to the other domain through identity chaining
{{I-D.ietf-oauth-identity-chaining}}. Neither the dialog identifier
nor the transaction token of the original domain crosses the boundary.

# Assumed Dialog Context Content {#dc-content}

This document relies on the following information in the dialog
context. The agent communication protocol defines the encoding and
any further content.

Dialog identifier (`dlg`):
: Identifies the dialog. The initiator proposes it, and the server
  that creates the dialog accepts it. The transaction token records it
  in `tctx.dlg` ({{tctx-dlg}}).

Initiator:
: The WIMSE workload identifier {{I-D.ietf-wimse-identifier}} of the
  participant that initiated the dialog. A receiver checks that the
  first proof in the chain was made by this workload ({{verify}}).

The agent communication protocol will add more meta-data to the dialog
context. The WIMSE proof covers the whole `Dialog-Context` field, so it
also covers any meta-data added in the base spec and future
extensions.

The dialog context does not carry credentials or tokens, which travel
in their own fields, nor the context an agent builds over the dialog.
Each participant knows only the hops it exchanges messages with; the
dialog context does not list participants.

# Dialog Establishment {#establish}

The initiator proposes a dialog context in its first request: a
`Dialog-Context` field with a new dialog identifier and the
initiator's WIMSE workload identifier, covered by the initiator's
WIMSE proof. The server checks that the initiator named in the field
equals the `sub` claim of the WIT used to verify the proof, and that
the dialog identifier is not already in use. It then creates the
dialog, or rejects the request.

When it creates the dialog, the server returns the accepted dialog
context in its response, and the initiator checks that it matches the
proposed one. With the WIMSE HTTP signature, the server signs the
response, and the signature covers `Dialog-Context` ({{http-sig}}).
The WPT defines no response proof, so with the WPT the response is not
authenticated. The request can carry the first input of the dialog, so
establishing a dialog adds no round trip.

The initiator proposes the dialog context so that its own proof covers
it from the first request, and a receiver several hops away can verify
that the initiator started the dialog ({{multi-hop}}). The server
still decides whether to create the dialog, and maintains its state.

A participant that maintains no dialog state, such as a stateless tool,
creates no dialog. It receives the dialog context of the caller with
each request.

# Overview {#overview}

{{fig-agents}} shows the agents in the example. All three are in
domain A. Agent A1 initiates dialog D1 with agent A2 ({{establish}})
and delegates a task to A2, which calls a tool in D1.

~~~ aasvg
                         Domain A
+--------------------------------------------------------+
|                                                        |
|  +--------+   request, D1   +--------+   request, D1   |
|  |   A1   |---------------->|   A2   |-------------+   |
|  +--------+                 +--------+             |   |
|  (initiator)                                       v   |
|                                               +------+ |
|                                               | Tool | |
|                                               +------+ |
+--------------------------------------------------------+
~~~
{: #fig-agents title="Agents in the example"}

{{fig-overview}} shows the request from A2 to the tool.

~~~
POST /invoke HTTP/1.1
Dialog-Context: dlg=D1; initiator=wimse://a.example/agent/a1
Txn-Token: <transaction token with tctx.dlg = D1>
Content-Digest: sha-256=:<hash of the body>:
Workload-Identity-Token: <WIT with sub = wimse://a.example/agent/a2>
Signature-Input: wimse=("@method" "@path" "dialog-context"
    "txn-token" "content-digest" "workload-identity-token");...
Signature: wimse=:<signature made with A2's WIT key>:
~~~
{: #fig-overview title="A request bound to its dialog"}

Values are shown informally. The `Dialog-Context` syntax is a
placeholder ({{intro}}). The `Signature-Input` field lists what the
signature covers: the method, the path, the dialog context, the
transaction token, the body digest, and A2's WIT.

The tool verifies the WIMSE proof with A2's WIT. The proof covers the
`Dialog-Context` field, so the dialog context has not changed on this
hop. The tool validates the transaction token and checks that its
dialog identifier equals the one in `Dialog-Context`. A token issued
for another dialog fails this check.

The proof of a single hop shows who sent the request. It does not show
who initiated the dialog when the sender is not the initiator.
Verifying the initiator at a later hop requires the proofs of earlier
hops, as described in {{multi-hop}}.

# Dialog Context in WIMSE HTTP Signatures {#http-sig}

When a request carries the `Dialog-Context` field, the WIMSE HTTP
signature of the request {{I-D.ietf-wimse-http-signature}}, a profile
of HTTP Message Signatures {{RFC9421}}, MUST include
`dialog-context` in its covered components. This adds `Dialog-Context`
to the fields that the profile requires to be covered when present.

When a response carries the `Dialog-Context` field and is signed, the
signature of the response MUST include `dialog-context` in its covered
components. The response signature also covers the method, path, and
nonce of the request it answers {{I-D.ietf-wimse-http-signature}}, so
it cannot be moved to another request. The requester MUST verify that
the `Dialog-Context` value in the response matches the one in its
request.

A receiver MUST reject a request that carries `Dialog-Context` and a
WIMSE HTTP signature that does not cover it.

A participant that forwards a request in the same dialog MUST NOT
change the `Dialog-Context` field, so that the proofs of earlier hops
verify against it ({{multi-hop}}).

A repeated or malformed `Dialog-Context` field is handled as specified
for covered fields in {{I-D.ietf-wimse-http-signature}} and
{{I-D.ietf-wimse-wpt}}.

# Dialog Context in the WIMSE Proof Token {#wpt}

This document defines the following claim for the WPT:

dth:
: The hash of the `Dialog-Context` field value, computed as specified
  for the `oth` claim {{I-D.ietf-wimse-wpt}}. It is REQUIRED when the
  request carries the `Dialog-Context` field.

The WPT is signed with the sender's WIT key, and the signature covers
the `dth` claim. The dialog context is therefore part of what the
sender signs. When a request carries both a WPT and the
`Dialog-Context` field, the receiver verifies the WPT signature
{{I-D.ietf-wimse-wpt}} and checks that `dth` matches the hash of the
`Dialog-Context` field in the request. If the signature does not
verify, `dth` is missing, or the hash does not match, the receiver
MUST reject the request.

# Multi-hop Provenance {#multi-hop}

A WIMSE HTTP signature and a WPT each protect a single hop. A receiver
verifies the hop that sent the message. To verify which participant
initiated the dialog, and that no earlier hop altered the dialog
context, the receiver needs the proof of the initiator and of each
participant that forwarded the message.
{{I-D.reddy-wimse-aggregate-signatures}} specifies how the WIMSE HTTP
signatures of earlier hops travel with a message, and how they can be
aggregated into one signature as an optimization. Each of those
signatures covers the same `Dialog-Context` value, so a change by any
hop breaks the chain.

Without the proofs of earlier hops, a receiver verifies only the
participant that sent the message. If that participant is not the
initiator, the receiver cannot tell whether the dialog context it
received is the one the initiator used. For example, the tool in
{{fig-agents}} can verify that A2 sent D1, but not that A1 initiated
D1. A transaction token, if used, does not help here: it shows that
the request is authorized for dialog D1, not who initiated D1.

# Dialog Identifier in Transaction Tokens {#txn-token}

This section applies to deployments that use transaction tokens.

## The dlg Member of tctx {#tctx-dlg}

This document defines the following member of the `tctx` claim of a
transaction token:

dlg:
: The dialog identifier of the dialog in which the transaction token
  is used, as carried in the `dlg` field of the dialog context. String.

The dialog identifier is part of the authorization context, and stays
unchanged for the life of the transaction. The transaction token
records only the dialog identifier, because it stays the same for the
life of the dialog. The rest of the dialog context, such as its state,
can change, and the WIMSE proof protects it instead.

~~~ json
{
  "txn": "97053963-771d-49cc-a4e3-20aad399c312",
  "sub": "user@a.example",
  "aud": "a.example",
  "scope": "trip.book",
  "tctx": {
    "dlg": "7f3c9a2e-5b1d-4e8a-9c6f-0d2b8e4a1f37"
  },
  "req_wl": "wimse://a.example/agent/a1"
}
~~~
{: #fig-txn title="Transaction token recording the dialog"}

## Requesting the Token {#txn-request}

To get a transaction token for a dialog, a participant includes the
dialog identifier as a `dlg` member of the `request_details` parameter
of its token exchange request {{RFC8693}}. The TTS checks the request
against its local policy. If the check succeeds, the TTS issues a
transaction token with the dialog identifier in `tctx.dlg`.

## Forwarding {#txn-forward}

Participants forward the transaction token down the call chain together
with the dialog context. They do not forward access tokens
{{I-D.ietf-wimse-aims}}. A transaction token does not leave its trust
domain.

A transaction token is short-lived, which limits how long a stale
authorization can be used. Revocation within that lifetime is
signaled through Shared Signals ({{sec-revocation}}).

## Receiver Check {#txn-check}

A receiver of a request that carries both a transaction token and a
`Dialog-Context` field MUST check that `tctx.dlg` equals the `dlg`
field of the dialog context, and MUST reject the request if it does
not. A receiver MUST reject a request that carries `Dialog-Context` and
a transaction token without `tctx.dlg`, or a transaction token with
`tctx.dlg` and no `Dialog-Context` field.

# Trust Boundaries and Identity Chaining {#boundary}

A participant can initiate a linked dialog within a trust domain,
where local policy calls for it, and always initiates one at a trust
boundary, since a dialog does not cross a trust boundary. At the
boundary, authorization moves to the other domain through identity
chaining {{I-D.ietf-oauth-identity-chaining}}, so the dialog and the
authorization reset at the same point.

In {{fig-boundary}}, A1 initiates dialog D1 with A2 and sends a
request in D1 (step 0, {{establish}}). A2 holds a domain A transaction
token that records D1.

~~~ aasvg
            Domain A                :             Domain B
                                    :
 A1      A2                  AS A   :  AS B          B1           TTS B
  |       |                    |    :    |            |             |
  | (0)   |                    |    :    |            |             |
  | D1    |                    |    :    |            |             |
  |------>|                    |    :    |            |             |
  |       |                    |    :    |            |             |
  |       | (1) txn token (D1) |    :    |            |             |
  |       |------------------->|    :    |            |             |
  |       | JWT grant          |    :    |            |             |
  |       |<-------------------|    :    |            |             |
  |       |                    |    :    |            |             |
  |       | (2) JWT grant      |    :    |            |             |
  |       |----------------------------->|            |             |
  |       | access token       |    :    |            |             |
  |       |<-----------------------------|            |             |
  |       |                    |    :    |            |             |
  |       | (3) request, access token,   |            |             |
  |       |     Dialog-Context: D2  :    |            |             |
  |       |------------------------------------------>|             |
  |       |                    |    :    |       [B1 creates D2]    |
  |       |                    |    :    |            |             |
  |       |                    |    :    |            | (4) access  |
  |       |                    |    :    |            | token (D2)  |
  |       |                    |    :    |            |------------>|
  |       |                    |    :    |            | txn token,  |
  |       |                    |    :    |            | tctx.dlg=D2 |
  |       |                    |    :    |            |<------------|
~~~
{: #fig-boundary title="Linked dialog D2 at the domain boundary"}

1. A2 needs to call B1 in domain B. A dialog does not cross a trust
   boundary, so A2 initiates a linked dialog D2 with B1. A2 sends its
   transaction token, which records D1, to the authorization server of
   domain A (AS A) and asks for a JWT authorization grant for domain
   B. If the transaction token is valid, AS A issues the grant.

2. A2 sends the grant to the authorization server of domain B (AS B)
   {{RFC7523}}. AS B returns an access token for domain B.

3. A2 sends a signed request to B1 carrying the access token, its
   task, and a `Dialog-Context` field that proposes D2, with A2 as
   initiator. B1 verifies A2's WIT and proof; this requires B1 to trust
   the WIT issuer of domain A. B1 then checks the proposal and creates
   D2 ({{establish}}).

4. B1 sends the access token to the Transaction Token Service of
   domain B (TTS B), with D2 in `request_details`. TTS B returns a
   transaction token for domain B that records D2 in `tctx.dlg`. B1
   uses this token for its own calls in domain B.

Neither D1 nor the transaction token of domain A reaches domain B. The
JWT authorization grant is exchanged for an access token of domain B,
and A2 establishes D2 in domain B with itself as initiator. Only A2
knows that D2 continues D1.

# Receiver Verification {#verify}

A receiver of a request that carries `Dialog-Context`:

1. Verifies the WIT and WIMSE proof of the sending hop, and that the
   proof covers `Dialog-Context` ({{http-sig}}, {{wpt}}).

2. If proofs of earlier hops are present, verifies each of them, and
   that the `sub` of the WIT at the origin of the chain equals the
   initiator named in the dialog context ({{multi-hop}}).

3. If a transaction token is present, validates it and checks
   `tctx.dlg` ({{txn-check}}).

If a check fails or cannot be completed, the receiver rejects the
request. Its error says which of the two happened. The error format is
defined by the agent communication protocol.

# Security Properties {#properties}

The binding in this document gives the dialog context the following
properties:

Origin:
: A receiver can verify which participant initiated the dialog
  ({{multi-hop}}).

Integrity:
: A hop cannot change the dialog context without breaking a WIMSE
  proof.

Sender binding:
: A receiver learns which workload presented the transaction token,
  and a party without a WIT cannot provide a valid WIMSE proof for it.

Dialog-bound authorization:
: A transaction token issued for one dialog is rejected in another.

Response binding:
: A signed response cannot be moved to another request, and it
  carries the dialog context of its request.

Domain separation:
: Neither the dialog identifier nor the transaction token of one
  domain crosses a trust boundary.

# Security Considerations {#security}

## Dialog Binding {#sec-binding}

The WIMSE proof covers `Dialog-Context`, so a hop cannot change the
dialog context without detection. The transaction token records the
dialog identifier, so a token issued for one dialog cannot be used in
another. Both are needed to bind a request, its authorization, and its
dialog. This document does not define additional replay protection.
Deployments rely on the replay protection of the WIMSE proof mechanism
they use.

The dialog identifier is not a secret; it grants nothing on its own.

## Responses {#sec-response}

An attacker that alters an unsigned response can change its result or
move it to another dialog. A response signed with the WIMSE HTTP
signature and covering `Dialog-Context` binds the result to the dialog
and to the request it answers. A requester can ask for a signed
response {{I-D.ietf-wimse-http-signature}}.

## Earlier Hops {#sec-hops}

Verifying only the sending hop does not detect an earlier hop that
changed the dialog context. Deployments that need this assurance use
the proofs of earlier hops ({{multi-hop}}).

## Revocation {#sec-revocation}

When the authorization behind a dialog is revoked, for example through
Shared Signals {{I-D.ietf-wimse-aims}}, participants stop relying on
transaction tokens and cached authorization decisions for that dialog.

# Privacy Considerations {#privacy}

The dialog identifier in `tctx.dlg` is visible to every participant in
the trust domain that receives the transaction token, and links the
transactions of a dialog. This is the purpose of the dialog identifier
within a domain. It does not cross a trust boundary ({{boundary}}).

When the proofs of earlier hops travel with a message ({{multi-hop}}),
each proof carries the WIT of its hop. A receiver then learns the
workload identity of every participant that forwarded the message, not
only the hop it received the message from.

# IANA Considerations {#iana}

## JSON Web Token Claims

IANA is requested to register the following claim in the "JSON Web
Token Claims" registry established by {{RFC7519}}:

* Claim Name: dth
* Claim Description: Hash of the Dialog-Context HTTP field
* Change Controller: IETF
* Specification Document(s): {{wpt}} of this document

## Transaction Token Context

The `dlg` member of `tctx` is defined in {{tctx-dlg}}. No registry for
`tctx` members exists at the time of writing.

--- back

# Comparison with the A2A Context {#a2a}

This appendix compares the context of A2A {{A2A}} with the dialog
context.

| | A2A contextId | Dialog context |
|---|---|---|
| What it is | An opaque string grouping related tasks and messages | An identifier and the initiator |
| Who creates it | The server; a client-supplied value is discouraged | The initiator proposes it; the server checks the proposal and creates the dialog |
| Scope | One client and one server, a single hop | Spans hops, or continues through a linked dialog |
| Delegation | A new, unrelated context with the next agent | The same dialog, or a linked dialog |
| Across domains | No defined behavior; each agent starts a new context | Continues through a linked dialog, with authorization moved by identity chaining |
| Lifecycle | None at the context level; per task | Dialog-level |
| Integrity | Not covered by a signature | Covered by each hop's WIMSE proof |
| Origin | Does not name the client that started it | Names the initiator, verified against the first proof |
| Authorization | Not tied to tokens | Recorded in the transaction token |
{: title="A2A context and dialog context"}

# Acknowledgments
{: numbered="false"}

TODO.
