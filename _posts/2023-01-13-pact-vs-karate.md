---
title: Pact vs Karate
tags: [test, pact, karate]
readtime: true
---

Karate's [definition of Consumer-Driven Contract Tests](https://www.linkedin.com/pulse/api-contract-testing-visual-guide-peter-thomas/) (CDCT) is opposite to that of Microsoft, Pact & Martin Fowler:

> if you only focus on validating schema, your contract test will not be very useful... your Contract Test should be a true functional test of the Provider without cutting any corners... I see a problem with some contract-testing tools and frameworks that encourage you to NOT write true functional tests when attempting contract tests. I think that is a cop-out.

and that

> your Contract Test has another important role, which is to ensure that your mock simulates the Provider to the extent possible

Whereas [Pact](https://docs.pact.io/consumer#use-pact-for-contract-testing-not-functional-testing-of-the-provider) says:

> Contract tests focus on the messages that flow between a consumer and provider, while functional tests also ensure that the correct side effects have occurred.

> Functional testing is about ensuring the provider does the right thing with a request. These tests belong in the provider codebase, and it's not the job of the consumer team to be writing them. Contract testing is about making sure your consumer team and provider team have a shared understanding of what the requests and responses will be in each possible scenario.

And [Martin Fowler](https://sqa.stackexchange.com/questions/42050/whats-the-difference-between-integration-and-contract-testing-of-microservices) says:

> [Contract tests] do not test the behavior of the service deeply but that the inputs and outputs of service calls contain required attributes and that response latency and throughput are within acceptable limits.

Karate's CDCT tests (based on this [sample](https://github.com/karatelabs/karate/blob/master/karate-demo/src/test/java/mock/contract/payment-service.feature)) do not run independently. Whereas according to Pact:

> Each interaction is tested in isolation, meaning you can’t do a PUT/POST/PATCH, and then follow it with a GET to ensure that the values you sent were actually read successfully by the Provider. For example, if you have an optional surname field, and you send lastname instead, a Provider will most likely ignore the misnamed field, and return a 200, failing to alert you to the fact that your lastname has gone to the big /dev/null in the sky.

It's worth pointing out that [Microsoft](https://microsoft.github.io/code-with-engineering-playbook/automated-testing/cdc-testing/) stated "**Pact is the de-facto standard** to use when working with CDC." Therefore, I'm more inclined to take Pact's & Martin Fowler's definition of CDCT.

