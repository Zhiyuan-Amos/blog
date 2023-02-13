---
title: Don't Handwrite OAS
tags: [openapi]
readtime: true
---

See [article](https://sookocheff.com/post/api/the-false-dichotomoy-of-design-first-and-code-first-api-development/). Personal opinions:

1. In my research, I've never found official OpenAPI documantation or any other non-official guides recommending handwriting OAS.
2. Typically, there's no benefit in handwriting the OpenAPI schema.
3. Writing JSON or YAML is typically harder (especially in complex scenarios) compared to writing in C#: Creating a stub Controller (i.e. return `501 Not Implemented` or `200 OK`, or disable it with a feature flag which returns `404` by default) with the request & response Models.
4. Increased likelihood of merge conflicts because modifications are made in a single file (e.g. developers adding a model / endpoint at the end of the file).
5. If you generate the Controller or just the Controller's Interface, you will find it hard to deviate from the standard Controllers design e.g. Vertical Slice Architecture using [ApiEndpoints](https://github.com/ardalis/ApiEndpoints).
6. Similar for Model Generation, it's harder to add features such as defensive copying for objects in the constructor (e.g. `List`) or adding custom convenience constructors. It's hard / impossible to annotate custom JSON Converters or custom Validation on the Model as well.
