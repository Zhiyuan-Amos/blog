---
title: OAuth for SWEs working on AuthZ systems
tags: [oauth]
readtime: true
---

## Preface

Over the past 1 year at my job as a Software Engineer, I had the opportunity to deep-dive into OAuth 2 while using Keycloak, Azure Entra ID (formerly known as Azure AD) and Auth0. Along the way, I encountered several uncommon scenarios / got-chas that aren't well-documented which I'll share here.

I am not covering everything related to OAuth 2 because there are already many great resources online, so this article assumes that you understand core concepts of OAuth 2 including:

1. What OAuth 2 solves
2. The key entities of OAuth 2 and how they interact with each other
   1. Client
   2. Resource Server
   3. Authorization Server
3. Access Token & Refresh Token

If any of these are unfamiliar to you, I highly recommend you to go through this [video tutorial](https://www.youtube.com/watch?v=t18YB3xDfXI) (from 00:00 - 10:56) which helped me tremendously when I was starting out to learn OAuth 2.

## OAuth 2

### Problem of long-running operations and expired Access Tokens

Typically, in a microservices architecture, the Access Token issued to the Client is used for authorization for both external-to-internal (from Client to Resource Server) and internal-to-internal (between services in Resource Server) requests. Suppose the following scenario:

1. Client sends a request to Service A.
2. Service A processes the request, which takes a long time to complete.
3. Then, Service A sends requests to other services.

As Access Tokens are short-lived, the Access Token may expire before Service A completes processing the request. Consequently, other services reject Service A's request.

The following solutions do not work:

1. Refresh the Access Token: Service A does not have the Refresh Token.
2. Use Service Account's Access Token: Service Account's Access Token does not contain User information which may be required for the other services' business & authorization logic.
3. Disable validation of Access Token's lifetime by internal services: [Violates Defence in Depth](https://github.com/OWASP/CheatSheetSeries/issues/1087#issuecomment-1436930264) as the internal services might be incorrectly configured to be directly accessible from external network.

What works is using a separate token for external-to-internal ("external token" for brevity) and internal-to-internal ("internal token" for brevity) requests:

1. When Clients send requests to the Resource Server, the API Gateway exchanges the external token for an internal token with the Authorization Server.
2. The API Gateway then forwards the request with the internal token to internal services.

A variant of Token Refresh using [mTLS or the internal service's credentials](https://cheatsheetseries.owasp.org/cheatsheets/Microservices_Security_Cheat_Sheet.html#service-to-service-authentication) can be used to obtain a new internal token.

[OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Microservices_Security_Cheat_Sheet.html#using-a-data-structure-signed-by-a-trusted-issuer) recommends this approach.

> after the external request is authenticated by the authentication service at the edge layer [referring to API Gateway], a data structure representing the external entity identity (e.g., containing user ID, user roles/groups, or permissions) is generated, signed, or encrypted by the trusted issuer and propagated to internal microservices.

Netflix (see references [1](https://www.infoq.com/presentations/netflix-user-identity/) & [2](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)) uses this approach.

> we created a new identity structure called Passport. What is Passport? It is an identity structure created at the edge [referring to API Gateway] for each request and services consumed in the scope of same request. It contains user and device identity information. It is internal to Netflix ecosystem, meaning it is an internal identity token that we don't send it out back to the device.

This solution also comes with other improvements:

1. Ease of handling different types of tokens
2. Better security

For more information about these improvements, see [OWASP's article](https://cheatsheetseries.owasp.org/cheatsheets/Microservices_Security_Cheat_Sheet.html#external-entity-identity-propagation).

### Different behaviours across OAuth 2 providers

Different OAuth 2 providers implement the finer details of OAuth 2 differently and some of which affect user experience. For example, during the login process, the `state` parameter is used to protect against CSRF attacks (more information on how it works in this [SO answer](https://stackoverflow.com/a/35988614/8828382)). This defence does not require OAuth 2 providers to store the value of `state` in its database, but Auth0 does that. Auth0 then clears the record after a set period of time (see [Auth0's documentation](https://auth0.com/docs/authenticate/login/auth0-universal-login/configure-default-login-routes#users-bookmark-login-page)). Thus, login fails if the user remains on the login page for a prolonged period of time and attempts to log in after Auth0 has cleared the record. Azure Entra ID does not have this limitation.

For more information about the differences across OAuth 2 providers, see [nango's article](https://www.nango.dev/blog/why-is-oauth-still-hard).

### 1 Entity to 1 Token

Even though Access Tokens are typically readable because they are JWTs, Clients must not read them (see [OAuth's documentation](https://oauth.net/2/access-tokens/)). This allows the Authorization Server to modify the token format to perhaps an encrypted token without breaking existing Clients.

The following table illustrates which OAuth 2 entity is allowed to read and/or use which token.

| OAuth 2 Entity       | Token         |
|----------------------|---------------|
| Client               | Id Token      |
| Resource Server      | Access Token  |
| Authorization Server | Refresh Token |

Apart from achieving separation of concerns, this design also provides better security. For example, it is technically possible to merge the Access Token and Refresh Token into a single token. However, Resource Servers are generally considered to be less secure than Authorization Servers. If the token is leaked by the Resource Server, then a malicious actor is one step closer to being able to access the user's resources for a prolonged period of time by performing Token Refresh with the leaked token. So, keeping the Access Token and Refresh Token separate improves Defence in Depth (see [SO answer](https://stackoverflow.com/a/77026028/8828382)).

### Personal Access Token (PAT) vs OAuth 2

PAT is a valid alternative to OAuth 2 and popular software like GitHub allow users to generate PATs. However, this is generally an inferior alternative to OAuth 2 for the following reasons:

1. **Security**: If the PAT is leaked, a malicious actor can gain access to the user's resources for a prolonged period of time. This is because not all PAT providers support expiration of PATs, so users have to manually revoke it upon detecting unauthorized usage. For providers that support expiration of PATs, the minimum lifetime is typically at least 1 day.

    Whereas if both Access Token and Refresh Token are leaked, a malicious actor can only gain access to the user's resources for a short period of time. This is because the lifetime of Access Tokens are typically short, ranging from a few minutes to a few hours.
    1. For Confidential Clients, the malicious actor cannot refresh the Access Token as Token Refresh requires the Client's Id and Secret too.
    2. For Public Clients, even though the malicious actor can refresh the Access Token, there are several mitigations in place such as invalidating of Refresh Tokens upon user logout and limiting its maximum lifetime (e.g. Azure Entra ID limits it to [24 hours](https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens)).

2. **Standards**: OAuth 2 is an industry-standard protocol while PAT has no standards governing it, which means there's no industry-wide recommendations for best practices and pitfalls to avoid when implementing a PAT solution.

PAT can be considered over OAuth 2 if it is necessary to make it easy for third-party developers to try out your APIs, as it only takes 2 steps to do so:

1. Create a PAT on your software's website
2. Add the PAT to HTTP Requests sent to your software

Whereas creating an OAuth 2 Client and importing a library to configure the OAuth 2 Client not only takes a couple more steps, it will also likely be non-trivial for a developer who is unfamiliar with OAuth 2.

## My experience with Keycloak

Of all the OAuth 2 providers, I used Keycloak the most, so I'll briefly share of my experience with using Keycloak in production.

Keycloak is a reasonably mature tool especially since it was recently added to Cloud Native Computing Foundation as an [incubating project](https://www.cncf.io/projects/keycloak/). I think using Keycloak is fine for the use case of administration for an organization. It should be noted that Keycloak:

1. [Does not have LTS support](https://github.com/keycloak/keycloak/discussions/25688) so major version upgrades may be necessary when security patches are released. However, major version upgrades may cause breaking changes (which I have encountered a few times).

2. Contains legacy features (since it has been around for a while). For example, ["login timeout"](https://www.keycloak.org/docs/latest/server_admin/index.html#_timeouts) causes the login to fail if the user remains on the login page for a prolonged period of time and attempts to log in after exceeding the duration of "login timeout" (see [SO](https://stackoverflow.com/questions/65266761/what-is-the-reason-for-the-login-timeout-setting-and-functionality)). "Login timeout" was probably meant to protect against Session Fixation attacks (see the answer of the above SO link. Note that it is not officially confirmed by [Keycloak](https://github.com/keycloak/keycloak/issues/21889)) but I wasn't able to perform the exploit. Thus, it appears that "login timeout" does not improve security and worsens the users' experience.

If Keycloak is integrated as part of a software solution, it should be noted that Keycloak's API specification may be lacking in details. For example, it does not specify the possible HTTP status codes that the API returns on error scenarios, and some of the request body's parameters may have undocumented restrictions such as input length restrictions. Thus, it may be helpful to create a service to wrap Keycloak's API and to write detailed API specification for other teams to consume.

Having encountered multiple unforeseen issues because exceptional cases are generally not spelt out in the documentation, sometimes I wonder if implementing an OAuth 2 Server using a well-established library (e.g. [OpenIddict](https://github.com/openiddict/openiddict-core) for C#) may reduce maintenance effort in the long run. This is because Keycloak only abstracts implementation, not knowledge: Whether using Keycloak or implementing an OAuth 2 Server, developers must do the hard work of understanding OAuth 2 well. With good OAuth 2 libraries, the cost of implementing an OAuth 2 Server may be lesser than the cost of dealing with the intricacies of Keycloak and identifying and plugging the gaps of Keycloak. Of course, your mileage may vary; you may find Keycloak working nicely for your use cases, especially so if your use cases aren't complex.

## Conclusion

I hope my experiences and learnings above are helpful to you! Let me know in the comments below if the content is unclear and if you have questions.

Thanks for reading!
