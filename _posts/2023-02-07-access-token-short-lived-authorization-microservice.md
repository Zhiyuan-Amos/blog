---
title: Using short-lived Access Tokens for Authorization in a microservice architecture
tags: [authorization, security, jwt]
readtime: true
---

Access Tokens can expire before all operations across all microservices have completed. If each microservice validates the lifetime of the Access Token, then it's possible that Authorization fails somewhere in the chain of microservices involved in the operation.

This problem can be resolved by using an API Gateway to validate the lifetime of the Access Token for **external** requests, while internal microservices (receiving requests from API Gateway & other internal microservices, i.e. **internal** requests) should not perform the validation (for clarity, other forms of validation should still be done e.g. signature, `iss`, `aud`). 

Even though I'm unable to find an information online explaining why validating lifetime of internal requests is not required, however:

1. In some of the guides on using Access Token for internal microservices, there's no mention of doing so:
    1. [OWASP Cheat Sheet Series on Microservices Security](https://cheatsheetseries.owasp.org/cheatsheets/Microservices_security.html).
    2. Netflix has a [slightly complex solution](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602), whereby the API Gateway converts the incoming JWT to a `Passport` which contains user context (e.g. user ID, user roles/groups). There's no mention of `Passport` having an expiration time, so there's no lifetime to be validated (which isn't a security concern because `Passport` is only used for internal communications and never exposed to the external world. `Passport` is not persisted by any of the internal microservices, so it is effectively "scoped to the life of the request [chain]").
2. There's no benefit of doing so.

Alternatives that don't work include:

1. Using a long-lived Access Token, as Access Tokens cannot be revoked upon issuance.
2. Having the microservice perform Token Refresh, as the microservice should not have access to the Refresh Token, nor the Client Id (and Client Secret) of the Client that obtained the Access Token.
