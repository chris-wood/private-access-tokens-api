# Private Access Tokens API Explainer

This document is an explainer for a potential future web platform API that for issuing and redeeming [Privacy Pass](https://datatracker.ietf.org/doc/draft-ietf-privacypass-architecture/) tokens.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Motivation](#motivation)
- [Overview](#overview)
- [Potential API](#potential-api)
  - [Private Token Issuance](#trust-token-issuance)
  - [Private Token Redemption](#trust-token-redemption)
  - [Extension: Public Metadata](#extension-public-metadata)
- [Privacy Considerations](#privacy-considerations)
  - [Cryptographic Property: Unlinkability](#cryptographic-property-unlinkability)
    - [Key Consistency](#key-consistency)
    - [Potential Attack: Side Channel Fingerprinting](#potential-attack-side-channel-fingerprinting)
  - [Cross-site Information Transfer](#cross-site-information-transfer)
    - [Mitigation: Dynamic Issuance / Redemption Limits](#mitigation-dynamic-issuance--redemption-limits)
    - [Mitigation: Allowed/Blocked Issuer Lists](#mitigation-allowedblocked-issuer-lists)
    - [Mitigation: Per-Site Issuer Limits](#mitigation-per-site-issuer-limits)
  - [First Party Tracking Potential](#first-party-tracking-potential)
- [Security Considerations](#security-considerations)
  - [Trust Token Exhaustion](#trust-token-exhaustion)
  - [Double-Spend Prevention](#double-spend-prevention)
- [Future Extensions](#future-extensions)
  - [Publicly Verifiable Tokens](#publicly-verifiable-tokens)
  - [Request mechanism not based on `fetch()`](#request-mechanism-not-based-on-fetch)
  - [Optimizing redemption RTT](#optimizing-redemption-rtt)
- [Appendix](#appendix)
  - [Sample API Usage](#sample-api-usage)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Motivation

[Privacy Pass](https://datatracker.ietf.org/doc/draft-ietf-privacypass-architecture/) is an architecture for enabling authorization based on anonymous tokens. It consists of two protocols, called issuance and redemption. The issuance protocol is used to obtain tokens for a given context, and the redemption protocol is used to present a token for authorization in said context. 

Both protocols are built on HTTP. However, there does not yet exist a standard API for driving these protocols from web contexts. This explainer document fills that void by describing an API for issuing and redeeming Privacy Pass tokens, referred to as the Private Access Tokens (PAT) API. 

## Potential API

The Private Access Tokens (PAT) API allows first parties to drive token issuance and redemption directly. The issuance protocol takes as input a token type, issuer name and public key, optional origin info, and redemption context, and produces as output a boolean indicating if issuance succeeded. The redemption protocol takes as input a base64url-encoded token and outputs a boolean indicating success or failure. We describe these APIs in more detail in the following subsections.

### Issuance API

When an origin.example context wants to provide tokens to a user (i.e. when the user is trusted), they can use a new Fetch API with the privateToken parameter:

```
fetch('<issuer>/private-token', {
  privateToken: {
    type: <token type>,
    issuer: <issuer>,
    issuer-key: <public key>,
    redemption-context: <redemption context>,
  }
}).then(...)
```

If the client has additional origin information they want to bind in the token, they can use the Fetch API with the following amended privateToken parameter:

```
fetch('<issuer>/private-token-issue', {
  privateToken: {
    type: <token type>,
    issuer: <issuer>,
    issuer-key: <public key>,
    privateTokenContext: {
      redemption-context: <redemption context>,
      origin-info: <origin info>,
    },
  }
}).then(...)
```

Setting the origin info to the empty string (or nil) is equivalent to requesting a cross-Origin token.

In any case, this API will invoke the [Privacy Pass](https://privacypass.github.io) issuance protocol:

*   Generate a nonce and produce a [TokenRequest](https://datatracker.ietf.org/doc/html/draft-ietf-privacypass-protocol-06#section-6.1) structure based on the token type.
*   Send a POST to the client's configured attestation endpoint.

When a response comes back the result is [finalized](https://datatracker.ietf.org/doc/html/draft-ietf-privacypass-protocol-06#section-6.3) and a base64-url encoded Token is produced and stored for the given origin context (origin.example). The token is bound to the origin for which the issuance request was made. In particular, the token cannot be redeemed across other origins. 

Raw tokens are never accessible to JavaScript. 

### Redemption API

When the user is browsing another site (publisher.example), that site (or origin.example embedded on that site) can optionally redeem origin.example to authorize user actions. One way to do this would be via new APIs:

```
document.hasPrivateToken(
  privateTokenContext: {
    redemption-context: <redemption context>,
    origin-info: <origin info>,
  }
)
```

This returns whether there are any valid tokens for a particular context so that the publisher can decide whether to attempt a token redemption.

```
fetch('<issuer>/trust-token-redeem', {
  privateTokenContext: {
    redemption-context: <redemption context>,
    origin-info: <origin info>,
  }
}).then(...)
```

If there are no tokens available for the given context, the returned promise rejects with an error. Otherwise, it invokes the redemption protocol against the designated origin, with the token attached in the `Authorization` header. The origin can consume the token and act based on the result or return an error. 

### Extension: iframe Activation

Some resources requests are performed via iframes or other non-Fetch-based methods. One extension to support such use cases would be the addition of a `privateToken` attribute to iframes that includes the parameters specified in the Fetch API. This would allow an RR to be sent with an iframe by setting an attribute of `privateToken="{issuer:<issuer>,...}"`.

## Privacy Considerations

See the [Privacy Pass](https://datatracker.ietf.org/doc/draft-ietf-privacypass-architecture/) protocol for privacy considerations. This section describes some additional considerations specific to these APIs.

### Potential Attack: Side Channel Fingerprinting

If the origin is able to use network-level fingerprinting or other side-channels to associate a browser at redemption time with the same browser at token issuance time, privacy is lost. Importantly, the API itself has not revealed any new information, since sites could use the same technique by issuing GET requests to the origin in the two separate contexts anyway.

### First Party Tracking Potential

Cached tokens have a similar tracking potential to first party cookies. Therefore these should be clearable by browser’s existing Clear Site Data functionality.

## Security Considerations

See the [Privacy Pass](https://datatracker.ietf.org/doc/draft-ietf-privacypass-architecture/) protocol for security considerations. This section describes some additional considerations specific to these APIs.

### Trust Token Exhaustion

The goal of a token exhaustion attack is to deplete a legitimate user's supply of tokens for a given issuer, so that user is less valuable to sites who depend on the issuer’s trust tokens.

There are a number of mitigations against this attack:

*   Issuers issue many tokens at once, so clients have a large supply of tokens.
*   Browsers will only ever redeem one token per top-level page view, so it will take many page views to deplete the full supply.
*   The browser will cache RRs per-origin and only refresh them when an issuer iframe opts-in, so malicious origins won't deplete many tokens. The “freshness” of the RR becomes an additional trust signal.
*   Browsers may choose to limit redemptions on a time-based schedule and return cached tokens if available.

When an origin detects that another origin is attacking its token supply, it can fail redemption (before the token is revealed) based on the referring origin, and prevent browsers from spending tokens there.


### Double-Spend Prevention

Origins can verify that each token is seen only once based on the redemption context. Origins that share redemption contexts must therefore share double spend state to prevent double spend. 


## Future Extensions

A possible enhancement would be to allow for sending Redemption Records (and signing requests using the trust keypair) for requests sent outside of `fetch()`, e.g. on top-level and iframe navigation requests. This would allow for the use of the API by entities that aren't running JavaScript directly on the page, or that want some level of trust before returning a main response.


### Optimizing redemption RTT

If the publisher can configure issuers in response headers (or otherwise early in the page load), then they could invoke a redemption in parallel with the page loading, before the relevant `fetch()` calls.

### Non-web sources of tokens

Token issuance could be expanded to other entities (the operating system, or native applications) capable of making an informed decision about whether to grant tokens. Naturally, this would need to take into consideration different systems' security models in order for these tokens to maintain their meaning. (For instance, on some platforms, malicious applications might routinely have similar privileges to the operating system itself, which would at best reduce the signal-to-noise ratio of tokens created on those operating systems.)

## Appendix


### Same-Origin API Usage Example

```
coolwebsite.example - Original top-level site
```

1.  User visits `coolwebsite.example`.
1.  `coolwebsite.example` verifies the user is a human, and calls `fetch('coolwebsite.example/trust-token-issue', {privateToken: {...}})` with no origin info. The browser stores the trust tokens associated with `coolwebsite.example`.
1.  Sometime later, the user goes back to `coolwebsite.example`.
1.  `coolwebsite.example` wants to know if the user has previously been authorized by calling `fetch('coolwebsite.example/trust-token-redeem', {privateToken: {...}})` with no origin info. If a token exists, the browser uses it, otherwise the promise returned fails.

### Cross-Origin API Usage Example

```
original.example - Original top-level site that issues tokens
publisher.example - Future top-level site that redeems tokens
```

1.  User visits `original.example`.
1.  `original.example` verifies the user is a human, and calls `fetch('original.example/trust-token-issue', {privateToken: {...}})` with origin info consisting of "original.example,publisher.example". The browser stores the trust tokens associated with this shared context.
1.  Sometime later, the user goes to `publisher.example`.
1.  `publisher.example` wants to know if the user has previously been authorized by original.example by calling `fetch('publisher.example/trust-token-redeem', {privateToken: {...}})` with origin info equal to "original.example,publisher.example". If a token exists, the browser uses it, otherwise the promise returned fails.

### Embedded iframe Cross-Origin API Usage Example

```
areyouhuman.example - Embedded origin that verifies humanity
original.example - Original top-level site that issues tokens and embeds areyouhuman.example
publisher.example - Future top-level site that redeems tokens and embeds areyouhuman.example
```

1.  User visits `original.example`, which loads `areyouhuman.example`.
1.  `areyouhuman.example` verifies the user is a human, and calls `fetch('areyouhuman.example/trust-token-issue', {privateToken: {...}})` with origin info consisting of "original.example,areyouhuman.example,publisher.example". The browser stores the trust tokens associated with this shared context.
1.  Sometime later, the user goes to `publisher.example`, which loads `areyouhuman.example`.
1.  `publisher.example` wants to know if the user has previously been authorized by areyouhuman.example by calling `fetch('publisher.example/trust-token-redeem', {privateToken: {...}})` with origin info equal to "original.example,areyouhuman.example,publisher.example". If a token exists, the browser uses it, otherwise the promise returned fails.

### Complete Example

```
// First party page origin.example
<html>
<head>
<script src="https://humanchecker.example/scripts/humanChecker.js"></script>
</head>
<body>
<button onclick="checkHumanity()">amihuman?</button>
</body>
</html>

// Embedded Javascript (humanChecker.js) from humanchecker.example
function checkHumanity() {
	let tokenType = ...; // type of token requested
	let tokenKey = ...; // public key of issuer
	let redemptionContext = ...; // token redemption context
  let tokenContext = {
    // origin-info is not specified, so it defaults to the origin for this script (humanchecker.example)
		"redemption-context": redemptionContext,
  };

	if (document.hasPrivateToken(privateTokenContext: tokenContext) {
    // If there is a token, redeem it.
		fetch('https://issuer.example/trust-token-redeem', { // Browser needs to send this to the Attester, not to the issuer or the origin.
		 	method: 'POST',
		 	privateToken: {
		 		"privateTokenContext": tokenContext,
		 	}
		 }).then((response) => /* do something with response */)
	 } else {
    // Otherwise, issue a token
    fetch('https://issuer.example/trust-token-issue', { // Browser needs to send this to the Attester, not to the issuer or the origin.
      method: 'POST',
      privateToken: {
        "tokenType": tokenType,
        "tokenKey": tokenKey,
        "privateTokenContext": tokenContext,
      }
    }).then((response) => /* do something with response */)
	}
}
```