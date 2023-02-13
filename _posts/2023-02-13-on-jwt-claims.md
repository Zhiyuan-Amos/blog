---
title: On JWT Claims
tags: [jwt]
readtime: true
---

> We are very particular about what data goes in there because we don't want Passport to become a kitchen sink of things that people want. We choose it based on how frequently the attribute is used, and something that doesn't change that often. Let's say customer ID or account owner ID does not change, but let's say membership status can change. That's why we don't put membership status in Passport, even if many teams have asked us. We go by the use case and we look at what is feasible and what is not. Generally, we are very strict about what goes in Passport. For services that want additional information that is not in Passport, we encourage them to make a remote call to the service, like let's say subscriber service, like give them a Passport and they'll give you more information.

[Netflix](https://www.infoq.com/presentations/netflix-user-identity/)

> the token is not about transporting information. It's about expressing... which API is the Client allowed to call... [otherwise, you will] run into exactly those problems [like] if you change permissions it doesn't become effective unless the user logs out and logs in again

[Securing SPAs and Blazor Applications using the BFF (Backend for Frontend) Pattern](https://www.youtube.com/watch?v=DdNssiaIY_Q&t=4982s)
