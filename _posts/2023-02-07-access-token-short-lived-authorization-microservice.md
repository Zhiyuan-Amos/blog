---
title: Using short-lived Access Tokens for Authorization in a microservice architecture
tags: [authorization, security, jwt]
readtime: true
---

Access Tokens can expire before all operations across all microservices have completed. If each microservice validates the lifetime of the Access Token, then it's possible that Authorization fails somewhere in the chain of microservices involved in the operation.

This can be solved by making the following changes:

1. The API Gateway converts external tokens (tokens coming from external clients) to internal tokens (or Passport, as Netflix terms it). Internal services only uses and understands internal tokens.
2. The Passport must expire after a set period of time, and it contains a corresponding Refresh Token can be used to renew the Passport. The service performing Passport renewal must send its Certificate for mTLS.
