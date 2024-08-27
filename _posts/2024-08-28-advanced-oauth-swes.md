---
title: OAuth for SWEs
tags: [oauth]
readtime: true
---

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
