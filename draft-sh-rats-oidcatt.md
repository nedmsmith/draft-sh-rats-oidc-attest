---
title: Attestation in OpenID-Connect
abbrev: OIDCATT
category: info
docname: draft-sh-rats-oidcatt-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Remote ATtestation ProcedureS"
keyword:
 - attestation
venue:
  group: "Remote ATtestation ProcedureS"
  type: "Working Group"
  mail: "rats@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/rats/"
  github: "nedmsmith/draft-sh-rats-oidc-attest"
  latest: "https://nedmsmith.github.io/draft-sh-rats-oidc-attest/draft-sh-rats-oidcatt.html"

author:
 -  ins: N. Smith
    fullname: Ned Smith
    organization: Intel Corporation
    country: United States of America
    email: ned.smith@intel.com
 -  ins: T. Hardjono
    fullname: Thomas Hardjono
    organization: Massachusetts Institute of Technology
    country: United States of America
    email: hardjono@mit.edu

normative:
  OCC2014:
    -: oidc
    author:
      ins: N. Sakimura
    title: "OpenID Connect Core 1.0 incorporating errata set 1"
    date: 2014-11

informative:
  RFC9334: rats-arch
  RFC6749: oauth2
  I-D.ftbs-rats-msg-wrap: msg-wrap

entity:
  SELF: "RFCthis"

--- abstract

This document defines message flows and extensions to OpenID-Connect (OIDC) messages that support attestation.
Attestation Evidence and Attestation Results is accessed via appropriate APIs that presumably require authorization using
OAuth 2.0 access tokens.
A common use case for OIDC is retrieval of user identity information authorized by an OIDC identity token. The Relying Party may
require Attestation Results that describes the trust properties of the UserInfo Endpoint. Trust properties may
be  a condition of accepting the user identity information.

--- middle


# Introduction {#intro}

This document defines attestation conceptual message flows that extend OpenID-Connect (OIDC) messages, see {{OCC2014}}.
Attestation Evidence and Attestation Results are RATS conceptual messages, see {{-rats-arch}} and {{-msg-wrap}}, that are obtained
via appropriate APIs conditional on OAuth 2.0 access tokens {{-oauth2}}.
A common use case for OIDC is retrieval of user identity information authorized by an OIDC identity token.
The Relying Party may require Attestation Results regarding the UserInfo Endpoint as a condition of accepting the user identity
information.


# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

## Terminology {#terminology}

This specification uses role names as defined by Remote ATtestation procedureS (RATS), {{-rats-arch}} and OpenID Connect (OIDC),
{{-oidc}}. If role names conflict, (e.g., Relying Party), then the RATS role is qualified by prepending ‘RATS’ or ‘R’.
For example, the RATS Relying Party is disambiguated as ‘RRP’.

A summary of roles used in this specification is provided here for convenience.

RATS roles are as follows:

* Attester (RA) – an endpoint that produces attestation Evidence.
* Reference Value Provider (RVP) – an endpoint that produces Reference Values used to appraise Evidence.
* Endorser (RE) – an endpoint that produces Endorsements used to assess the trustworthiness of the Attester’s attestation capability.
* Verifier (RV) – an endpoint that consumes Evidence, Endorsements and Reference Values and produces Attestation Results.
* Relying Party (RRP) – an endpoint that consumes Attestation Results and applies them to an application or usage context.

OIDC roles are as follows:

* OpenID Provider (OP) – an endpoint that authenticates an End User, obtains authorization, and responds with an ID Token
and (usually) an Access Token.
(a.k.a., an OAuth 2.0 Authorization Server, {{-oauth2}}).
* Relying Party (RP) / Client – an endpoint that sends a request to an OpenID Provider.
* UserInfo Endpoint (UE) – an API endpoint that receives an Access Token and sends Claims about an End User.
* User Agent (UA) - a browser or other code that may interact with an End User or access user resources.
* End User (EU) – a human participant.

OAuth 2.0 roles are as follows:

* Resource Server (RS) – a service that controls a resource.
* Client - synonymous with User Agent.
* Resource Owner (RO) - synonymous with End User.

# OIDC Sequence with Attestation {#oidc-sequence}

OpenID-Connect (OIDC) {{-oidc}} defines user authentication protocol and messages based on OAuth 2.0 {{-oauth2}} authorization
protocol and messages. This section shows an example OIDC protocol sequence with extensions for attestation Evidence and
Attestation Results (AR) exchanges.
The protocol is divided into two phases. A setup phase and an operational phase. The setup phase models protocol initialization
steps that are anticipated but often ignored. An understanding of the initialization steps may be helpful when determining how
various steps in the operational phase are authorized.

## Protocol Endpoints {#protocol-endpoints}

The example protocol message exchange involves four main endpoints:

1. Device – a RATS Attester that consists of two sub entities:

  * A UserInfo Endpoint (UE) that supplies user information for OIDC authentication, and

  * A lead Attesting Environment, that collects device attestation Evidence. When using RATS terminology, the device may be
  referred to as the RATS Attester (RA). The RA is technically an OAuth 2.0 Resource Server (RS) that performs attestation
  Evidence collection. The Attester device may consist of multiple components that typically include a root of trust,
  boot code, system software and the browser. The lead Attesting Environment typically seeks to collect Evidence that
  describes all the components, from the root of trust to the UA, that may influence endpoint behavior.

{:start="2"}
1. User Agent (UA) – a native application that can engage the End User directly.

1. Relying Party (RP) – an endpoint that seeks UserInfo used to replay user authentication responses for OIDC exchanges.
The RP may rely on the OP to appraise attestation results on its behalf as a RATS Relying Party (RRP). As such the RP may be the RATS AR Owner. Alternatively, the AR may directly process Attestation Results.

1. OpenID Provider (OP) – an Authorization Server (AS) that implements OIDC such that receipt of an OpenID 'code' from the UA results in the issuance of an OpenID token, 'id-token'. The OP may implement the RATS Relying Party (RRP) role such that issuance of the OpenID token is conditional on suitable Attestation Results. The RP may take on the role of AR Owner to ensure the OP evaluates attestation results that align with its risk requirements.

1. Verifier (RV) – a RATS attestation Verifier that processes device Evidence.
If the Verifier is combined with the OP, the Verifier becomes an additional processing
stage within the OP.

## Setup Phase {#setup-phase}

The setup phase creates the various identity (‘id-token’) and access (‘access-token’) credentials that are used during
the operational phase to authorize the exchange of the various OIDC protocol messages.

### Identity Token Creation {#identity-token-creation}

In this example, there is a single end user, “Alice”, that creates an identity token ‘id-token’. The Native App signals
the UE when it is appropriate to create the id-token. For example, the 'id-token' contains: { "sub": "A21CE", "name": "Alice" }.

### Attestation Access Token Creation {#access-token-creation}

The RA exposes an attestation API that invokes the attestation capabilities of the Attester device. An access token,
‘access-token-attest’, is needed to authorize use of the attestation API.

### UserInfo Access Token Creation {#userinfo-token-creation}

The UE exposes a UserInfo API that invokes the user information capabilities of the User Agent. An access token,
‘access-token-uinfo’, is needed to authorize use of the UserInfo API.

### Evidence Appraisal Access Token Creation {#evidence-token-creation}

The RV exposes an API for appraising Evidence. An access token, ‘access-token-appraisal’, is needed to authorize
use of the appraisal API.

### Register Device {#register-device}

The Attester device is registered with the RP client in anticipation of subsequent operational flows.
The registration process is out of scope for this document.

### Attestation Evidence Payload {#attestation-evidence-payload}

The RA produces an Evidence payload that is conveyed to the RV. Some OIDC messages are extended to carry Evidence.

### Attestation Results Payload {#attestation-results-payload}

The RV produces an Attestation Results payload that is conveyed to the RP. Some OIDC messages are extended to carry
Attestation Results.

## Operational Phase {#operational-phase}

The operational phase protocol builds on the abstract OIDC protocol in {{-oidc}}. The five OIDC steps are described
here for convenience and attestation related steps are described as sub-steps.

### AuthN Request {#authn-req}

The RP sends an AuthN request to the OP containing the RP’s identity ‘client-id’. Additionally, the RP includes an
attestation scope, e.g., ‘scope=”device-attest”’ that instructs the OP to obtain an attestation from the UE device.
The trigger for sending the AuthN request is out of scope for this document.

~~~~ aasvg
{::include authn-req.txt}
~~~~
{: #authn-request-flow artwork-align="center" title="AuthN Request Flow"}

#### AuthN Request Payload {#authn-req-payload}

The following non-normative AuthN Request payload example identifies the OP server location, the RP client identity,
and an attestation scope:

~~~
AuthN_Req = {
    "location": https://op.example.com/authn"
    "client_id": "s6BhdRkqt3",
    "scope": "device-attest"
}
~~~

### Forwarded AuthN Request {#forwarded-authn-req}

The OP forwards the original AuthN request to the UE. The attestation scope instructs the UE to configure the device for attestation.
For example, an internal interface between the UE and RA (a.k.a., Resource Server) might be used to configure a ‘client-id’ nonce that the RA Attesting Environment includes with attestation Evidence.
The UE normally returns a payload containing the ‘client-id’, response type (i.e., resp-type = “code”), and the
authentication result (i.e., authn-proof). However, a successful response is returned on condition of successful
configuration of the attestation scope.
The End User may consent to the disclosure of attestation Evidence using the 'prompt' parameter. An "attestation-consent"
authorization string is supplied as one of the 'prompt' parameters.
* *attestation-consent - The OP (a.k.a., Authorization Server) SHOULD prompt the End User for consent before returning
information to the RP (a.k.a., Client). If it cannot obtain attestation consent, it MUST return an error,
typically 'consent_required'.

~~~~ aasvg
{::include forwarded-authn.txt}
~~~~
{: #forwarded-authn-flow artwork-align="center" title="Forwarded AuthN Request-Response Flow"}

#### Forwarded AuthN Request and Response Payloads {#forwarded-authn-payloads}

The forwarded AuthN Request is identical to AuthN Request.
The forwarded AuthN Response payload example identifies the originating RP, scope, response type, and authentication proof:

~~~
AuthN_Rsp = {
    "client_id": "s6BhdRkqt3",
    "scope": "device-attest",
    "resp_type": "code",
    "authn_proof": "<tbd>"
}
~~~

### User Authorization of AuthN, AuthZ, and Attest {#user-authorization}

The OP authenticates the End User (e.g., “Alice”) and obtains authorization. Normally, authorization is limited to
an authentication or authorization context as defined by the legacy OIDC protocol.
But when attestation scope is used, the End User may wish to approve attestation. Attestation normally reveals
Evidence details about the UE device. If those details contain privacy sensitive information,
the End User may wish to opt-out of attestation.
If the Authentication Request contains the 'prompt' parameter with the value 'attestation-consent',
the OP MUST inform the End User that attestation Evidence is about to be disclosed to the RP (a.k.a., Client),
and the End User MUST be given the option to withhold Evidence.

~~~~ aasvg
{::include user-auth.txt}
~~~~
{: #user-auth-flow artwork-align="center" title="End User Authentication Flow"}

#### End User Authorization Payload {#user-authorization-payload}

TODO add example

### Attestation Request and Response {#attestation-req-rsp}

If the End User doesn’t opt-out of attestation, the OP requests attestation Evidence from the RA (as a Resource Server).
The OP sends the ‘access-token-attest’ and ‘id-token = “Alice”’ tokens to the RA. The RA collects Evidence according to
the configured attestation scope. For example, if a ‘client-id’ specific nonce was configured, the nonce is included with Evidence.
The Evidence is returned to the OP through the UE, which normally returns the ‘client-id’, ‘access-token’, and ‘id-token’.

~~~~ aasvg
{::include attest-req-rsp.txt}
~~~~
{: #attest-req-rsp-flow artwork-align="center" title="Attestation Request-Response Flow"}

#### Attestation Request and Response Payloads {#attestation-req-rsp-payloads}

The Attestation Request and Response payload example contains an access_token that authorizes use of the
attestation API of the RA and an id_token that identifies the End User.

~~~
access_token = {
    "iss": "https://jwt-op.example.com",
    "sub": "https://jwt-ra.example.com/24400320",
    "aud": "https://jwt-rp.example.com/s6BhdRkqt3",
    "nbf": 1300815780,
    "exp": 1300819380,
    "claims.example.com/attest-api": true
}
~~~

~~~
id_token = {
    "iss": "https://jwt-op.example.com",
    "sub": "https://jwt-ra.example.com/24400320",
    "aud": "https://jwt-rp.example.com/s6BhdRkqt3",
    "nbf": 1300815780,
    "exp": 1300819380,
    "name": "Alice"
}
~~~

The response payload contains an Evidence value as described by a conceptual message wrapper {{-msg-wrap}}.

~~~
evidence_cmw = [
    "application/eat+jwt",
    "<base64-string containing a JWT>"
]
~~~

### Appraisal Request and Response {#appraisal-req-rsp}

The OP requests appraisal of Evidence by sending the ‘access-token-appraisal’ token and Evidence to the RV.
The token authorizes use of the appraisal API, which when appraisal completes, supplies Attestation Results.
The verification response contains the Attestation Results and ‘access-token’, that the RV sends to the OP.

~~~~ aasvg
{::include appraisal-req-rsp.txt}
~~~~
{: #appraisal-req-rsp-flow artwork-align="center" title="Appraisal Request-Response Flow"}

#### Appraisal Request and Response Payloads {#appraisal-req-rsp-payloads}

The Appraisal Request payload example contains an access_token that authorizes use of the appraisal API of the RV
and the Evidence to be appraised.

~~~
access_token = {
    "iss": "https://jwt-op.example.com",
    "sub": "https://jwt-rv.example.com",
    "aud": "https://jwt-rp.example.com/s6BhdRkqt3",
    "nbf": 1300815780,
    "exp": 1300819380,
    "claims.example.com/appraisal-api": true
}
~~~

~~~
evidence_cmw = [
    "application/eat+jwt",
    "<base64-string containing a JWT>"
]
~~~

The response payload contains an Attestation Results value as described by a conceptual message wrapper {{-msg-wrap}}.

~~~
attestation_result_cmw = [
    "application/eat+jwt",
    "<base64-string containing a JWT>"
]
~~~

### AuthN Response {#authn-rsp}

The OP sends ‘client-id’, ‘id-token = “Alice”’, ‘access-token-uinfo’, and the Attestation Results to the RP.
The RP processes the Attestation Results to determine if the UE device is trustworthy.
Presumably, if the UE isn’t trustworthy, the protocol is terminated.

~~~~ aasvg
{::include authn-rsp.txt}
~~~~
{: #authn-rsp-flow artwork-align="center" title="AuthN Response Flow"}

#### AuthN Response Payload {#authn-rsp-payload}

TODO add example

### UserInfo Request and Response {#user-info-req-rsp}

The UserInfo request is initiated by the RP, who sends ‘client-id’, ‘id-token = “Alice”’,
and ‘access-token-uinfo’ to the UE to collect user identity information.
The UserInfo response is initiated by the UE, who sends ‘client-id’, ‘id-token = “Alice”’,
‘access-token-uinfo’, and the UserInfo payload to the RP to process user claims and complete the OIDC protocol.

~~~~ aasvg
{::include userinfo-flow.txt}
~~~~
{: #user-info-flow artwork-align="center" title="UserInfo Request-Response Flow"}

#### UserInfo Request and Response Payloads {#user-info-req-rsp-payloads}

TODO add example

# Security Considerations {#security-considerations}

TODO Security


# IANA Considerations {#iana-considerations}

This document has no IANA actions.


--- back

# Acknowledgments {#acknowledgements}
{:numbered="false"}

The authors would like to thank the following people for their input:

* Jay Chetty - for review feedback.
