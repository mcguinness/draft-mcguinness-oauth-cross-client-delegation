---
title: "Cross-Client Delegation Profile for OAuth 2.0 Token Exchange"
abbrev: "OAuth Cross-Client Delegation"
category: std

docname: draft-mcguinness-oauth-cross-client-delegation-latest
submissiontype: IETF
number:
date:
v: 3
ipr: "trust200902"
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - token exchange
 - delegation
 - cross-client
 - actor
 - gateway
 - agent
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-cross-client-delegation"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-cross-client-delegation/draft-mcguinness-oauth-cross-client-delegation.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC7519:
  RFC7523:
  RFC7591:
  RFC7662:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC9396:
  RFC9700:
  SAML2.Core:
    title: "Assertions and Protocols for the OASIS Security Assertion Markup Language (SAML) V2.0"
    author:
      org: OASIS
    date: 2005-03
    target: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
  I-D.ietf-oauth-identity-assertion-authz-grant:
  OpenID.Core:
    title: "OpenID Connect Core 1.0"
    author:
      org: OpenID Foundation
    date: 2014-11-08
    target: https://openid.net/specs/openid-connect-core-1_0.html

informative:
  RFC6755:
  RFC7636:
  RFC7800:
  RFC8705:
  RFC8725:
  RFC9068:
  RFC9449:
  I-D.ietf-oauth-attestation-based-client-auth:
  I-D.ietf-oauth-rfc7523bis:
  I-D.mcguinness-oauth-actor-profile:
  OpenID.KeyBinding:
    title: "OpenID Connect Key Binding 1.0 - draft 02"
    author:
     -
        fullname: Dick Hardt
        organization: Hellō
     -
        fullname: Ethan Heilman
        organization: Cloudflare
    date: 2026-06-24
    target: https://openid.net/specs/openid-connect-key-binding-1_0.html
  Google.CrossClient:
    title: "Cross-client Identity"
    author:
      org: Google
    target: https://developers.google.com/identity/protocols/oauth2/cross-client-identity
...

--- abstract

This document defines the Cross-Client Delegation profile for OAuth 2.0 Token Exchange (RFC 8693).  The profile permits a confidential OAuth client (the Delegate) to obtain a token on behalf of an End-User whose Identity Assertion was issued to a different OAuth client (the Initiator), when the authorization server administers an explicit Cross-Client Delegation Relationship (CCDR) between the two client registrations.  The Delegate presents the Initiator's Identity Assertion as the subject token, establishes its own identity as the acting party with an actor token, and authenticates as itself.  The authorization server validates the administered relationship, enforces any `may_act` constraint carried by the assertion, evaluates policy at exchange time, and issues a token whose `act` claim identifies the Delegate as the current actor.


--- middle

# Introduction

Modern OAuth deployments increasingly encounter topologies where the OAuth client that authenticates the End-User is not the OAuth client that will present the resulting token to a downstream service.  Enterprise agent-plus-gateway topologies, mobile-app-plus-backend-for-frontend patterns, and Model Context Protocol (MCP) gateway deployments all share this shape: an initiating client (the Initiator) authenticates the End-User and holds an Identity Assertion whose audience is the Initiator itself, while a distinct confidential client (the Delegate) performs the downstream protocol steps on the End-User's behalf.

OAuth 2.0 {{RFC6749}} and Token Exchange {{RFC8693}} do not define how an authorization server administers an eligibility relationship between two OAuth client registrations.  Deployments today rely on:

*  deployment-specific cross-client identity conventions (for example, {{Google.CrossClient}});
*  ad-hoc administrative configuration invisible to protocol readers; or
*  non-standard Token Exchange conventions.

Token profiles that bind an Identity Assertion to its audience, such as the Identity Assertion JWT Authorization Grant (ID-JAG) {{I-D.ietf-oauth-identity-assertion-authz-grant}}, correctly reject Token Exchange requests in which the requesting client is not the audience of the presented assertion.  That audience check is an important default.  Some deployments, however, deliberately operate a client pair in which one client authenticates the user and another, administratively linked, client acts downstream.  Those deployments need a policy-controlled, auditable exception rather than a bilateral workaround.

This profile defines a Token Exchange path for cross-client delegation using:

*  client authentication to identify the requester;
*  the {{RFC8693}} `actor_token` parameter to establish the acting party;
*  an administered relationship, the Cross-Client Delegation Relationship (CCDR), as an authorization-server-side eligibility rule;
*  the optional {{RFC8693}} `may_act` claim as a per-assertion constraint; and
*  the {{RFC8693}} `act` claim to identify the actor accepted into the issued token.

The administered relationship and `may_act` have related but distinct functions.  The relationship states that a client pairing is eligible under authorization server policy.  A `may_act` claim, when present, restricts which actor may use a particular subject token.  Neither mechanism by itself proves that the Initiator actively conveyed a particular assertion to the Delegate; see {{security-captured}}.

This profile factors out the shape that OpenID Connect cross-client identity conventions ({{Google.CrossClient}}), the ID-JAG Enterprise Broker deployment pattern ({{appendix-broker}}), and similar patterns implement, and defines it once at the Token Exchange layer.

## Applicability {#applicability}

This profile applies when a confidential client acts for an End-User using an Identity Assertion issued to a different client registration, and the authorization server administers a trust relationship between those two registrations.  The Initiator, the Delegate, and the Identity Assertion are all administered by a single authorization server (the IdP); cross-organizational and multi-IdP delegation are out of scope (see {{non-goals}} and {{security-cross-vendor}}).

This profile is intended to compose with token-specific profiles such as ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  That composition currently has an unresolved normative dependency on ID-JAG and is not yet jointly implementable with unmodified ID-JAG; see {{related-idjag}}.  {{appendix-broker}} describes an informative deployment pattern applying this profile to ID-JAG in an enterprise gateway topology.

## Non-Goals {#non-goals}

The following are explicit non-goals of this profile:

CCDR enrollment protocol:
: How a Cross-Client Delegation Relationship is established at the authorization server is a deployment concern.  This profile defines the semantics of the relationship, not its provisioning protocol or user interface.

Cross-organizational trust establishment:
: Recognition of a Delegate administered by a different organization is out of scope; see {{security-cross-vendor}}.

Standardizing an authenticated handoff mechanism:
: This profile does not define or standardize a mechanism that proves the Initiator actively conveyed a particular Identity Assertion to the Delegate.  Such a mechanism raises the handoff-assurance property described in {{authorization-model}}; {{appendix-handoff}} describes a non-normative approach.  See also {{security-captured}}.

End-User consent user experience:
: How delegation is surfaced to the End-User (if at all) is a deployment concern subject to applicable policy; see {{security-consent}}.

Public discovery of the CCDR graph:
: This profile does not require that CCDRs be discoverable by other clients; see {{privacy}}.


# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

Unless otherwise specified, OAuth and Token Exchange terms such as client, confidential client, authorization server, access token, subject token, and actor token are used as defined in {{RFC6749}}, {{RFC8693}}, and {{OpenID.Core}}.

The following terms are used in this document:

Initiator:
: The OAuth client through which the End-User authenticated and to which the IdP issued an Identity Assertion.

Delegate:
: A confidential OAuth client authorized to present the Initiator's Identity Assertion to Token Exchange in order to obtain a token on the End-User's behalf.

Cross-Client Delegation Relationship (CCDR):
: An authorization-server-administered eligibility relationship between an Initiator and one or more Delegates, described in {{ccdr}}.

Per-Assertion Actor Authorization:
: An optional authorization expressed by an {{RFC8693}} `may_act` claim in an Identity Assertion.  It identifies an actor eligible to act on behalf of the subject of that assertion.  It narrows, and does not expand, the actors permitted by the CCDR and the IdP's exchange-time policy.  See {{may-act}}.

Identity Assertion:
: A security token issued by the IdP that conveys claims about the End-User, identifies the Initiator as an intended audience or authorized party, and is suitable for use as `subject_token` in Token Exchange.  In the mandatory-to-implement combination, an Identity Assertion is an OpenID Connect ID Token {{OpenID.Core}}.  A companion profile may extend the profile to a SAML 2.0 assertion {{SAML2.Core}}; see {{processing-steps}}.

Identity Provider (IdP):
: The authorization server that issued the Identity Assertion and performs the Token Exchange defined by this profile.  This document uses "IdP" consistent with {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

Client registration:
: The authorization server's record of a registered OAuth client.  Wherever this document compares, resolves, or relates clients (for example, in a CCDR, in Initiator resolution, or in matching an actor to the authenticated client), the unit of identity is the client registration.  For a `client_id` string to identify a client registration unambiguously in this profile, that string MUST be unique within the issuer namespace against which Initiator resolution, CCDR lookup, `may_act`, and `act` are evaluated.  A deployment whose issuer serves multiple tenants or administrative partitions MUST either assign globally unique `client_id` values within that issuer namespace or use a distinct issuer identifier per partition; this profile does not define a structured tenant-qualified client identifier, so identically named registrations under one issuer are not distinguishable on the wire and MUST NOT occur.

Effective relationship:
: The currently active state of the CCDR for a given (Initiator, Delegate) pair, including any constraints the CCDR carries (such as permitted target audiences, resources, or scopes; see {{ccdr}}).  References in this document to what the effective relationship permits mean this active state, not merely the existence of a pairing.

Examples in this document are illustrative and focus on the claims and parameters relevant to cross-client delegation.  They may omit unrelated claims, parameters, or validation steps required by the underlying specifications for a complete deployment.


# Overview {#overview}

At a high level, the profile operates as follows:

1. The End-User authenticates to the IdP through the Initiator normally.  The Initiator holds an Identity Assertion that identifies the Initiator as its audience or authorized party.  A JWT Identity Assertion MAY contain a `may_act` claim identifying a Delegate that is eligible to act on behalf of its subject.

2. The Initiator conveys the Identity Assertion to the Delegate.  This profile does not define the transport and does not treat the mere audience value as cryptographic proof of this handoff; see {{security-transport}}.

3. The Delegate performs Token Exchange at the IdP, presenting:
   * `subject_token`: the Identity Assertion;
   * `actor_token`: a credential establishing the Delegate as the actor; and
   * client authentication: authentication of the Token Exchange requester as the Delegate.

4. The IdP validates the Cross-Client Delegation Relationship between the Initiator and the Delegate.

5. If the Identity Assertion contains `may_act`, the IdP verifies that it identifies the same Delegate established by `actor_token`.  A mismatch is fatal; the CCDR MUST NOT override or broaden `may_act`.

6. The IdP evaluates exchange-time policy over the tuple (End-User, Initiator, Delegate, requested audiences, resources, scope, authorization details).

7. The IdP issues the requested token with an `act` claim identifying the Delegate as the current actor.

~~~
+----------+  +-----------+   +----------+   +---------------+
| End-User |  | Initiator |   | Delegate |   |      IdP      |
+----+-----+  +-----+-----+   +----+-----+   +-------+-------+
     |              |              |                 |
     | (A) user authentication     |                 |
     |<------------>|<------------------------------>|
     |              |              |                 |
     |              | (B) Identity Assertion         |
     |              |     (for Initiator)            |
     |              |<-------------------------------|
     |              |              |                 |
     |              | (C) convey   |                 |
     |              |   assertion  |                 |
     |              |------------->|                 |
     |              |              | (D) Token       |
     |              |              |     Exchange    |
     |              |              |---------------->|
     |              |              |                 |
     |              |              |  (E) validate   |
     |              |              |  CCDR, may_act, |
     |              |              | current policy  |
     |              |              |                 |
     |              |              | (F) token with  |
     |              |              |   act=Delegate  |
     |              |              |<----------------|
     |              |              |                 |
~~~
{: #fig-overview title="Cross-Client Delegation Overview"}

## Example {#overview-example}

A minimal illustration of the input assertion, the Token Exchange request, and the issued token follows.  Values are abbreviated and form-encoding is omitted for readability.  This example uses one JWT client assertion as both client authentication and the actor token.

Identity Assertion presented as `subject_token`:

~~~json
{
  "iss": "https://idp.example",
  "sub": "user-123",
  "aud": "initiator-client",
  "may_act": {
    "iss": "https://idp.example",
    "sub": "delegate-client"
  }
}
~~~

Token Exchange request from the Delegate:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<identity-assertion>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&actor_token=<delegate-client-assertion>
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
&audience=<target-audience>
&client_assertion_type=<jwt-bearer-client-assertion-type>
&client_assertion=<delegate-client-assertion>
~~~

Issued token payload (subject to the requested token type):

~~~json
{
  "iss": "https://idp.example",
  "sub": "<user identifier for target audience>",
  "aud": "<target audience>",
  "client_id": "delegate-client",
  "act": {
    "iss": "https://idp.example",
    "sub": "delegate-client"
  }
}
~~~


# Authorization Model {#authorization-model}

This profile authorizes the tuple (subject, Initiator, Delegate, target audiences, resources, scope, `authorization_details`) from three inputs, evaluated conjunctively: the administered CCDR ({{ccdr}}), the optional per-assertion `may_act` authorization ({{may-act}}), and the IdP's exchange-time policy ({{security-freshness}}).  The Initiator is part of the tuple because it selects the CCDR and materially affects policy; it is not merely context.  A CCDR and an exchange-time policy evaluation are always required; `may_act` is the only optional input.  Every input that is present must authorize the exchange; none broadens another.

Two independent properties describe how strongly a given exchange is bound, and they must not be conflated:

Token-endpoint authorization:
: What the IdP can verify from the request itself.  This ranges from CCDR-only (the administered relationship plus exchange-time policy) to `may_act`-bound or otherwise bound by an IdP-verifiable artifact carried in or with the `subject_token`.  This property is visible to this protocol and is what {{processing-steps}} evaluates.  The IdP SHOULD prefer an IdP-verifiable per-assertion artifact such as `may_act` where the deployment can supply one, because it narrows authorization from "any Delegate eligible under the CCDR with any eligible assertion" toward "this Delegate for this assertion."

Handoff assurance:
: Whether the Initiator's conveyance of the assertion to the Delegate was authenticated and correlated (see {{appendix-handoff}}) or was uncorrelated bearer forwarding.  This is a deployment property enforced at the Delegate, not at the IdP: nothing representing an authenticated handoff is necessarily present in the Token Exchange request, so the IdP cannot determine it per request, and a compromised Delegate could omit the correlation while sending an otherwise identical request.  Deployments relying on captured-assertion resistance MUST NOT assume the IdP enforces handoff assurance.

CCDR-only, uncorrelated-bearer operation is the weakest combination and carries the captured-assertion risk of {{security-captured}}; it is a legitimate administrative-authorization model but the IdP MUST NOT be relied on to detect that a captured assertion was not conveyed by the Initiator.  Neither property, by itself, establishes End-User consent or per-request authorization by the End-User; see {{security-consent}}.  A deployment SHOULD document the token-endpoint authorization it requires and the handoff assurance it enforces, and when it operates without an IdP-verifiable per-assertion artifact SHOULD apply the compensating controls of {{security}}: short assertion lifetimes and replay controls ({{security-captured}}), narrowly scoped CCDRs ({{security-trust-anchor}}), and constrained downstream authorization ({{security-freshness}}).  Whether a future revision should require an IdP-verifiable per-assertion artifact is left open; see {{open-items}}.


# Cross-Client Delegation Relationship {#ccdr}

A Cross-Client Delegation Relationship (CCDR) is an eligibility relationship, administered at the authorization server, between an Initiator client registration and one or more Delegate client registrations.  The CCDR states that a client pairing is eligible for cross-client delegation under IdP policy.  It is an input to authorization policy, not an entitlement to any specific exchange; see {{security-freshness}}.

A CCDR MAY additionally carry, or reference, constraints that bound the exchanges it makes eligible, such as an allowlist of permitted target audiences, resources, or scopes.  When a CCDR carries such constraints, the policy evaluation in step 7 of {{processing-steps}} MUST enforce them.  Whether or not a CCDR carries such constraints, that policy evaluation MUST be governed by an explicit, administratively configured policy for the specific (Initiator, Delegate) pair; the mere existence of the CCDR is not that policy.

## Establishment and Administration {#ccdr-admin}

Establishing or modifying a CCDR requires authorization by an administrative principal at the IdP.  A client MUST NOT be permitted to establish or expand a CCDR merely by submitting client metadata values through dynamic client registration or registration management {{RFC7591}}.  This profile does not define the enrollment protocol or user interface for CCDR management; that is a deployment concern.

The authorization server MUST evaluate one consistent effective relationship for each (Initiator, Delegate) pair, and the CCDR store and every client comparison in {{processing}} operate on client registrations, per {{conventions}}.  The metadata views defined in {{client-metadata}} expose only the relationship endpoints (the peer `client_id` values), not the audience, resource, or scope constraints an effective relationship may carry.  If those views are exposed, their endpoints MUST be consistent with the same effective relationships the IdP enforces, subject to any recipient-scoped filtering applied under {{privacy}} (an empty or omitted view to a recipient not entitled to see a relationship is such filtering, not a statement that no relationship exists).

The relationship MAY be maintained purely in IdP-side configuration.  This profile does not define public discovery of the relationship graph.

## Client Metadata {#client-metadata}

This profile defines two OPTIONAL client metadata parameters that expose read-only views of administered CCDRs in client registration responses or other authenticated client-metadata representations.  They are not client requests to establish a relationship.

For Initiators:

`authorized_delegates`:
: OPTIONAL.  A JSON array of `client_id` values eligible to act as Delegates for this client.

For Delegates:

`delegate_of`:
: OPTIONAL.  A JSON array of `client_id` values for Initiators on whose behalf this client is eligible to act.

Each array member MUST be a JSON string containing a `client_id` that is unique within the issuer namespace, per {{conventions}}, so that the view names the same registration the CCDR store does.  An omitted parameter conveys no information about whether relationships exist.  An empty array states that the authorization server currently exposes no relationships of that kind to the recipient.

These values are authorization-server-administered policy views.  Per {{ccdr-admin}}, they cannot be self-asserted through registration requests.  An authorization server that receives either parameter in a client registration or registration-management request MUST ignore the submitted value for authorization purposes or reject the request; it MUST NOT establish or expand a CCDR from the submitted value.


# Token Exchange Request {#request}

A Token Exchange request under this profile is a request to the token endpoint as defined in {{RFC8693, Section 2.1}}, with the following parameters:

`grant_type`:
: REQUIRED.  The value `urn:ietf:params:oauth:grant-type:token-exchange`, per {{RFC8693}}.

`subject_token`:
: REQUIRED.  The Initiator's Identity Assertion.

`subject_token_type`:
: REQUIRED.  A token type identifier appropriate for the Identity Assertion type.  For the mandatory-to-implement combination this is `urn:ietf:params:oauth:token-type:id_token`; other identifiers (for example `urn:ietf:params:oauth:token-type:saml2`) require a companion profile that specifies their processing.

`actor_token`:
: REQUIRED.  Under the base profile, an {{RFC7523}} JWT client assertion identifying the Delegate as the acting party, as described in {{actor-token}}.  Companion specifications MAY define additional actor-credential types per {{actor-token-requirements}}.  Note that `actor_token` is OPTIONAL in base {{RFC8693}}; this profile requires it.

`actor_token_type`:
: REQUIRED.  Under the base profile, the value `urn:ietf:params:oauth:token-type:jwt`.  A companion specification that defines an additional actor-credential type also defines its token type identifier.

Other Token Exchange parameters (`audience`, `resource` {{RFC8707}}, `scope`, `authorization_details` {{RFC9396}}, and `requested_token_type`) are included as appropriate for the requested downstream token and its token profile.

The Delegate MUST authenticate the Token Exchange request using a client authentication method accepted by the IdP for confidential clients.  Requests from public clients are outside the scope of this profile.

The authenticated client and the actor are distinct protocol concepts.  This profile requires them to identify the same Delegate; see {{actor-token-requirements}}.

An IdP that supports this profile MUST support the mandatory-to-implement combination: an OpenID Connect ID Token as `subject_token`, an {{RFC7523}} JWT client assertion as `actor_token`, and, as output, a JWT access token {{RFC9068}} that carries the Delegate as actor in the `act` claim ({{issued-token}}).  This combination is independently implementable today and does not depend on any other specification's progression.

Everything else the document describes is OPTIONAL and additive: SAML 2.0 Identity Assertions as input (which require a companion profile, see {{processing-steps}}), actor-credential types defined by companion specifications, the client-metadata views of {{client-metadata}}, refresh tokens ({{issued-token}}), and output token types other than a JWT access token.  In particular, issuing an ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}} as output is OPTIONAL and, as {{related-idjag}} explains, is not yet jointly implementable with unmodified ID-JAG; it is therefore not part of the mandatory core.  An implementation conforms by supporting the mandatory combination.

## Accepted ID Token Profile {#id-token-profile}

An ID Token used as `subject_token` in the mandatory-to-implement combination is validated at the token endpoint by the IdP that issued it, not by a relying party.  Relying-party ID Token validation steps that assume an authorization-response context, such as `nonce` correlation and flow-specific state, do not apply.  This subsection gives the token-endpoint validation algorithm for the ID Token case; it refines step 1 of {{processing-steps}} and does not duplicate other steps.  The IdP MUST validate:

*  **Signature and algorithm.**  The signature verifies under a key the IdP publishes for ID Token signing, using an asymmetric `alg` the IdP supports; the IdP MUST reject a JOSE `alg` of `none` and MUST apply the algorithm-verification guidance of {{RFC8725}}.

*  **Issuer.**  `iss` is exactly the IdP's own issuer identifier.

*  **Temporal validity.**  `exp`, and `nbf`/`iat` where present, are within acceptable bounds.

*  **Type.**  The token is not positively identifiable as a JWT type other than an ID Token, per the type-disambiguation rule in step 1 of {{processing-steps}}.

*  **Initiator resolution.**  The `aud`/`azp` resolution of {{processing-steps}} step 2 yields exactly one Initiator client registration.  The audience-equals-requesting-client check is replaced as described there and is the only ID Token validation this profile relaxes.

*  **Registration status.**  The Initiator and Delegate registrations are currently enabled (also required by {{processing-steps}} step 5).

*  **Issuance and revocation state.**  Where the IdP maintains such state, the ID Token has not been revoked or superseded.

*  **Authentication context.**  Any authentication-context or assurance policy the IdP applies to this exchange is satisfied.

The ID Token additionally MUST NOT be encrypted in the base profile: an encrypted ID Token is encrypted to the registered client (typically the Initiator), and a Delegate normally lacks the decryption key; a deployment that needs encrypted assertions to reach the Delegate MUST define key handling in a companion profile and MUST NOT require the Initiator's private decryption key to be shared.

The ID Token additionally MUST NOT be a sender-constrained or proof-of-possession-bound ID Token (for example, one bound to the Initiator's key via OpenID Connect Key Binding {{OpenID.KeyBinding}}) in the base profile.  The Delegate cannot demonstrate possession of the Initiator's key, so full unrelaxed validation of such an assertion cannot succeed.  A deployment combining key binding with this profile MUST define a presenter-transition mechanism in a companion profile, as required by {{security-captured}}; see also {{related-key-binding}}.  (Note that DPoP {{RFC9449}} binds OAuth access and refresh tokens, not ID Tokens, so it is not a mechanism for binding an ID Token used here; DPoP remains applicable to the access token issued as output and to a Broker-audience access token in {{appendix-broker}}.)

Because {{OpenID.Core}} defines no ID-Token-specific `typ` JOSE header, the IdP separates ID Tokens from other JWT types it issues as described in {{processing-steps}} (from the other types' explicit markers and from issuance context), not by requiring a distinguishing `typ` on the ID Token itself.

Where the ID Token would otherwise carry claims unrelated to the delegation, deployments SHOULD prefer a purpose-minimized assertion; see {{privacy}}.


# Actor Token {#actor-token}

## Requirements {#actor-token-requirements}

This document defines one interoperable actor credential for the base profile: an {{RFC7523}} JWT client assertion for the Delegate.  A companion specification MAY define additional direct actor-credential types.  Such a specification MUST define the credential's validation, actor-identity extraction, replay protection, and binding to the authenticated client; the JWT token-type identifier alone is not sufficient to select an unspecified JWT credential profile.

The actor token MUST:

*  identify the Delegate's `client_id` in both the `iss` and `sub` claims;
*  be verifiable by the IdP under {{RFC7523}};
*  include `iat`, `exp`, and `jti` claims and be subject to replay detection keyed by the IdP's issuer identifier together with the Delegate's `client_id` and the `jti` value, independent of the accompanying subject token, Initiator, or CCDR;
*  not contain an `act` claim, because it is a direct credential for the new actor rather than an existing delegation chain; and
*  resolve to the same `client_id` as the client authenticated for the Token Exchange request.

The IdP MUST reject a JWT access token, ID Token, ID-JAG, or other JWT profile presented as the base profile's `actor_token`, even if its claims happen to include the Delegate's `client_id`.  Each JWT position MUST be validated under the rules for that position; see {{security-confusion}}.

The IdP MUST confirm that the actor established by `actor_token` is the same Delegate identified by client authentication.  A profile that permits the actor to differ from the authenticated OAuth client is outside the scope of this document.

In this profile the actor is the Delegate's client registration itself, and the `act` claim in the issued token identifies that registration.  It does not identify a runtime agent, workload, or user operating behind the Delegate.  In the Enterprise Broker pattern ({{appendix-broker}}), for example, `act` identifies the Broker registration, not any agent invoking the Broker.  A shared client registration does not uniquely identify such a subordinate actor (see {{I-D.mcguinness-oauth-actor-profile}}); representing a distinct runtime actor behind the Delegate requires a workload or agent credential and explicit binding rules, which a companion profile would define and which are outside the scope of this document.

The IdP maps the validated Delegate `client_id` to the actor identifier (`act.iss`, `act.sub`) = (IdP issuer identifier, Delegate `client_id`).  The `iss` claim in an {{RFC7523}} client assertion is itself the Delegate's `client_id`; it is not copied into `act.iss`.

## JWT Client Assertion Actor Token {#actor-token-jwt}

When `actor_token` is a JWT client assertion {{RFC7523}}:

*  `actor_token_type` is `urn:ietf:params:oauth:token-type:jwt`;
*  the `iss` and `sub` claims identify the Delegate's `client_id`;
*  the `aud` claim SHOULD contain the IdP's issuer identifier as its sole value, consistent with the client-authentication audience rules of {{I-D.ietf-oauth-rfc7523bis}}; an IdP MAY accept other audience values permitted by {{RFC7523}}, but new deployments are advised that {{I-D.ietf-oauth-rfc7523bis}} forbids the token endpoint URL as a client-authentication audience;
*  the JWT includes `iat`, `exp`, and `jti` claims (note that `iat` and `jti` are OPTIONAL in base {{RFC7523}}; this profile requires them);
*  the JWT is signed using a key registered for the Delegate; and
*  the IdP applies its {{RFC7523}} validation and replay-detection rules.

The same JWT value MAY be presented as both `client_assertion` and `actor_token`.  When it is, the IdP MUST treat the two parameter occurrences as one acceptance of one assertion for replay-detection purposes; it MUST NOT reject the second occurrence as a replay within the same request.  A later request using the same `jti` is a replay and MUST be rejected.  Alternatively, the Delegate MAY authenticate using another accepted method (for example, mutual TLS {{RFC8705}}) and present a separate JWT client assertion as its actor credential.

These actor-token validation, dual-use processing, and actor-identifier construction rules are designed to align with the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}.


# Per-Assertion Actor Authorization {#may-act}

A JWT Identity Assertion used as `subject_token` MAY include a `may_act` claim as defined in {{RFC8693, Section 4.4}}.  This document does not define an equivalent field for non-JWT Identity Assertions.

How and when the IdP populates `may_act` in an Identity Assertion is an IdP-side, authorization-time decision (for example, driven by CCDR configuration, an End-User authorization step, or a request from the Initiator) and is out of scope for this profile, which specifies only how a `may_act` claim, once present, is processed at Token Exchange.

When present, `may_act`:

*  identifies a party eligible to become the actor for the subject of the Identity Assertion;
*  MUST contain `iss` and `sub` claims that identify the Delegate in the IdP's actor-identifier namespace;
*  MUST identify the same party established by `actor_token`; and
*  narrows the CCDR and exchange-time policy rather than expanding them.

For example, within an Identity Assertion for `user-123` audienced to `initiator-client` (as shown in {{overview-example}}):

~~~json
"may_act": {
  "iss": "https://idp.example",
  "sub": "delegate-client"
}
~~~

For the single-IdP deployment defined by this profile, `may_act.iss` MUST equal the IdP issuer identifier and `may_act.sub` MUST equal the Delegate's `client_id`, unless an applicable companion profile defines a different actor-identifier mapping.  A companion profile that defines a different actor-identifier mapping MUST also define how the equality comparison in step 6 of {{processing-steps}} is performed under that mapping and MUST ensure that mapped identifiers do not collide across namespaces.  The IdP MUST NOT authorize a different Delegate merely because that Delegate is eligible under the CCDR.  A `may_act` mismatch MUST cause the exchange to fail.

Absence of `may_act` does not by itself prohibit the exchange.  In that case, the IdP uses the CCDR and exchange-time policy to decide whether the authenticated Delegate may become the actor.

A `may_act` claim issued by the IdP represents authorization by the IdP for the named actor to act on behalf of the assertion subject.  It does not prove that the Initiator subsequently participated in, or authorized, a particular handoff of the assertion; see {{security-captured}}.


# Token Exchange Processing {#processing}

## Profile Selection {#profile-selection}

A Token Exchange request under this profile is a request to the token endpoint with the parameters of {{request}}.  This profile defines no request parameter that names itself; selection is by request content.  To avoid a circular dependency on this profile's own audience relaxation, the IdP dispatches using only requester authentication, assertion-type-independent validation, and provisional Initiator resolution, and applies the relaxation only after the profile's full checks succeed.  The IdP applies the following phased algorithm:

1. Authenticate the requesting client per {{request}}.

2. Perform the validation of the `subject_token` that does not depend on which profile governs: verify the signature, that the issuer is the IdP's own issuer identifier, temporal validity, and that the token is not positively identifiable as a type other than the declared `subject_token_type`.  These are the checks in step 1 of {{processing-steps}} other than the audience-equals-requesting-client check, which is neither performed nor relaxed at this phase.

3. Provisionally resolve the Initiator from the validated assertion using the rules in step 2 of {{processing-steps}}.

4. Select processing based on the provisional Initiator:

   *  If the resolved Initiator is the same client registration as the authenticated client, this profile does not govern.  The request is a same-client exchange handled under base {{RFC8693}} and the applicable token profile, whose audience check the assertion already satisfies.

   *  Otherwise the request is a cross-client delegation and this profile governs.

5. When this profile governs, apply the remaining rules of {{processing-steps}} (steps 3 through 8: actor token, CCDR, `may_act`, policy, and issuance) in full.  Only successful completion of these rules authorizes relaxing the audience-equals-requesting-client check.  The IdP MUST NOT relax that check, and MUST NOT treat the authenticated client's membership in a multi-valued `aud` claim as satisfying it, on any other basis.

An IdP that receives a request it cannot process under either the base profile or this profile MUST return an error per {{errors}}.

## Processing Rules {#processing-steps}

The following steps specify the full processing.  Per {{profile-selection}}, steps 1 and 2 (subject-token validation other than the audience check, and Initiator resolution) are performed during profile selection and yield the same result regardless of which profile governs; steps 3 through 8 are performed once this profile governs.  Presented as one sequence, the IdP MUST apply:

1. Validate the `subject_token` signature, issuer, temporal validity, and all other requirements applicable to its Identity Assertion type, except for the normal requirement that the assertion identify the requesting client as its audience.  This profile replaces that one check with steps 2, 5, and 6 below; it does not relax any other Identity Assertion validation.  The IdP MUST verify that the assertion's issuer is the IdP's own issuer identifier; assertions from any other issuer, including issuers the IdP trusts for other purposes, are outside this profile.

   The IdP MUST reject a token that it can positively identify as a type other than the one declared by `subject_token_type` (for example, a JWT access token, JWT client assertion, or ID-JAG presented as `urn:ietf:params:oauth:token-type:id_token`), even when its claims resemble an ID Token; see {{security-confusion}}.  {{OpenID.Core}} does not define an ID-Token-specific `typ` JOSE header, so the IdP MUST NOT depend on an ID Token carrying a distinguishing `typ`, and MUST NOT reject an otherwise-valid ID Token because its `typ` is absent or is the generic `JWT`.  Disambiguation instead rests on the explicit type markers carried by the other, confusable JWT profiles, such as the `at+jwt` media type for JWT access tokens {{RFC9068}} and any explicit `typ` used for client-authentication assertions {{I-D.ietf-oauth-rfc7523bis}} or for ID-JAGs, together with the IdP's own record of which JWTs it issued as ID Tokens.  An IdP that mints multiple JWT types from one signing key SHOULD apply explicit typing to those other types per {{RFC8725, Section 3.11}} and/or retain issuance records, so that the positive-mismatch rejection above is effective.  Reliance on claim shape alone to accept a token as an ID Token is NOT RECOMMENDED.  If the IdP cannot distinguish the presented JWT from another JWT type it issues, it MUST reject the token rather than accept it as the declared type.

   An Identity Assertion carrying an existing delegation chain, a JWT `act` claim, or any actor-context representation the IdP itself defines or recognizes in a non-JWT assertion type (for example, a SAML attribute conveying an acting or on-behalf-of party), is outside the base profile and MUST be rejected unless a companion profile defines validation and preservation of that chain.  A companion profile that permits such an input MUST bound chain depth and MUST prevent cycles in the resulting actor chain.

2. Determine exactly one Initiator using the assertion-type-specific rules and the IdP's issuer-scoped client-registration mapping:

   * For an ID Token, the Initiator is identified by the `azp` (authorized party) claim when `azp` is present, and otherwise by the sole `aud` value.  Consistent with {{OpenID.Core}}, `azp` names the party to which the ID Token was issued, so it is authoritative for Initiator selection whether or not it also appears in `aud`.  If `azp` is absent and `aud` has more than one value, the Initiator cannot be determined and the IdP MUST reject the assertion.  The selected identifier MUST resolve to exactly one client registration at the IdP.  Requiring `azp` to be present whenever `aud` is multi-valued is a constraint this profile imposes for unambiguous Initiator selection; it is not required by {{OpenID.Core}}, which leaves `azp` OPTIONAL.

   * SAML 2.0 assertions are not fully specified as `subject_token` by this document and require a companion profile.  Initiator resolution from a SAML assertion would use `AudienceRestriction`/`Audience` and a configured service-provider-to-registration mapping, but a complete SAML input profile must also define subject confirmation, recipient validation, replay and `OneTimeUse` handling, and holder-of-key confirmation.  Until such a companion profile exists, an IdP MUST NOT accept a SAML 2.0 assertion under this profile.

   The IdP MUST reject an assertion when the Initiator cannot be determined unambiguously.  If the resolved Initiator is the same client registration as the client authenticated for the request, this profile does not govern the exchange, per {{profile-selection}}; the IdP processes it under base {{RFC8693}} and the applicable token profile and MUST NOT apply this profile's audience exception to it.

3. Validate `actor_token` according to its token type and determine the actor identity, per {{actor-token}}.

4. Confirm that the actor identity matches the client authenticated for the request (the Delegate).

5. Verify that the Initiator and Delegate registrations are currently enabled and that a CCDR currently authorizes the Delegate to act for the Initiator.

6. If the subject token contains `may_act`, verify that `may_act` identifies the same Delegate.  The IdP MUST NOT ignore a mismatch or use the CCDR to broaden the assertion's per-token authorization.

7. Evaluate exchange-time policy over the tuple (End-User, Initiator, Delegate, requested audiences, resources, scope, authorization details) according to IdP configuration.  The IdP MUST NOT treat successful End-User authentication or the existence of a CCDR as authorization for arbitrary target audiences, resources, scopes, or authorization details.

8. On success, issue the requested token according to {{issued-token}}.

## Errors {#errors}

If any step in {{processing-steps}} fails, the IdP MUST return an error response as specified in {{RFC6749, Section 5.2}} and {{RFC8693, Section 2.2.2}}:

*  Failures of client authentication MUST use the `invalid_client` error code per {{RFC6749}}.

*  An invalid `subject_token` or `actor_token`, a CCDR that does not authorize the (Initiator, Delegate) pair, a `may_act` mismatch, or any other policy-based rejection of the presented tokens MUST use the `invalid_request` error code, per {{RFC8693, Section 2.2.2}}.

*  A request whose `audience` or `resource` is unacceptable SHOULD use the `invalid_target` error code per {{RFC8693}}.

When one JWT value is presented in both `client_assertion` and `actor_token` ({{actor-token-jwt}}) and a single defect fails it in both roles (for example, a bad signature or expiry), the IdP MUST report `invalid_client`; client authentication takes priority over actor-token validation for error classification.

An `invalid_target` response can indicate that the presented tokens and the (Initiator, Delegate) relationship were otherwise acceptable, since a target is only assessed for a request that could otherwise succeed.  An IdP that treats the existence of a CCDR as confidential, including from a requester authenticated as the Delegate, therefore MUST return `invalid_request` uniformly for CCDR, `may_act`, policy, and target failures rather than distinguishing them, accepting the reduced target diagnostics as the cost of not disclosing the relationship.  This document does not otherwise constrain the order in which an IdP evaluates the checks in {{processing-steps}}; the ordering there is written for exposition.

Error responses for actor-token validation and `may_act` or CCDR authorization failures SHOULD NOT reveal the IdP's relationship graph.  In particular, `error_description` values MUST NOT enumerate eligible Delegates or Initiators, and error responses SHOULD NOT allow a client to distinguish "no CCDR exists for this pair" from other policy rejections.  Implementations SHOULD take reasonable measures to minimize observable timing differences between these rejection paths, while recognizing that eliminating timing side channels entirely is difficult.  See {{privacy}}.


# Issued Token {#issued-token}

Every token that this profile causes to be presented to a token consumer (a resource server or a downstream authorization server) MUST make the accepted actor context available to that consumer.  This requirement governs access tokens and other consumer-facing tokens; it does not apply to a refresh token, which is presented back to the IdP rather than to a token consumer and which retains the actor, Initiator, and CCDR context in IdP-side state per this section.  For the mandatory-to-implement combination, the consumer-facing output is a JWT access token {{RFC9068}}.  A JWT issued under this profile MUST contain:

*  `sub`: an identifier for the End-User appropriate to the issued token's audience;

*  `aud`: the audience determined for the requested token;

*  `act`: an actor claim per {{RFC8693, Section 4.1}} identifying the Delegate as the current actor:

~~~json
"act": {
  "iss": "https://idp.example",
  "sub": "delegate-client"
}
~~~

The issued JWT MUST also contain `client_id` when required by the issued token's profile (for example, {{RFC9068}} or ID-JAG).  The outer `act.sub` MUST NOT be omitted merely because it duplicates a `client_id` claim.  The two claims have distinct semantics: `client_id` identifies the client registration under which the token will be used, while `act.sub` identifies the acting party accepted at exchange time.

The outer `act` object MUST contain both `iss` and `sub`.  In the single-IdP deployment defined by this profile, `act.iss` MUST be the IdP issuer identifier as the namespace for the actor identifier, and `act.sub` MUST be the Delegate's `client_id`.

The Initiator MUST NOT be added as a nested prior actor solely because its client identifier was the audience of the input Identity Assertion or because a `may_act` claim was present.  A nested Initiator MAY be included only when an additional mechanism establishes that the Initiator actively participated in the delegation of that particular assertion:

~~~json
"act": {
  "iss": "https://idp.example",
  "sub": "delegate-client",
  "act": {
    "iss": "https://idp.example",
    "sub": "initiator-client"
  }
}
~~~

As required by {{RFC8693}}, nested prior actors are informational and MUST NOT be used by token consumers for access-control decisions.

Other claims are included according to the requested token type and applicable token profile.  When the IdP issues an opaque access token, its OAuth Token Introspection {{RFC7662}} response MUST include the `act` member with the same semantics, as permitted by {{RFC8693, Section 4.1}}, so that the actor context is available to the protected resource for token processing; if the opaque token is sender-constrained, the introspection response MUST also convey the corresponding `cnf` member.  The IdP MUST NOT issue a token under this profile when the selected token type and deployment cannot convey the current actor to the token consumer.  In particular, issuing a token type that has no defined representation for actor context (for example, a SAML 2.0 assertion, for which this document defines no `act` equivalent) is outside the base profile absent a companion profile that defines how the actor is conveyed.

Where the requested token type and deployment support proof of possession, the issued token SHOULD be sender-constrained to a key controlled by the Delegate (for example, via mutual TLS certificate binding {{RFC8705}} or DPoP {{RFC9449}}), with the binding conveyed by the `cnf` claim {{RFC7800}}.  Sender constraining the issued token bounds the value of a stolen token and complements the captured-assertion controls in {{security-captured}}.

The Token Exchange response is constructed according to {{RFC8693, Section 2.2}} and the requested token's profile.  The IdP SHOULD NOT issue a refresh token in response to an exchange under this profile.  If the originating subject token carried a `may_act` claim, the IdP MUST NOT issue a refresh token: a `may_act` claim is a point-in-time, per-assertion authorization (see {{may-act}}) that a refresh token, decoupled from that assertion and typically outliving it, cannot preserve.

If the IdP issues a refresh token in the absence of `may_act`, it MUST bind the refresh token to the Delegate and MUST record the concrete authorization actually granted at the original exchange, that is, the audience or audiences, resource or resources, scope, and `authorization_details` present in the issued token, together with the Initiator and CCDR context.  On each use of the refresh token, the IdP MUST re-evaluate the current status of the Delegate, Initiator, and CCDR and MUST re-apply the policy evaluation of step 7 of {{processing-steps}}.  The audiences, resources, scope, and `authorization_details` obtainable through the refresh token MUST NOT exceed the recorded original grant, and MUST be further reduced when exchange-time policy or the effective relationship is narrower.  Revoking or narrowing the CCDR MUST prevent the refresh token from obtaining authorization no longer permitted by the effective relationship.


# Authorization Server Metadata {#as-metadata}

An IdP advertises support for this profile in its authorization server metadata {{RFC8414}} using the `authorization_grant_profiles_supported` parameter defined by {{I-D.ietf-oauth-identity-assertion-authz-grant}}, together with a companion parameter defined by this document:

`authorization_grant_profiles_supported`:
: The set of grant profile identifiers the authorization server supports, as defined by {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  An authorization server that supports this profile MUST include the value `urn:ietf:params:oauth:grant-profile:cross-client-delegation` in this array and MUST also include `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported`.

`cross_client_delegation_token_types_supported`:
: OPTIONAL.  A JSON array of objects describing the (`subject_token_type`, `actor_token_type`) combinations the authorization server will accept under this profile.  Each object MUST contain a `subject_token_type` member and an `actor_token_type` member, each of whose values is a token type identifier URI.  Unknown object members MUST be ignored.  If this parameter is omitted, support for the mandatory ID Token and JWT client assertion combination is implied.  If it is present, it MUST include that combination.

Example authorization server metadata:

~~~json
{
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:token-exchange"
  ],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:cross-client-delegation"
  ],
  "cross_client_delegation_token_types_supported": [
    {
      "subject_token_type":
        "urn:ietf:params:oauth:token-type:id_token",
      "actor_token_type":
        "urn:ietf:params:oauth:token-type:jwt"
    }
  ]
}
~~~

These metadata values indicate what the authorization server implements.  They do not disclose any particular CCDR and do not guarantee that a particular Initiator, Delegate, subject token, actor token, audience, resource, scope, or authorization request will be accepted for a given exchange.

## Initiator-Facing Discovery {#initiator-discovery}

How an Initiator learns which token kind to convey to a Delegate (for example, how an agent learns what a gateway expects) is out of scope for this profile.  It is a Delegate-side or application-layer metadata concern with different actors and different discovery surfaces from authorization server metadata.


# Relationship to Existing Mechanisms {#related}

## RFC 8693 `may_act` {#related-may-act}

This profile uses `may_act` without changing its {{RFC8693}} semantics.  `may_act` is an input authorization statement identifying an eligible actor; `actor_token` proves the proposed actor's identity; and `act` records the actor accepted into the output token.  The conjunctive relationship between the CCDR and `may_act` is specified in {{authorization-model}} and {{security-conjunctive}}.

## OpenID Connect Cross-Client Identity {#related-google}

Deployments MAY implement both OpenID Connect cross-client identity conventions ({{Google.CrossClient}}) and this profile.  This profile keeps the base Identity Assertion audience unchanged unless a deployment deliberately adds `may_act` as an extension.  It does not require an ID Token audience array.

## Identity Assertion JWT Authorization Grant (ID-JAG) {#related-idjag}

This profile is intended to compose with ID-JAG Token Exchange {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  When the requested token is an ID-JAG (`requested_token_type` of `urn:ietf:params:oauth:token-type:id-jag`), the issued ID-JAG carries the Delegate as the current actor, and the ID-JAG's base audience and client-binding rules otherwise continue to apply, with only the Identity Assertion audience-equals-requesting-client check replaced by the processing rules in {{processing}}.  No other ID-JAG validation rule is relaxed.

This composition has an unresolved normative dependency.  ID-JAG currently requires, without exception, that the Identity Assertion's audience match the `client_id` of the authenticating client, and provides no extension hook for this profile to introduce an exception.  An implementation therefore cannot presently satisfy both this profile and unmodified ID-JAG, and this document has no authority to override ID-JAG's requirement.  Closing the gap requires one of: an extension hook in ID-JAG; a coordinated update in which this document formally updates ID-JAG (for example, via an "Updates" relationship) with matching text in both drafts; or defining cross-client delegation within ID-JAG.  This prerequisite is tracked as open question 7 in {{open-items}}; until it is resolved, the ID-JAG composition and the Enterprise Broker pattern of {{appendix-broker}} illustrate the intended end state rather than being jointly implementable with unmodified ID-JAG.

The mandatory-to-implement combination ({{request}}), which issues a JWT access token, does not depend on that resolution and is implementable today.  It does, however, reuse two ID-JAG-defined conventions even for the mandatory combination: the `authorization_grant_profiles_supported` metadata parameter ({{as-metadata}}) and the `urn:ietf:params:oauth:grant-profile:` value convention ({{iana}}).  This discovery-and-registration dependency is distinct from the audience-resolution dependency above; if ID-JAG does not progress, a future revision may define an independent discovery parameter and grant-profile registration.

The Initiator is not automatically a nested actor.  Recording it as a prior actor requires an additional authenticated handoff mechanism as described in {{issued-token}}.

{{appendix-broker}} describes the Enterprise Broker deployment pattern, an informative application of this profile to ID-JAG.

## OpenID Connect Key Binding {#related-key-binding}

The mechanism specified by OpenID Connect Key Binding {{OpenID.KeyBinding}} can bind an Identity Assertion to the Initiator's key, but that binding alone is not resolved by this profile: without a presenter-transition mechanism, the Delegate cannot demonstrate proof of possession for such an assertion.  Deployments requiring both key binding and cross-client delegation need to define that presenter-transition mechanism via a companion profile.  This remains an open problem across the delegation space and is not unique to this profile.

## OAuth Actor Profile for Delegation {#related-actor-profile}

The actor-token validation and `act` claim construction rules in this profile align with the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}, including its use of an {{RFC7523}} client assertion as both client authentication and an actor credential.


# Security Considerations {#security}

The OAuth 2.0 Security Best Current Practice {{RFC9700}} applies in addition to the requirements in this section.

## Trust Anchor for the Relationship {#security-trust-anchor}

The CCDR is an eligibility input to authorization policy.  Its correctness depends on the IdP's administration discipline.  Deployments SHOULD:

*  restrict the ability to establish a CCDR to administrative principals;
*  scope relationships as narrowly as possible;
*  support prompt revocation; and
*  audit relationship changes.

## CCDR and `may_act` Are Conjunctive {#security-conjunctive}

The CCDR and `may_act` are conjunctive when both are present.  A CCDR MUST NOT override a `may_act` claim that identifies a different actor.

The presence of `may_act` does not eliminate the need for exchange-time policy.  The claim describes an eligible actor for the assertion subject; it does not grant arbitrary audience, resource, scope, or `authorization_details` values.

## Captured Identity Assertions {#security-captured}

In the absence of a cryptographically authenticated handoff, any Delegate eligible under the CCDR might be able to use a captured bearer Identity Assertion.  A `may_act` claim limits which Delegate can exploit such a captured assertion but does not prove that the Initiator conveyed it.

Deployments SHOULD use short assertion lifetimes.  Where the Identity Assertion carries a `jti` or another unique identifier, deployments operating a one-time handoff SHOULD record successful use until the assertion expires and reject subsequent use.  Deployments that intentionally permit repeated exchange of one assertion need to bound that behavior by audience, resource, scope, time, and policy.  These subject-token replay controls are separate from the mandatory replay detection for the Delegate's actor token.

Where sender constraining is used, the deployment MUST define how proof of possession transitions from the Initiator to the Delegate; an assertion bound only to the Initiator's key cannot be presented by the Delegate.  See {{related-key-binding}}.

Because this profile raises the value of a captured assertion, deployments SHOULD also sender-constrain the token issued by the exchange to a Delegate-controlled key, per {{issued-token}}, so that a token stolen after issuance is likewise bounded.

## Delegate Compromise {#security-delegate-compromise}

Compromise of a Delegate's credentials may permit obtaining tokens for every Initiator linked to that Delegate.  Relationships SHOULD be narrowly scoped, and Delegate credentials SHOULD be rotated with the discipline applied to sensitive service credentials.

Where Identity Assertions include `may_act`, compromise impact is limited to assertions that name the compromised Delegate, but existing assertions remain usable until they expire, are revoked, or are rejected by exchange-time policy.

## Exchange-Time Policy Freshness {#security-freshness}

The IdP MUST evaluate policy at exchange time.  The CCDR and `may_act` establish eligibility, not entitlement to any specific exchange.  Audience, resource, scope, and `authorization_details` remain subject to policy.

The IdP MUST re-check the current status of the Initiator, the Delegate, and their CCDR at each mint.  A `may_act` authorization, when it was present, is bound to the originating assertion and is not re-established at a later mint that does not re-present that assertion; this is why {{issued-token}} forbids issuing a refresh token for an exchange whose subject token carried `may_act`.  Prior actors in any nested `act` chain are informational under {{RFC8693}} and MUST NOT be used for access-control decisions.

## Transport of the Identity Assertion {#security-transport}

This profile does not define how the Initiator conveys the Identity Assertion to the Delegate.  Transport SHOULD provide confidentiality, integrity, and session correlation.

A Delegate SHOULD accept Identity Assertions only within authenticated invocations that it can correlate with the presenting party, and SHOULD NOT accept assertions submitted over unauthenticated channels.  This applies with particular force when the Initiator is a public client whose `client_id` is not a secret, because possession of a captured assertion is then the only barrier an unauthenticated submitter must overcome.

Deployments requiring proof that the Initiator actively authorized a particular handoff need an additional Initiator-authenticated mechanism.  Such a mechanism MAY allow the IdP to record the Initiator as a nested prior actor per {{issued-token}}.  {{appendix-handoff}} describes the correlation controls one deployment pattern applies.

## Cross-Organizational Delegates {#security-cross-vendor}

The CCDR is administered by the IdP that validates the Identity Assertion.  Cross-vendor gateway deployments require the IdP to recognize the Delegate and its actor credential.  This profile does not define how trust is established across organizations.

## Consent Semantics {#security-consent}

Neither a CCDR nor `may_act` by itself establishes that the End-User understood or approved disclosure to the Delegate.  Deployments are expected to have an applicable consent, enterprise-policy, or other authorization basis and SHOULD reflect delegation in user experience where appropriate.

## Revocation and Previously Issued Tokens {#security-revocation}

Revoking an Initiator, Delegate, or CCDR MUST prevent new tokens from being minted through this profile.  Redeeming a refresh token issued under this profile is minting through this profile for this purpose and is subject to the re-evaluation required by {{issued-token}}.  A CCDR change does not, by itself, invalidate tokens already issued.  Deployments that require immediate invalidation need an operational revocation mechanism, such as token revocation, introspection-backed tokens, or another deployment-specific control.  Otherwise, short `exp` values on issued tokens bound the residual authorization window.

The same bounded window applies to `may_act`.  A `may_act` claim is evaluated when the exchange occurs.  If the named Delegate or its CCDR is later revoked, the status of tokens already minted depends on the deployment's revocation mechanism and the issued token's lifetime.

## Token and Key Confusion {#security-confusion}

Identity Assertions, actor tokens, and client authentication assertions may all be JWTs signed within the same deployment.  The IdP MUST validate each token under the rules for the position in which it is presented, including explicit typing and audience restrictions where available, so that a token issued for one function cannot be replayed in another (for example, an ID Token presented as `actor_token`, or a client assertion presented as `subject_token`).  The recommendations of {{RFC8725}} apply.

When the same client assertion is used in both `client_assertion` and `actor_token`, accepting it once for the request does not authorize its use in any later request.  Replay state MUST be shared across both processing paths so that changing only the parameter position cannot bypass replay detection.


# Privacy Considerations {#privacy}

The CCDR graph reveals which clients are administratively linked, which may itself be sensitive (for example, disclosing an organization's internal gateway topology or vendor relationships).  This profile does not define public discovery of the relationship graph, and {{errors}} requires that error responses not enumerate it.  The client metadata views defined in {{client-metadata}}, when exposed at all, SHOULD be restricted to the affected clients and administrative principals.

Issuing a token for a new audience on the basis of an assertion issued to the Initiator can correlate a user's identity across clients and audiences.  IdPs that use pairwise or audience-scoped subject identifiers SHOULD apply the subject identifier policy of the issued token's audience rather than copying an Initiator-scoped identifier; composition profiles such as ID-JAG define audience-appropriate subject resolution.

The `act` claim in the issued token discloses the Delegate's identity to downstream token consumers.  This disclosure is deliberate: making the acting party explicit is a design goal of this profile.

The most immediate disclosure in this profile is to the Delegate itself: to perform the exchange, the Delegate receives the Initiator's complete Identity Assertion and can read every claim in it, including claims present for the Initiator's own purposes and never intended for the Delegate to consume.  Accordingly:

*  Identity Assertions intended for cross-client delegation SHOULD carry the minimum claims the exchange requires; deployments SHOULD prefer a purpose-built assertion or a token handle over a general-purpose ID Token when the ID Token would otherwise carry unrelated personal data.

*  The Delegate MUST NOT treat claims read from the Identity Assertion as authorization for the downstream resource; authorization derives from the token the IdP issues after the exchange, not from the input assertion (see {{appendix-handoff}}).

*  Deployments SHOULD protect the assertion in transit and at the Delegate against logging, retention, and secondary use beyond performing the exchange, consistent with {{security-transport}}.

*  Deployment documentation SHOULD state that the Delegate becomes able to observe the Initiator-audience identity claims carried by assertions it is authorized to exchange.


# IANA Considerations {#iana}

## OAuth URI Registration {#iana-oauth-uri}

This document requests IANA to register the following value in the "OAuth URI" registry established by {{RFC6755}}:

*  URN: `urn:ietf:params:oauth:grant-profile:cross-client-delegation`
*  Common Name: OAuth 2.0 Token Exchange Cross-Client Delegation Profile
*  Change Controller: IETF
*  Specification Document: {{as-metadata}} of this document

Note: The `authorization_grant_profiles_supported` metadata parameter and the `urn:ietf:params:oauth:grant-profile:` value convention referenced by this document are defined by {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  The registration above is contingent on the progression of that specification; if its grant profile identifier scheme changes, this registration should be aligned with the final scheme.

## OAuth Dynamic Client Registration Metadata Registration {#iana-client-metadata}

This document requests IANA to register the following values in the "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}:

*  Client Metadata Name: `authorized_delegates`
*  Client Metadata Description: Array of client identifiers eligible to act as Delegates for this client under the Cross-Client Delegation profile
*  Change Controller: IESG
*  Specification Document: {{client-metadata}} of this document

and:

*  Client Metadata Name: `delegate_of`
*  Client Metadata Description: Array of client identifiers for Initiators on whose behalf this client is eligible to act under the Cross-Client Delegation profile
*  Change Controller: IESG
*  Specification Document: {{client-metadata}} of this document

## OAuth Authorization Server Metadata Registration {#iana-as-metadata}

This document requests IANA to register the following value in the "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

*  Metadata Name: `cross_client_delegation_token_types_supported`
*  Metadata Description: JSON array of objects describing the subject token type and actor token type combinations accepted under the Cross-Client Delegation profile
*  Change Controller: IESG
*  Specification Document: {{as-metadata}} of this document

## JWT Claims {#iana-jwt-claims}

This document requests no new JWT claim registrations.  The `act` and `may_act` claims are registered by {{RFC8693}}, and the `jti` claim is registered by {{RFC7519}}; this profile uses them without modification.


--- back

# Deployment Pattern: ID-JAG Enterprise Broker {#appendix-broker}

This appendix is informative.  It describes the "Enterprise Broker" deployment pattern: an application of this profile to ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}} in which a confidential gateway (the Broker, acting as the Delegate) obtains an ID-JAG following an invocation from a centrally administered initiating client (the Managed Client, acting as the Initiator).  All normative mechanics (CCDR administration, `actor_token` handling, `act` claim construction, and `may_act` interaction) follow the body of this document; this appendix adds the deployment controls that surround them.

## Deployment Context {#appendix-context}

Enterprise deployments of agent-plus-gateway topologies increasingly exhibit a common shape:

*  The initiating client (typically an agent) is distributed via Mobile Device Management (MDM) or a comparable enterprise deployment channel.

*  Every install of the agent receives the same centrally administered `client_id`.  The `client_id` identifies the managed software registration; it is not a secret and does not authenticate an individual installation.

*  Each End-User signs in to the IdP via the agent using an authorization code flow with PKCE {{RFC7636}}, producing an Identity Assertion whose audience equals the pinned agent `client_id` and whose subject is per-user.

*  When supported by the deployment, the same authorization produces an access token for invoking the Broker.  That access token identifies the Managed Client and End-User and can be sender-constrained to an instance key (for example, with DPoP {{RFC9449}}).

*  The agent invokes a gateway (the Broker) to reach one or more upstream Resource Authorization Servers.

*  The gateway is a distinct confidential OAuth client, administered by the same enterprise IT organization that provisioned the agent.

Under base ID-JAG rules, the Broker cannot obtain an ID-JAG on the End-User's behalf: the ID Token's audience is the Managed Client, not the Broker, and the ID-JAG Token Exchange audience check rejects the exchange.  This profile defines the policy-controlled exception for an authorized client pair; this appendix describes the enterprise deployment controls surrounding that exception.

The pattern deliberately replaces literal audience equality with a narrower policy decision over a client pair.  These controls authorize a known Broker to present an assertion issued for a known Managed Client.  They do not prove, merely from the ID Token audience or the CCDR, that the Managed Client conveyed a particular assertion, and they increase the utility of a captured Identity Assertion.  The authenticated handoff and replay controls described in {{appendix-handoff}} are therefore part of the deployment pattern, not optional consequences of static registration.

## Actors {#appendix-actors}

Managed Client:
: An initiating OAuth client (the Initiator) whose registration is administered or approved by the IdP and distributed via MDM or a comparable channel with a pinned `client_id`.  It is commonly a public client.  Users sign in through the Managed Client and receive Identity Assertions.

Broker:
: A confidential OAuth client (the Delegate) that mediates access to upstream Resource Authorization Servers.  The Broker authenticates the Token Exchange request to the IdP.

End-User:
: The subject of the delegation.  Authenticates once through the Managed Client.

## Flow {#appendix-flow}

1. The End-User authenticates the Managed Client at the IdP via authorization code flow with PKCE.  The Managed Client receives an ID Token with the Managed Client as audience and the End-User as subject, optionally carrying a `may_act` claim identifying the Broker, and, when supported, an access token for invoking the Broker.

2. The Managed Client invokes the Broker and conveys the ID Token within that request.  The Broker authenticates the invocation, validates the Broker-audience access token or equivalent request credential, and correlates it with the ID Token as described in {{appendix-handoff}}.

3. The Broker performs Token Exchange at the IdP per this profile, with `requested_token_type` of `urn:ietf:params:oauth:token-type:id-jag`, `audience` identifying the target Resource Authorization Server, and `resource`, `scope`, and `authorization_details` as appropriate for the target protected resource.

4. The IdP validates the CCDR, confirms `may_act` if present, evaluates exchange-time policy, resolves the Broker's client identifier at the target Resource Authorization Server, and issues an ID-JAG with `act` identifying the Broker.

5. The Broker presents the ID-JAG at the target Resource Authorization Server per base ID-JAG, authenticating under the client identifier carried by the ID-JAG and proving possession of the bound key when the ID-JAG is sender-constrained.

For example, the Token Exchange request includes:

~~~
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&subject_token=<managed-client-id-token>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&actor_token=<broker-client-assertion>
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
&audience=https://authorization-server.example/
&resource=https://api.example/
&scope=tool.invoke
&client_assertion_type=<jwt-bearer-client-assertion-type>
&client_assertion=<broker-client-assertion>
~~~

An illustrative issued ID-JAG contains:

~~~json
{
  "iss": "https://idp.example",
  "sub": "<user identifier for the target trust domain>",
  "aud": "https://authorization-server.example/",
  "client_id": "broker-client-at-resource-as",
  "cnf": {
    "jkt": "<Broker key thumbprint>"
  },
  "act": {
    "iss": "https://idp.example",
    "sub": "broker-client-at-idp"
  }
}
~~~

The `cnf` claim {{RFC7800}} appears when the ID-JAG is sender-constrained to a Broker-controlled key.

The Managed Client is not automatically included as a nested prior actor.  The ID Token audience and CCDR establish the client relationship, not proof that the Managed Client actively authorized this particular handoff.  A nested Managed Client is appropriate only when the selected handoff mechanism satisfies the authenticated prior-actor requirements of {{issued-token}}.

## Trust Model {#appendix-trust-model}

The authorization decision rests on three facts the IdP itself administers:

1. The Managed Client registration is eligible to participate in the enterprise deployment and is one endpoint of the CCDR.

2. The Broker registration is the other endpoint of the CCDR, and the Broker is authenticated on the Token Exchange request.

3. The CCDR authorizes this specific Broker to act for this specific Managed Client.

These facts allow the IdP to replace literal audience equality with an explicit policy check over the (Managed Client, Broker) pair.  They establish eligibility, not proof of a particular handoff and not entitlement to arbitrary audiences, resources, scopes, or authorization details.

Within the single-IdP scope of this pattern, the IdP re-checks the current status of the Managed Client, the Broker, and their CCDR at each mint, per {{security-freshness}}.  Short `exp` values on issued tokens bound the residual window when the deployment does not provide immediate revocation of already-issued tokens.

The pattern intentionally increases the value of an Identity Assertion captured in transit or compromised at the Broker.  Its security therefore also depends on the Broker enforcing the authenticated handoff rules, short assertion lifetimes and replay policy, narrowly scoped CCDRs, and sender constraining where the presenter-transition problem has been addressed.

This trust model does not itself establish End-User consent; see {{security-consent}}.

## Authenticated Handoff and Correlation {#appendix-handoff}

The CCDR authorizes a client pairing but does not demonstrate that the Managed Client conveyed a particular ID Token to the Broker.  This deployment pattern therefore expects the ID Token to arrive within an authenticated Broker invocation.

Where the Managed Client presents an access token for the Broker, the Broker validates that token according to its type and local policy and correlates it with the accompanying ID Token.  Correlation includes, where available:

*  the same validated IdP issuer or a configured issuer relationship;

*  the same End-User subject, or a trusted mapping between the subject identifiers in the two tokens;

*  a `client_id` or equivalent authorized-party value identifying the Managed Client registration that is the Initiator endpoint of the CCDR;

*  compatible authentication time, session, tenant, and authorization context; and

*  proof of the same Managed Client instance key when the invocation and Identity Assertion support a common sender constraint.

The Broker rejects an invocation when the request credential and ID Token cannot be correlated at the assurance required by deployment policy.  The Broker does not treat claims from the ID Token as authorization for the upstream resource before the IdP has successfully performed the ID-JAG Token Exchange.

An {{RFC8693}} `may_act` claim naming the Broker provides per-assertion actor authorization and further narrows the CCDR.  It does not, by itself, authenticate the transport or prove that the Managed Client initiated a particular request.

Deployments unable to authenticate and correlate the handoff operate in a weaker relationship-only bearer mode.  Such deployments need to document that capturing any eligible Managed Client ID Token may be sufficient for an authorized Broker to obtain an ID-JAG, and should apply correspondingly short lifetimes, one-time-use or replay controls, narrow CCDRs, and constrained downstream grants.

## What This Pattern Does Not Cover {#appendix-non-scope}

This pattern is deliberately narrow.  It does not apply to:

*  **Unadministered client registrations.**  The registration mechanism is not decisive: static registration alone is insufficient, and dynamic client registration {{RFC7591}} is not automatically disqualifying.  The base ID-JAG audience check applies unless the IdP has established, through an administrative or otherwise trusted process, that both registrations may participate in the specific CCDR.

*  **Public clients acting as Brokers.**  The Broker must be a confidential client.  The Managed Client may be public; public clients presenting their own ID Tokens directly are the base ID-JAG case rather than this pattern.

*  **Cross-organizational Brokers.**  When the Broker and the Managed Client are controlled by different organizations, establishing the CCDR requires trustworthy onboarding, identifier namespacing, credential validation, and administrative authority.  Those mechanisms are outside this pattern, although a single IdP may recognize such a Broker once that trust has been established.

*  **Autonomous (no-user-present) invocations.**  The pattern assumes an Identity Assertion the Managed Client obtained on the End-User's behalf.  Deployments where the Managed Client operates without a user session (scheduled tasks, background workers) require a different mechanism.

*  **Uncorrelated bearer forwarding as a high-assurance handoff.**  Merely placing an ID Token in a request does not authenticate the Managed Client's participation.  Relationship-only bearer deployments have the limitations described in {{appendix-handoff}}.

## Composition with Base ID-JAG {#appendix-idjag-composition}

This pattern is additive over base ID-JAG.  It does not modify base ID-JAG processing rules for cases outside its scope.  Specifically:

*  Base ID-JAG's audience check applies unchanged where no CCDR exists.  This profile specifies the CCDR-authorized exception to that check, subject to the unresolved ID-JAG dependency described in {{related-idjag}}; this informative deployment pattern does not independently override the base requirement, and it is jointly implementable with ID-JAG only once that dependency is resolved.

*  The issued ID-JAG's structure, target-specific subject resolution, tenant context, and downstream presentation to the Resource Authorization Server follow base ID-JAG.

*  The ID-JAG `client_id` identifies the Broker's client registration at the target Resource Authorization Server, because the Broker will authenticate under that registration when redeeming the ID-JAG.

*  The Broker's `client_id` at the IdP, the Broker's `client_id` at the Resource Authorization Server, and the Managed Client's `client_id` at the IdP may all differ.  The IdP needs a trusted mapping from the authenticated Broker to the target-side Broker registration.

*  The ID-JAG `sub` and related subject or tenant claims identify the End-User in the namespace expected by the target Resource Authorization Server, rather than blindly copying an Initiator-scoped pairwise subject; see {{privacy}}.

*  When the ID-JAG is sender-constrained, its `cnf` claim binds the ID-JAG to a Broker-controlled key that the Broker proves when redeeming it.

The CCDR-authorized relaxation applies only to the audience check on the Identity Assertion at Token Exchange time, and only when the processing rules of {{processing}} are followed.

Client attestation ({{I-D.ietf-oauth-attestation-based-client-auth}}) can authenticate a particular Managed Client or Broker instance and prove possession of an instance key.  It complements rather than replaces the CCDR: attestation establishes instance authenticity, while the CCDR authorizes the cross-client relationship.


# Design Decisions and Open Items {#open-items}

## Positions Taken in This Revision {#design-decisions}

The following positions are settled for this revision.  They are recorded here, rather than left as open questions, so implementers know what to build; the working group may revisit any of them.

Two conformant authorization modes:
: Following the two-axis model of {{authorization-model}}, this revision does not require an IdP-verifiable per-assertion artifact: both CCDR-only and `may_act`-bound token-endpoint authorization are conformant, with the `may_act`-bound form RECOMMENDED.  Handoff assurance (authenticated-and-correlated versus uncorrelated bearer) is a deployment property this protocol does not enforce.  Whether a future revision should require an IdP-verifiable per-assertion artifact is left open (see {{security-captured}} for the risk that motivates the question).

Actor credential:
: The base profile is limited to an {{RFC7523}} JWT client assertion as `actor_token` ({{actor-token}}).  Workload credentials and access tokens as actor credentials are deferred to companion profiles.

ID-JAG relationship:
: This document declares its dependency on ID-JAG in prose ({{related-idjag}}) and does not assert a formal "Updates" relationship to it.  ID-JAG is an unpublished Internet-Draft, and the audience-check exception is best introduced by coordinated normative text in ID-JAG itself rather than by this document claiming to modify it.  The mandatory-to-implement combination does not depend on that audience-resolution coordination; the ID-JAG output composition does.  Separately, this profile's discovery and IANA model reuse ID-JAG's `authorization_grant_profiles_supported` parameter and `grant-profile` URN convention even for the mandatory combination ({{related-idjag}}); if ID-JAG does not progress, a future revision may define an independent discovery parameter and registration.  Which coordination path the working group adopts remains open (an extension hook in ID-JAG, coordinated text in both drafts, or defining cross-client delegation within ID-JAG).

Scope:
: This revision retains the generic Token-Exchange-layer profile.  The mandatory-to-implement core is a signed OpenID Connect ID Token as `subject_token` with an {{RFC7523}} JWT client assertion as `actor_token`, producing a JWT access token {{RFC9068}} that carries the Delegate as actor ({{request}}, {{issued-token}}).  SAML 2.0 Identity Assertions (which require a companion profile), the client-metadata views ({{client-metadata}}), refresh tokens ({{issued-token}}), and output token types other than a JWT access token (including ID-JAG) are OPTIONAL.  A leaner core is possible; whether to narrow to the mandatory combination alone is left to the working group.

## Open Questions for the Working Group {#open-questions}

1. **Authenticated handoff.**  What mechanism, if any, proves that the Initiator actively conveyed a particular assertion to the Delegate?  A related sub-question: does a `may_act` claim signed by the assertion's issuer represent sufficient authorization to record the Initiator as a nested prior actor in `act`, or does this document's strict requirement (Initiator-authenticated evidence) hold?

2. **Relationship exposure.**  Are the optional client-metadata views sufficient, or should an authenticated client be able to query its own CCDRs through a protected endpoint?

3. **Consent surface.**  Under which deployment policies is administrative authorization sufficient, and when must the delegation be shown to the End-User?

4. **Cross-organizational Delegates.**  How does the IdP recognize and namespace a Delegate controlled by another organization?

5. **Generic client relationships.**  If a broader client-relationships primitive emerges, how should the CCDR be represented within it?

6. **Multiple eligible Delegates per assertion.**  {{RFC8693}} defines `may_act` as a single JSON object, so an Identity Assertion can name only one eligible Delegate.  Is a per-assertion authorization for multiple Delegates needed, and if so, how should it be represented without changing {{RFC8693}} semantics?

7. **ID-JAG coordination path.**  Given the dependency recorded above, which mechanism should carry the audience-check exception: an extension hook added to ID-JAG, coordinated normative text in both drafts, or defining cross-client delegation within ID-JAG itself?


# Acknowledgments
{:numbered="false"}

This profile factors out patterns discussed in the Identity Assertion JWT Authorization Grant issue tracker, in particular the "Architectural Conflict in Gateway / Proxy Topology with Public Clients" discussion.  The enterprise deployment shape described in {{appendix-broker}} derives from an observation contributed to that discussion by GitHub user zekth, who also noted that the relaxation raises the value of a captured Identity Assertion.  The author thanks the participants in that discussion and the authors of OAuth 2.0 Token Exchange and the Identity Assertion JWT Authorization Grant, on whose work this profile builds.


# Document History
{:numbered="false"}

\[\[ To be removed from the final specification ]]

-00

* Initial revision, derived from the "Cross-Client Delegation for OAuth 2.0 Token Exchange" sketches discussed in ID-JAG issue #114.  Defines CCDR administration, client and authorization-server metadata, RFC 7523 actor credentials, `may_act` interaction, issued-token actor semantics, and an informative ID-JAG Enterprise Broker deployment pattern.
