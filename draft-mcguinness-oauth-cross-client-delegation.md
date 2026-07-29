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
    title: "Google Identity: Cross-client Identity"
    author:
      org: Google
    target: https://developers.google.com/identity/protocols/oauth2/cross-client-identity
...

--- abstract

This document defines the Cross-Client Delegation profile for OAuth 2.0 Token Exchange (RFC 8693).  The profile permits a confidential OAuth client (the Delegate) to obtain a token on behalf of an End-User whose Identity Assertion was issued to a different OAuth client (the Initiator), when the authorization server administers an explicit Cross-Client Delegation Relationship (CCDR) between the two client registrations.  The Delegate presents the Initiator's Identity Assertion as the subject token, establishes its own identity as the acting party with an actor token, and authenticates as itself.  The authorization server validates the administered relationship, enforces any `may_act` constraint carried by the assertion, evaluates current policy, and issues a token whose `act` claim identifies the Delegate as the current actor.


--- middle

# Introduction

Modern OAuth deployments increasingly encounter topologies where the OAuth client that authenticates the End-User is not the OAuth client that will present the resulting token to a downstream service.  Enterprise agent-plus-gateway topologies, mobile-app-plus-backend-for-frontend patterns, and Model Context Protocol (MCP) gateway deployments all share this shape: an initiating client (the Initiator) authenticates the End-User and holds an Identity Assertion audienced to itself, while a distinct confidential client (the Delegate) performs the downstream protocol steps on the End-User's behalf.

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

This profile applies when a confidential client acts for an End-User using an Identity Assertion issued to a different client registration, and the authorization server administers a trust relationship between those two registrations.  It composes with token-specific profiles such as ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}}, which may impose additional constraints on the requested token type, audience, or claims.  {{appendix-broker}} describes an informative deployment pattern applying this profile to ID-JAG in an enterprise gateway topology.

## Non-Goals {#non-goals}

The following are explicit non-goals of this profile:

CCDR enrollment protocol:
: How a Cross-Client Delegation Relationship is established at the authorization server is a deployment concern.  This profile defines the semantics of the relationship, not its provisioning protocol or user interface.

Cross-organizational trust establishment:
: Recognition of a Delegate administered by a different organization is out of scope; see {{security-cross-vendor}}.

Authenticated handoff:
: A mechanism proving that the Initiator actively conveyed a particular Identity Assertion to the Delegate is out of scope; see {{security-captured}}.

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
: An optional authorization expressed by an {{RFC8693}} `may_act` claim in an Identity Assertion.  It identifies an actor eligible to act on behalf of the subject of that assertion.  It narrows, and does not expand, the actors permitted by the CCDR and current authorization server policy.  See {{may-act}}.

Identity Assertion:
: A security token issued by the IdP that conveys claims about the End-User, identifies the Initiator as an intended audience or authorized party, and is suitable for use as `subject_token` in Token Exchange.  In this profile, an Identity Assertion is an OpenID Connect ID Token {{OpenID.Core}} or a SAML 2.0 assertion {{SAML2.Core}}.

Identity Provider (IdP):
: The authorization server that issued the Identity Assertion and performs the Token Exchange defined by this profile.  This document uses "IdP" consistent with {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

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

6. The IdP evaluates current policy over the tuple (End-User, Initiator, Delegate, requested audiences, resources, scope, authorization details).

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


# Cross-Client Delegation Relationship {#ccdr}

A Cross-Client Delegation Relationship (CCDR) is an eligibility relationship, administered at the authorization server, between an Initiator client registration and one or more Delegate client registrations.  The CCDR states that a client pairing is eligible for cross-client delegation under IdP policy.  It is an input to authorization policy, not an entitlement to any specific exchange; see {{security-freshness}}.

## Establishment and Administration {#ccdr-admin}

Establishing or modifying a CCDR requires authorization by an administrative principal at the IdP.  A client MUST NOT be permitted to establish or expand a CCDR merely by submitting client metadata values through dynamic client registration or registration management {{RFC7591}}.  This profile does not define the enrollment protocol or user interface for CCDR management; that is a deployment concern.

The authorization server MUST evaluate one consistent effective relationship for each (Initiator, Delegate) pair.  If the metadata views defined in {{client-metadata}} are exposed, they MUST reflect the same effective relationship.

The relationship MAY be maintained purely in IdP-side configuration.  This profile does not define public discovery of the relationship graph.

## Client Metadata {#client-metadata}

This profile defines two OPTIONAL client metadata parameters that expose read-only views of administered CCDRs in client registration responses or other authenticated client-metadata representations.  They are not client requests to establish a relationship.

For Initiators:

`authorized_delegates`:
: OPTIONAL.  A JSON array of `client_id` values eligible to act as Delegates for this client.

For Delegates:

`delegate_of`:
: OPTIONAL.  A JSON array of `client_id` values for Initiators on whose behalf this client is eligible to act.

Each array member MUST be a JSON string containing a `client_id` in the authorization server's issuer-scoped client namespace.  An omitted parameter conveys no information about whether relationships exist.  An empty array states that the authorization server currently exposes no relationships of that kind to the recipient.

These values are authorization-server-administered policy views.  Per {{ccdr-admin}}, they cannot be self-asserted through registration requests.  An authorization server that receives either parameter in a client registration or registration-management request MUST ignore the submitted value for authorization purposes or reject the request; it MUST NOT establish or expand a CCDR from the submitted value.


# Token Exchange Request {#request}

A Token Exchange request under this profile is a request to the token endpoint as defined in {{RFC8693, Section 2.1}}, with the following parameters:

`grant_type`:
: REQUIRED.  The value `urn:ietf:params:oauth:grant-type:token-exchange`, per {{RFC8693}}.

`subject_token`:
: REQUIRED.  The Initiator's Identity Assertion.

`subject_token_type`:
: REQUIRED.  A token type identifier appropriate for the Identity Assertion type, for example `urn:ietf:params:oauth:token-type:id_token` or `urn:ietf:params:oauth:token-type:saml2`.

`actor_token`:
: REQUIRED.  An {{RFC7523}} JWT client assertion identifying the Delegate as the acting party, as described in {{actor-token}}.  Note that `actor_token` is OPTIONAL in base {{RFC8693}}; this profile requires it.

`actor_token_type`:
: REQUIRED.  The value `urn:ietf:params:oauth:token-type:jwt`.

Other Token Exchange parameters (`audience`, `resource` {{RFC8707}}, `scope`, `authorization_details` {{RFC9396}}, and `requested_token_type`) are included as appropriate for the requested downstream token and its token profile.

The Delegate MUST authenticate the Token Exchange request using a client authentication method accepted by the IdP for confidential clients.  Requests from public clients are outside the scope of this profile.

The authenticated client and the actor are distinct protocol concepts.  This profile requires them to identify the same Delegate; see {{actor-token-requirements}}.

An IdP that supports this profile MUST support the combination of an OpenID Connect ID Token as `subject_token` and an {{RFC7523}} JWT client assertion as `actor_token`.  An IdP MAY additionally support SAML 2.0 Identity Assertions and actor-credential combinations defined by companion specifications.


# Actor Token {#actor-token}

## Requirements {#actor-token-requirements}

This document defines one interoperable actor credential for the base profile: an {{RFC7523}} JWT client assertion for the Delegate.  A companion specification MAY define additional direct actor-credential types.  Such a specification MUST define the credential's validation, actor-identity extraction, replay protection, and binding to the authenticated client; the JWT token-type identifier alone is not sufficient to select an unspecified JWT credential profile.

The actor token MUST:

*  identify the Delegate's `client_id` in both the `iss` and `sub` claims;
*  be verifiable by the IdP under {{RFC7523}};
*  include `iat`, `exp`, and `jti` claims and be subject to replay detection;
*  not contain an `act` claim, because it is a direct credential for the new actor rather than an existing delegation chain; and
*  resolve to the same `client_id` as the client authenticated for the Token Exchange request.

The IdP MUST reject a JWT access token, ID Token, ID-JAG, or other JWT profile presented as the base profile's `actor_token`, even if its claims happen to include the Delegate's `client_id`.  Each JWT position MUST be validated under the rules for that position; see {{security-confusion}}.

The IdP MUST confirm that the actor established by `actor_token` is the same Delegate identified by client authentication.  A profile that permits the actor to differ from the authenticated OAuth client is outside the scope of this document.

The IdP maps the validated Delegate `client_id` to the actor identifier (`act.iss`, `act.sub`) = (IdP issuer identifier, Delegate `client_id`).  The `iss` claim in an {{RFC7523}} client assertion is itself the Delegate's `client_id`; it is not copied into `act.iss`.

## JWT Client Assertion Actor Token {#actor-token-jwt}

When `actor_token` is a JWT client assertion {{RFC7523}}:

*  `actor_token_type` is `urn:ietf:params:oauth:token-type:jwt`;
*  the `iss` and `sub` claims identify the Delegate's `client_id`;
*  the `aud` claim identifies the IdP token endpoint or another audience accepted by the IdP under {{RFC7523}};
*  the JWT includes `iat`, `exp`, and `jti` claims (note that `iat` and `jti` are OPTIONAL in base {{RFC7523}}; this profile requires them);
*  the JWT is signed using a key registered for the Delegate; and
*  the IdP applies its {{RFC7523}} validation and replay-detection rules.

The same JWT value MAY be presented as both `client_assertion` and `actor_token`.  When it is, the IdP MUST treat the two parameter occurrences as one acceptance of one assertion for replay-detection purposes; it MUST NOT reject the second occurrence as a replay within the same request.  A later request using the same `jti` is a replay and MUST be rejected.  Alternatively, the Delegate MAY authenticate using another accepted method (for example, mutual TLS {{RFC8705}}) and present a separate JWT client assertion as its actor credential.

These actor-token validation, dual-use processing, and actor-identifier construction rules are designed to align with the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}.


# Per-Assertion Actor Authorization {#may-act}

A JWT Identity Assertion used as `subject_token` MAY include a `may_act` claim as defined in {{RFC8693, Section 4.4}}.  This document does not define an equivalent field for non-JWT Identity Assertions.

When present, `may_act`:

*  identifies a party eligible to become the actor for the subject of the Identity Assertion;
*  MUST contain `iss` and `sub` claims that identify the Delegate in the IdP's actor-identifier namespace;
*  MUST identify the same party established by `actor_token`; and
*  narrows the CCDR and current IdP policy rather than expanding them.

For example:

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

For the single-IdP deployment defined by this profile, `may_act.iss` MUST equal the IdP issuer identifier and `may_act.sub` MUST equal the Delegate's `client_id`, unless an applicable companion profile defines a different actor-identifier mapping.  The IdP MUST NOT authorize a different Delegate merely because that Delegate is eligible under the CCDR.  A `may_act` mismatch MUST cause the exchange to fail.

Absence of `may_act` does not by itself prohibit the exchange.  In that case, the IdP uses the CCDR and current policy to decide whether the authenticated Delegate may become the actor.

A `may_act` claim issued by the IdP represents authorization by the IdP for the named actor to act on behalf of the assertion subject.  It does not prove that the Initiator subsequently participated in, or authorized, a particular handoff of the assertion; see {{security-captured}}.


# Token Exchange Processing {#processing}

## Processing Rules {#processing-steps}

The IdP MUST apply the following processing to a Token Exchange request under this profile:

1. Validate the `subject_token` signature, issuer, temporal validity, and all other requirements applicable to its Identity Assertion type, except for the normal requirement that the assertion identify the requesting client as its audience.  This profile replaces that one check with steps 2, 5, and 6 below; it does not relax any other Identity Assertion validation.  An Identity Assertion containing an `act` claim is outside the base profile and MUST be rejected unless a companion profile defines validation and preservation of its existing delegation chain.

2. Determine exactly one Initiator using the assertion-type-specific rules and the IdP's issuer-scoped client-registration mapping:

   * For an ID Token whose `aud` claim is a string or a single-element array, that audience value identifies the Initiator.  If `azp` is present in this case, it MUST equal that audience value.  For an ID Token with multiple audience values, the `azp` claim MUST be present, MUST identify the Initiator, and MUST be one of the audience values.  The selected value MUST resolve to exactly one client registration at the IdP.

   * For a SAML 2.0 assertion, the IdP MUST validate all applicable `AudienceRestriction` conditions and use its configured mapping from the intended SAML service-provider audience to exactly one Initiator client registration.

   The IdP MUST reject an assertion when the Initiator cannot be determined unambiguously.

3. Validate `actor_token` according to its token type and determine the actor identity, per {{actor-token}}.

4. Confirm that the actor identity matches the client authenticated for the request (the Delegate).

5. Verify that the Initiator and Delegate registrations are currently enabled and that a CCDR currently authorizes the Delegate to act for the Initiator.

6. If the subject token contains `may_act`, verify that `may_act` identifies the same Delegate.  The IdP MUST NOT ignore a mismatch or use the CCDR to broaden the assertion's per-token authorization.

7. Evaluate current policy over the tuple (End-User, Initiator, Delegate, requested audiences, resources, scope, authorization details) according to IdP configuration.  The IdP MUST NOT treat successful End-User authentication or the existence of a CCDR as authorization for arbitrary target audiences, resources, scopes, or authorization details.

8. On success, issue the requested token according to {{issued-token}}.

## Errors {#errors}

If any step in {{processing-steps}} fails, the IdP MUST return an error response as specified in {{RFC6749, Section 5.2}} and {{RFC8693, Section 2.2.2}}:

*  Failures of client authentication MUST use the `invalid_client` error code per {{RFC6749}}.

*  An invalid `subject_token` or `actor_token`, a CCDR that does not authorize the (Initiator, Delegate) pair, a `may_act` mismatch, or any other policy-based rejection of the presented tokens MUST use the `invalid_request` error code, per {{RFC8693, Section 2.2.2}}.

*  A request whose `audience` or `resource` is unacceptable SHOULD use the `invalid_target` error code per {{RFC8693}}.

Error responses for actor-token validation and `may_act` or CCDR authorization failures SHOULD NOT reveal the IdP's relationship graph.  In particular, error codes and `error_description` values SHOULD NOT allow a client to distinguish "no CCDR exists for this pair" from other policy rejections, and SHOULD NOT enumerate eligible Delegates or Initiators.  See {{privacy}}.


# Issued Token {#issued-token}

Every token issued under this profile MUST make the accepted actor context available to the token consumer.  A JWT issued under this profile MUST contain:

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

Other claims are included according to the requested token type and applicable token profile.  When the IdP issues an opaque access token, its OAuth Token Introspection {{RFC7662}} response MUST include the `act` member with the same semantics, as permitted by {{RFC8693, Section 4.1}}, and the IdP and protected resource MUST have an arrangement that makes that actor context available for token processing.  The IdP MUST NOT issue a token under this profile when the selected token type and deployment cannot convey the current actor to the token consumer.

The Token Exchange response is constructed according to {{RFC8693, Section 2.2}} and the requested token's profile.  The IdP SHOULD NOT issue a refresh token in response to an exchange under this profile.  If it does, the IdP MUST bind the refresh token to the Delegate, retain the Initiator and CCDR context needed for later authorization, and re-evaluate the current status of the Delegate, Initiator, CCDR, and requested authorization whenever the refresh token is used.  Revoking or narrowing the CCDR MUST prevent the refresh token from obtaining authorization no longer permitted by the effective relationship.


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
    },
    {
      "subject_token_type":
        "urn:ietf:params:oauth:token-type:saml2",
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

This profile uses `may_act` without changing its {{RFC8693}} semantics.  `may_act` is an input authorization statement identifying an eligible actor; `actor_token` proves the proposed actor's identity; and `act` records the actor accepted into the output token.

The CCDR is an authorization-server-side eligibility rule that can be used when the subject token does not carry `may_act`.  When both are present, both must authorize the Delegate; see {{security-conjunctive}}.

## OpenID Connect Cross-Client Identity {#related-google}

Deployments MAY implement both OpenID Connect cross-client identity conventions ({{Google.CrossClient}}) and this profile.  This profile keeps the base Identity Assertion audience unchanged unless a deployment deliberately adds `may_act` as an extension.  It does not require an ID Token audience array.

## Identity Assertion JWT Authorization Grant (ID-JAG) {#related-idjag}

This profile can be composed with ID-JAG Token Exchange {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  When the requested token is an ID-JAG (`requested_token_type` of `urn:ietf:params:oauth:token-type:id-jag`), the issued ID-JAG carries the Delegate as the current actor, and the ID-JAG's base audience and client-binding rules continue to apply.  A deployment jointly conforming to ID-JAG and this profile replaces ID-JAG's default Identity Assertion audience-equals-requesting-client check only with the processing rules in {{processing}}.  No other ID-JAG validation rule is relaxed.

The Initiator is not automatically a nested actor.  Recording it as a prior actor requires an additional authenticated handoff mechanism as described in {{issued-token}}.

{{appendix-broker}} describes the Enterprise Broker deployment pattern, an informative application of this profile to ID-JAG.

## OpenID Connect Key Binding {#related-key-binding}

The mechanism specified by OpenID Connect Key Binding {{OpenID.KeyBinding}} can bind an Identity Assertion to the Initiator's key, but that binding alone is not resolved by this profile: without a presenter-continuation or key-transition mechanism, the Delegate cannot demonstrate proof of possession for such an assertion.  Deployments requiring both key binding and cross-client delegation need to define the presenter-transition mechanism via a companion profile.  This remains an open problem across the delegation space and is not unique to this profile.

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

## Delegate Compromise {#security-delegate-compromise}

Compromise of a Delegate's credentials may permit obtaining tokens for every Initiator linked to that Delegate.  Relationships SHOULD be narrowly scoped, and Delegate credentials SHOULD be rotated with the discipline applied to sensitive service credentials.

Where Identity Assertions include `may_act`, compromise impact is limited to assertions that name the compromised Delegate, but existing assertions remain usable until they expire, are revoked, or are rejected by current IdP policy.

## Exchange-Time Policy Freshness {#security-freshness}

The IdP MUST evaluate policy at exchange time.  The CCDR and `may_act` establish eligibility, not entitlement to any specific exchange.  Audience, resource, scope, and `authorization_details` remain subject to policy.

The IdP MUST re-check the current status of the Initiator, the Delegate, and their CCDR at each mint.  Prior actors in any nested `act` chain are informational under {{RFC8693}} and MUST NOT be used for access-control decisions.

## Transport of the Identity Assertion {#security-transport}

This profile does not define how the Initiator conveys the Identity Assertion to the Delegate.  Transport SHOULD provide confidentiality, integrity, and session correlation.

Deployments requiring proof that the Initiator actively authorized a particular handoff need an additional Initiator-authenticated mechanism.  Such a mechanism MAY allow the IdP to record the Initiator as a nested prior actor per {{issued-token}}.  {{appendix-handoff}} describes the correlation controls one deployment pattern applies.

## Cross-Organizational Delegates {#security-cross-vendor}

The CCDR is administered by the IdP that validates the Identity Assertion.  Cross-vendor gateway deployments require the IdP to recognize the Delegate and its actor credential.  This profile does not define how trust is established across organizations.

## Consent Semantics {#security-consent}

Neither a CCDR nor `may_act` by itself establishes that the End-User understood or approved disclosure to the Delegate.  Deployments MUST have an applicable consent, enterprise-policy, or other authorization basis and SHOULD reflect delegation in user experience where appropriate.

## Revocation and Previously Issued Tokens {#security-revocation}

Revoking an Initiator, Delegate, or CCDR MUST prevent new tokens from being minted through this profile.  A CCDR change does not, by itself, invalidate tokens already issued.  Deployments that require immediate invalidation need an operational revocation mechanism, such as token revocation, introspection-backed tokens, or another deployment-specific control.  Otherwise, short `exp` values on issued tokens bound the residual authorization window.

The same bounded window applies to `may_act`.  A `may_act` claim is evaluated when the exchange occurs.  If the named Delegate or its CCDR is later revoked, the status of tokens already minted depends on the deployment's revocation mechanism and the issued token's lifetime.

## Token and Key Confusion {#security-confusion}

Identity Assertions, actor tokens, and client authentication assertions may all be JWTs signed within the same deployment.  The IdP MUST validate each token under the rules for the position in which it is presented, including explicit typing and audience restrictions where available, so that a token issued for one function cannot be replayed in another (for example, an ID Token presented as `actor_token`, or a client assertion presented as `subject_token`).  The recommendations of {{RFC8725}} apply.

When the same client assertion is used in both `client_assertion` and `actor_token`, accepting it once for the request does not authorize its use in any later request.  Replay state MUST be shared across both processing paths so that changing only the parameter position cannot bypass replay detection.


# Privacy Considerations {#privacy}

The CCDR graph reveals which clients are administratively linked, which may itself be sensitive (for example, disclosing an organization's internal gateway topology or vendor relationships).  This profile does not define public discovery of the relationship graph, and {{errors}} requires that error responses not enumerate it.  The client metadata views defined in {{client-metadata}}, when exposed at all, SHOULD be restricted to the affected clients and administrative principals.

Issuing a token for a new audience on the basis of an assertion issued to the Initiator can correlate a user's identity across clients and audiences.  IdPs that use pairwise or audience-scoped subject identifiers SHOULD apply the subject identifier policy of the issued token's audience rather than copying an Initiator-scoped identifier; composition profiles such as ID-JAG define audience-appropriate subject resolution.

The `act` claim in the issued token discloses the Delegate's identity to downstream token consumers.  This disclosure is deliberate: making the acting party explicit is a design goal of this profile.


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
*  Change Controller: IETF
*  Specification Document: {{client-metadata}} of this document

and:

*  Client Metadata Name: `delegate_of`
*  Client Metadata Description: Array of client identifiers for Initiators on whose behalf this client is eligible to act under the Cross-Client Delegation profile
*  Change Controller: IETF
*  Specification Document: {{client-metadata}} of this document

## OAuth Authorization Server Metadata Registration {#iana-as-metadata}

This document requests IANA to register the following value in the "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

*  Metadata Name: `cross_client_delegation_token_types_supported`
*  Metadata Description: JSON array of objects describing the subject token type and actor token type combinations accepted under the Cross-Client Delegation profile
*  Change Controller: IETF
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

4. The IdP validates the CCDR, confirms `may_act` if present, evaluates current policy, resolves the Broker's client identifier at the target Resource Authorization Server, and issues an ID-JAG with `act` identifying the Broker.

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

Within the single-IdP scope of this pattern, the IdP re-checks the current status of the Managed Client, the Broker, and their CCDR at each mint, per {{security-revocation}}.  Short `exp` values on issued tokens bound the residual window when the deployment does not provide immediate revocation of already-issued tokens.

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

*  Base ID-JAG's audience check applies unchanged where no CCDR exists.  This profile defines the CCDR-authorized exception to that check; this informative deployment pattern does not independently override the base requirement.

*  The issued ID-JAG's structure, target-specific subject resolution, tenant context, and downstream presentation to the Resource Authorization Server follow base ID-JAG.

*  The ID-JAG `client_id` identifies the Broker's client registration at the target Resource Authorization Server, because the Broker will authenticate under that registration when redeeming the ID-JAG.

*  The Broker's `client_id` at the IdP, the Broker's `client_id` at the Resource Authorization Server, and the Managed Client's `client_id` at the IdP may all differ.  The IdP needs a trusted mapping from the authenticated Broker to the target-side Broker registration.

*  The ID-JAG `sub` and related subject or tenant claims identify the End-User in the namespace expected by the target Resource Authorization Server, rather than blindly copying an Initiator-scoped pairwise subject; see {{privacy}}.

*  When the ID-JAG is sender-constrained, its `cnf` claim binds the ID-JAG to a Broker-controlled key that the Broker proves when redeeming it.

The CCDR-authorized relaxation applies only to the audience check on the Identity Assertion at Token Exchange time, and only when the processing rules of {{processing}} are followed.

Client attestation ({{I-D.ietf-oauth-attestation-based-client-auth}}) can authenticate a particular Managed Client or Broker instance and prove possession of an instance key.  It complements rather than replaces the CCDR: attestation establishes instance authenticity, while the CCDR authorizes the cross-client relationship.


# Open Items for Working Group Discussion {#open-items}

1. **Per-assertion authorization.**  Should profiles built on OpenID Connect require `may_act`, or retain the CCDR-only mode for unchanged ID Tokens?

2. **Authenticated handoff.**  What mechanism, if any, proves that the Initiator actively conveyed a particular assertion to the Delegate?  A related sub-question: does a `may_act` claim signed by the assertion's issuer represent sufficient authorization to record the Initiator as a nested prior actor in `act`, or does this document's strict requirement (Initiator-authenticated evidence) hold?

3. **Relationship exposure.**  Are the optional client-metadata views sufficient, or should an authenticated client be able to query its own CCDRs through a protected endpoint?

4. **Actor credential types.**  Should the base profile remain limited to an {{RFC7523}} client assertion, with workload credentials and access tokens defined by companion profiles?

5. **Consent surface.**  Under which deployment policies is administrative authorization sufficient, and when must the delegation be shown to the End-User?

6. **Cross-organizational Delegates.**  How does the IdP recognize and namespace a Delegate controlled by another organization?

7. **Generic client relationships.**  If a broader client-relationships primitive emerges, how should the CCDR be represented within it?


# Acknowledgments
{:numbered="false"}

This profile factors out patterns discussed in the Identity Assertion JWT Authorization Grant issue tracker, in particular the "Architectural Conflict in Gateway / Proxy Topology with Public Clients" discussion.  The enterprise deployment shape described in {{appendix-broker}} derives from an observation contributed to that discussion by GitHub user zekth, who also noted that the relaxation raises the value of a captured Identity Assertion.  The author thanks the participants in that discussion and the authors of OAuth 2.0 Token Exchange and the Identity Assertion JWT Authorization Grant, on whose work this profile builds.


# Document History
{:numbered="false"}

\[\[ To be removed from the final specification ]]

-00

* Initial revision, derived from the "Cross-Client Delegation for OAuth 2.0 Token Exchange" sketches discussed in ID-JAG issue #114.  Defines CCDR administration, client and authorization-server metadata, RFC 7523 actor credentials, `may_act` interaction, issued-token actor semantics, and an informative ID-JAG Enterprise Broker deployment pattern.
