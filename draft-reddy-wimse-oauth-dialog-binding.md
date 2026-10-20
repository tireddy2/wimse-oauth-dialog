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
    fullname: Tirumaleswar Reddy
    organization: Nokia
    email: k.tirumaleswar_reddy@nokia.com

normative:
  RFC2119:
  RFC8174:
  RFC7519:
  RFC8693:
  RFC9421:
  I-D.ietf-wimse-workload-creds:
  I-D.ietf-wimse-http-signature:
  I-D.ietf-wimse-wpt:
  I-D.ietf-oauth-transaction-tokens:
  I-D.ietf-oauth-identity-chaining:

informative:
  RFC6750:
  RFC7523:
  I-D.ietf-wimse-arch:
  I-D.ietf-wimse-aims:
  I-D.reddy-wimse-aggregate-signatures:
  I-D.zahed-agentproto-refarch:

--- abstract

AI agents and tools that take part in a dialog carry a dialog context
with each request. This document specifies how the dialog context is
bound to WIMSE and OAuth credentials. It extends the WIMSE profile of
HTTP Message Signatures and the WIMSE Proof Token to cover the dialog
context, records the dialog identifier in OAuth transaction tokens, and
describes how a new dialog at a trust boundary aligns with OAuth
identity chaining. A receiver can then verify who sent a request, what
the request may do, and which dialog it belongs to, and that the three
answers agree.

--- middle

# Introduction {#intro}

The agent communication architecture {{I-D.zahed-agentproto-refarch}}
groups the interactions between users, agents, and tools that serve a
task into a dialog. An agent communication protocol carries a dialog
context with each request, which identifies the dialog and the
participant that created it. This document assumes that the dialog
context is carried in an HTTP field named `Dialog-Context`.

The dialog management protocol defines no credentials, signatures, or
authorization model of its own. It relies on WIMSE for workload
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
sender's WIT key over the request, adds three properties:

Sender binding:
: A transaction token is a bearer token {{RFC6750}}: whoever holds it
  can present it. The proof covers the token, so the receiver learns
  which workload presented it and applies its policy to that workload.
  A party without a WIT cannot present the token.

Protection across intermediaries:
: The receiver verifies the proof itself, even when an intermediary on
  the path, such as a gateway, terminates TLS.

Provenance:
: When the proofs of earlier hops travel with the request, a receiver
  can verify which participant created the dialog.

Provenance matters most for agents. An agent that delegates is an
endpoint, not an intermediary: it receives one request and sends a new
one. Neither TLS nor a single-hop proof shows the next participant who
created the dialog. That requires the proofs of earlier hops
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

It also defines the following terms:

Dialog:
: The correlated sequence of interactions between participants over
  the course of a task.

Dialog Context:
: The identifiers and lifecycle state carried with each request of a
  dialog.

Dialog Identifier:
: The identifier, carried in the dialog context, that names the
  dialog.

Participant:
: An agent or tool that takes part in a dialog. Each participant is a
  workload with its own WIT.

Creator:
: The participant that created a dialog.

Linked Dialog:
: A new dialog, with a new dialog context, that a participant creates
  to continue a dialog it received.

# Security Architecture {#architecture}

~~~ aasvg
+------------+   +-----------------+   +----------------------+
| WIT issuer |   | Transaction     |   | Authorization server |
|            |   | Token Service   |   |                      |
+------------+   +-----------------+   +----------------------+
      |                  |                        |
      | WIT              | transaction token      | grant for
      |                  | (records dialog)       | another domain
      v                  v                        v
+--------------------------------------------------------------+
|                         Participant                          |
+--------------------------------------------------------------+
      |
      | Request: Dialog-Context, HTTP signature or WPT,
      |          transaction token (over TLS)
      v
+--------------------------------------------------------------+
|                   Next participant or tool                   |
+--------------------------------------------------------------+
~~~
{: #fig-arch title="Security architecture"}

The dialog defines no credentials or signatures of its own. It reuses
WIMSE for workload identity and message integrity, and OAuth for
authorization. Each answers a different question about a request: the
WIMSE HTTP signature or WPT says who sent it, the transaction token
says what it may do, and the dialog context says which dialog it
belongs to.

Each participant holds a WIT issued by its domain's WIT issuer. Each
hop proves possession of its WIT key for the message it sends, using
either the WIMSE HTTP signature {{I-D.ietf-wimse-http-signature}},
which can also sign the response, or the WPT {{I-D.ietf-wimse-wpt}}.
The proof covers the dialog context, so the receiver can verify that
the sending hop bound the message to that dialog context. No separate
signature over the dialog context is needed.

Both proofs protect a single hop. For a receiver to verify which
participant created the dialog, the proofs of earlier hops travel with
the request ({{multi-hop}}).

Where a deployment uses transaction tokens, the Transaction Token
Service records the dialog identifier in the token. Participants
forward the token within the trust domain, and a receiver rejects a
token issued for another dialog.

At a trust boundary, a participant creates a linked dialog, and
authorization moves to the other domain through identity chaining
{{I-D.ietf-oauth-identity-chaining}}. Neither the dialog identifier
nor the tokens of the original domain cross the boundary.

# Overview {#overview}

{{fig-agents}} shows the agents in the example. Agent A1 creates
dialog D1 and delegates to agent A2, which calls a tool. All three are
in domain A.

~~~ aasvg
                         Domain A
+--------------------------------------------------------+
|                                                        |
|  +--------+   request, D1   +--------+   request, D1   |
|  |   A1   |---------------->|   A2   |-------------+   |
|  +--------+                 +--------+             |   |
|  (creator)                                         v   |
|                                               +------+ |
|                                               | Tool | |
|                                               +------+ |
+--------------------------------------------------------+
~~~
{: #fig-agents title="Agents in the example"}

{{fig-overview}} shows the request from A2 to the tool.

~~~ aasvg
+----------------------------------------------------------+
| POST /invoke                                             |
| Dialog-Context: D1, version 1, created by A1   (dialog)  |
| Txn-Token: tctx.dlg = D1                       (OAuth)   |
| Content-Digest: sha-256 of body                          |
| Workload-Identity-Token: A2                    (WIMSE)   |
| Signature: A2, covering all of the above       (WIMSE)   |
+----------------------------------------------------------+
~~~
{: #fig-overview title="A request bound to its dialog"}

The tool verifies the WIMSE proof with A2's WIT. The proof covers the
`Dialog-Context` field, so the dialog context has not changed on this
hop. The tool validates the transaction token and checks that its
dialog identifier equals the one in `Dialog-Context`. A token issued
for another dialog fails this check.

The proof of a single hop shows who sent the request. It does not show
who created the dialog when the sender is not the creator. Verifying
the creator at a later hop requires the proofs of earlier hops, as
described in {{multi-hop}}.

# Dialog Context in WIMSE HTTP Signatures {#http-sig}

When a request carries the `Dialog-Context` field, the WIMSE HTTP
signature of the request {{I-D.ietf-wimse-http-signature}}, a profile
of HTTP Message Signatures {{RFC9421}}, MUST include
`dialog-context` in its covered components. This adds `Dialog-Context`
to the fields that the profile requires to be covered when present.

When a response carries the `Dialog-Context` field and is signed, the
signature of the response MUST include `dialog-context` in its covered
components.

A receiver MUST reject a request that carries `Dialog-Context` and a
WIMSE HTTP signature that does not cover it.

A participant that forwards a dialog context sends it byte for byte as
it received it. Its signature then covers the same value as the
signature of the hop before it.

# Dialog Context in the WIMSE Proof Token {#wpt}

This document defines the following claim for the WPT:

dth:
: Hash of the `Dialog-Context` field, computed over the ASCII
  representation of the field value as it appears in the request, in
  the same way as the `tth` claim {{I-D.ietf-wimse-wpt}}. It is
  REQUIRED when the request carries the `Dialog-Context` field.


When a request carries both a WPT and the `Dialog-Context` field, the
receiver computes the hash of the field and compares it with the `dth`
claim. If the claim is missing or the values differ, the receiver MUST
reject the request.

# Multi-hop Provenance {#multi-hop}

A WIMSE HTTP signature and a WPT each protect a single hop. A receiver
verifies the hop that sent the request. To verify which participant
created the dialog, and that no earlier hop altered the dialog context,
the receiver needs the proof of the creator and of each participant
that forwarded the request. {{I-D.reddy-wimse-aggregate-signatures}}
specifies how the WIMSE HTTP signatures of earlier hops travel with a
request, and how they can be aggregated into one signature as an
optimization. Each of those signatures covers the same `Dialog-Context`
value, so a change by any hop breaks the chain.

Without the proofs of earlier hops, a receiver verifies only the
participant that sent the request. If that participant is not the
creator, the receiver cannot tell whether the creator issued the dialog
context it received. For example, the tool in {{fig-agents}} can
verify that A2 sent D1, but not that A1 issued that version of D1. A
transaction token, if used, confirms the dialog identifier but not the
version.

# Dialog Identifier in Transaction Tokens {#txn-token}

This section applies to deployments that use transaction tokens.

## The dlg Member of tctx {#tctx-dlg}

This document defines the following member of the `tctx` claim of a
transaction token:

dlg:
: The dialog identifier of the dialog in which the transaction token
  is used, as carried in the `dlg` field of the dialog context. String.

The dialog identifier is part of the authorization context. The
Transaction Token Service (TTS) sets it, and it stays unchanged for the
life of the transaction. The transaction token records the dialog
identifier and not the dialog context, because the dialog context
changes as the creator issues new versions. The WIMSE proof binds the
request to the version it carries.

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

The first participant in a trust domain that needs a transaction token
for a dialog includes the dialog identifier in the `request_details`
parameter of its token exchange request {{RFC8693}} to the TTS, as a
`dlg` member. The TTS
applies local policy before recording it. For example, the TTS can
require that the requester is the creator named in the dialog context,
or that the requester shows the dialog context it received.

The TTS MUST copy the dialog identifier into `tctx.dlg` unchanged.

## Replacement Tokens {#txn-replace}

When the TTS issues a replacement transaction token, it MUST keep
`tctx.dlg` unchanged. A participant that needs a token for a different
dialog requests a new transaction token.

## Forwarding {#txn-forward}

Participants forward the transaction token down the call chain together
with the dialog context. They do not forward access tokens
{{I-D.ietf-wimse-aims}}. A transaction token does not leave its trust
domain.

## Receiver Check {#txn-check}

A receiver of a request that carries both a transaction token and a
`Dialog-Context` field MUST check that `tctx.dlg` equals the `dlg`
field of the dialog context, and MUST reject the request if it does
not. A receiver MUST reject a request that carries `Dialog-Context` and
a transaction token without `tctx.dlg`, unless local policy allows it.

# Trust Boundaries and Identity Chaining {#boundary}

A participant can create a linked dialog within a trust domain, where
local policy calls for it, and always creates one at a trust boundary,
since a dialog does not cross a trust boundary. At the boundary,
authorization moves to the other domain through identity chaining
{{I-D.ietf-oauth-identity-chaining}}, so the dialog and the
authorization reset at the same point.

In {{fig-boundary}}, A1 creates dialog D1 and sends a request to A2 in
D1 (step 0). A2 holds a domain A transaction token that records D1.

~~~ aasvg
             Domain A            :              Domain B
                                 :
  A1      A2            AS A     :     AS B          B1          TTS B
  |       |               |      :       |            |             |
  | (0)   |               |      :       |            |             |
  | D1    |               |      :       |            |             |
  |------>|               |      :       |            |             |
  |       | (1) txn token |      :       |            |             |
  |       |     (D1),     |      :       |            |             |
  |       |   record D2   |      :       |            |             |
  |       |-------------->|      :       |            |             |
  |       | JWT grant, D2 |      :       |            |             |
  |       |<--------------|      :       |            |             |
  |       |                      :       |            |             |
  |       | (2) JWT grant        :       |            |             |
  |       |----------------------------->|            |             |
  |       |       access token, D2       |            |             |
  |       |<-----------------------------|            |             |
  |       |                      :       |            |             |
  |       | (3) request, access token,                |             |
  |       |     Dialog-Context: D2                    |             |
  |       |------------------------------------------>|             |
  |       |                      :       |            |             |
  |       |                      :       |            | (4) access  |
  |       |                      :       |            |  token, D2  |
  |       |                      :       |            |------------>|
  |       |                      :       |            | txn token,  |
  |       |                      :       |            | tctx.dlg=D2 |
  |       |                      :       |            |<------------|
~~~
{: #fig-boundary title="Linked dialog D2 at the boundary between domains A and B"}

1. A2 needs agent B1 in domain B, so it creates a linked dialog D2. A2
   is the creator of D2. A2 sends its transaction token to the
   authorization server of domain A (AS A) and asks for a JWT
   authorization grant for domain B that records D2. AS A returns the
   grant. The grant records D2, not D1.

2. A2 sends the grant to the authorization server of domain B (AS B)
   {{RFC7523}}. AS B returns an access token for domain B. AS B can
   copy D2 from the grant into the access token.

3. A2 sends its request to B1. The request carries the access token
   and `Dialog-Context` D2, and A2 signs it.

4. B1 sends the access token to the Transaction Token Service of
   domain B (TTS B), with D2 in `request_details`. TTS B returns a
   transaction token for domain B that records D2 in `tctx.dlg`. B1
   uses this token for its own calls in domain B.

Neither D1 nor any token of domain A reaches domain B. Only A2 knows
that D2 continues D1.

# Receiver Verification {#verify}

A receiver of a request that carries `Dialog-Context` performs the
following checks, in addition to those of the dialog management
protocol:

1. Verify the WIT of the sending hop and its WIMSE proof, and that the
   proof covers `Dialog-Context` ({{http-sig}}, {{wpt}}).

2. If the proofs of earlier hops are present, verify each of them, and
   that the proof at the origin of the chain was made by the creator
   named in the dialog context ({{multi-hop}}).

3. If the request carries a transaction token, validate it and check
   that `tctx.dlg` equals the `dlg` field ({{txn-check}}).

If a check fails or cannot be completed, the receiver rejects the
request and signals the reason. The receiver distinguishes a check
that failed, such as a proof that does not verify, from one that could
not be completed, such as when the key needed to validate a token
cannot be obtained. The reason is precise enough for the sender to
act on, but does not reveal information an attacker could use. The
error format is defined by the protocol that carries the dialog
context.

# Security Considerations {#security}

Covering `Dialog-Context` in the WIMSE proof prevents a hop from
changing the dialog context without detection. Recording the dialog
identifier in the transaction token prevents a token issued for one
dialog from being used in another. Neither alone binds the request,
its authorization, and its dialog together.

The dialog identifier is not a credential. Knowing it grants nothing.
A request is accepted only with a valid WIMSE proof, and a valid
transaction token where one is used.

Responses need the same protection as requests. A response carries the
result of a task in a dialog. An intermediary or attacker that alters
an unsigned response can change the result, or move a result from one
dialog to another, and the requester acts on it. A response signed
with the responder's WIT key and covering `Dialog-Context` binds the
result to the dialog and to the request it answers. The WPT defines
no proof for responses, so deployments that need this use the WIMSE
HTTP signature, and a requester can ask for the response to be signed
{{I-D.ietf-wimse-http-signature}}.

The `dth` claim and the WIMSE HTTP signature bind the dialog context to
a single request. Replay protection is that of the WPT and of the WIMSE
HTTP signature.

A receiver that verifies only the sending hop cannot detect an earlier
hop that replaced the dialog context with another version of the same
dialog. Deployments that need this assurance use the proofs of earlier
hops ({{multi-hop}}).

The TTS records the dialog identifier it receives. A TTS that records
it without checking the requester's relation to the dialog lets a
participant obtain a token for a dialog it does not belong to. The TTS
policy in {{txn-request}} addresses this.

The dialog identifier gives operators the correlation identifier that
audit records need to reconstruct the execution chain of an agent task
across participants {{I-D.ietf-wimse-aims}}. Logs record token hashes,
such as the `dth` value, rather than tokens. A participant that creates
a linked dialog records the link, since no other participant knows it.

When the authorization behind a dialog is revoked, for example through
Shared Signals as described in {{I-D.ietf-wimse-aims}}, participants
stop relying on transaction tokens and cached authorization decisions
for that dialog.

The agent runtime, not the AI model, holds the WIT key and the tokens
and produces the proofs. The model does not have access to them
{{I-D.ietf-wimse-aims}}.

# Privacy Considerations {#privacy}

The dialog identifier in `tctx.dlg` is visible to every participant in
the trust domain that receives the transaction token, and links the
transactions of a dialog. This is the purpose of the dialog identifier
within a domain. It does not cross a trust boundary ({{boundary}}).

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

# Acknowledgments
{: numbered="false"}

TODO.
