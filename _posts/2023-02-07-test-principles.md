---
title: Test Principles
tags: [test]
readtime: true
---

The purpose of writing tests is to "support the ability to change. Whether you're adding new features, doing a refactoring focused on code health, or undertaking a larger redesign, automated testing can quickly catch mistakes, and this makes it possible to change software with confidence."<sup>[1][1]</sup> This is obvious, but is sometimes forgotten and instead replaced by unhelpful metrics like achieving a certain code coverage.

This article is mainly a consolidation of quotes from big tech companies like Google and other experts in the industry like Kent Beck, containing pointers on how to write good tests to achieve the above goals. I have read these 2 years ago and have since applied it at work, and I personally found these pointers to be very helpful.

As I have not had the opportunity to write E2E tests and so can't vouch for their usefulness, I only include quotes and personal opinions for these other levels of the test pyramid and some general principles that are applicable to all levels of the test pyramid:

1. Internal
2. Unit
3. Integration

## Test Pyramid

### Internal Tests

Internal tests verify internal implementation details that are not exposed to the end-user consuming the software. Generally, internal tests are costly.

> Maintainable tests are ones that "just work": after writing them, engineers don't need to think about them again until they fail, and those failures indicate real bugs with clear causes... You shouldn't need to touch that test again as you refactor the system, fix bugs, or add new features.<sup>[2][2]</sup>

Mocking (or stubbing), a key component of internal tests, causes tests to brittle.

> When mocking frameworks first came into use at Google, they seemed like a hammer fit for every nail... It wasn't until several years and countless tests later that we began to realize the cost of such tests: though these tests were easy to write, we suffered greatly given that they required constant effort to maintain while rarely finding bugs...
>
> Stubbing leaks implementation details of your code into your test. When implementation details in your production code change, you'll need to update your tests to reflect these changes.<sup>[3][3]</sup>

So, internal tests should be dis-preferred. If it is necessary to write internal tests, consider testing observable behavior instead of implementation detail to make these tests more maintainable:

> Think about
>
> if I enter values x and y, will the result be z?
>
> instead of
>
> if I enter x and y, will the method call class A first, then call class B and then return the result of class A plus the result of class B?<sup>[4][4]</sup>

### Unit Tests

Unit tests verify the behaviour of public APIs while using mock services (e.g. database, message queue).

> If a method or class exists only to support one or two other classes (i.e., it is a "helper class"), it probably shouldn't be considered its own unit, and its functionality should be tested through those classes instead of directly.
>
> If a package or class is designed to be accessible by anyone without having to consult with its owners, it almost certainly constitutes a unit that should be tested directly, where its tests access the unit in the same way that the users would.
>
> If a package or class can be accessed only by the people who own it, but it is designed to provide a general piece of functionality useful in a range of contexts (i.e., it is a "support library"), it should also be considered a unit and tested directly.<sup>[2][2]</sup>

> At Google, we've found that engineers sometimes need to be persuaded that testing via public APIs is better than testing against implementation details...
>
> Tests using only public APIs are, by definition, accessing the system under test in the same manner that its users would. Such tests are more realistic and less brittle because they form explicit contracts: if such a test breaks, it implies that an existing user of the system will also be broken. Testing only these contracts means that you're free to do whatever internal refactoring of the system you want without having to worry about making tedious changes to tests.<sup>[2][2]</sup>

### Integration Tests

Integration tests verify the behaviour of public APIs while using real services.

> The more your tests resemble the way your software is used, the more confidence they can give you.<sup>[5][5]</sup>

> As indicated here, the pyramid shows from bottom to top: Unit, Integration, E2E. As you move up the pyramid the tests get slower to write/run and more expensive (in terms of time and resources) to run/maintain. It's meant to indicate that you should spend more of your time on unit tests due to these factors.
>
> One thing that it doesn't show though is that as you move up the pyramid, the confidence quotient of each form of testing increases. You get more bang for your buck.
> 
> So while having some unit tests to verify these pieces work in isolation isn't a bad thing, it doesn't do you any good if you don't also verify that they work together properly. And you'll find that by testing that they work together properly, you often don't need to bother testing them in isolation.<sup>[6][6]</sup>

See [gif](https://twitter.com/erinfranmc/status/1148986961207730176).

> When you mock something, you're making a trade-off. You're trading confidence for something else. For me, that something else is usually practicality - meaning I wouldn't be able to test this thing at all, or it may be pretty difficult/messy, without mocking.<sup>[5][5]</sup>

## General Principles

### Write just enough Tests

> there's one more pitfall to avoid: duplicating tests throughout the different layers of the pyramid. While your gut feeling might say that there's no such thing as too many tests let me assure you, there is. Every single test in your test suite is additional baggage and doesn't come for free.<sup>[4][4]</sup>

> I get paid for code that works, not for tests, so my philosophy is to test as little as possible to reach a given level of confidence.<sup>[7][7]</sup>

> Test until fear is transformed into boredom. Do what seems to help until it doesn’t seem to help any more, then stop.<sup>[8][8]</sup>

> The way that tests help you solve problems is by mitigating risk... Test code itself does not directly deliver value. It's valuable for its loss prevention, both in terms of the real harm of bugs (lost revenue, violated privacy, errors in results) and in terms of the time spent detecting and fixing those bugs. We don't like paying that cost, so we pay for tests instead. It's an insurance policy... Selecting your policy is selecting how much risk you want to take on, or how much you can afford to avoid.<sup>[9][9]</sup>

Practically, when coming up with a suite of test cases, consider starting from the top of the test pyramid: firstly, identify the test cases that are best covered by E2E tests, then Integration tests, then Unit tests, then Internal tests.

### Maximize Test Clarity

> Test clarity becomes significant over time. Tests will often outlast the engineers who wrote them.
>
> [While the purpose of] unclear production code [can be determined] with enough effort by looking at what calls it and what breaks when it's removed, [that's not the case] with an unclear test, since removing the test will have no effect other than (potentially) introducing a subtle hole in test coverage.
>
> In the worst case, these obscure tests just end up getting deleted when engineers can't figure out how to fix them.<sup>[2][2]</sup>

#### Trivial to Inspect

> Clear tests are trivially correct upon inspection; that is, it is obvious that a test is doing the correct thing just from glancing at it.<sup>[2][2]</sup>

For example,

> you shouldn't use If statements anywhere in your tests.
>
> Your tests should be as dumb as possible. They basically need to verify only one use case. With those if statements, you diverged from that guideline.<sup>[10][10]</sup>

> consider tolerating some duplication when it makes the test more descriptive and meaningful... [so that each test] can be understood entirely without leaving the test body. A reader of these tests can feel confident that the tests do what they claim to do and aren't hiding any bugs.<sup>[2][2]</sup>

## References

1. [Software Engineering at Google - Testing Overview](https://abseil.io/resources/swe-book/html/ch11.html)
1. [Software Engineering at Google - Unit Testing](https://abseil.io/resources/swe-book/html/ch12.html)
1. [Software Engineering at Google - Test Doubles](https://abseil.io/resources/swe-book/html/ch13.html)
1. [Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
1. [The Merits of Mocking](https://kentcdodds.com/blog/the-merits-of-mocking)
1. [Write tests. Not too many. Mostly integration.](https://kentcdodds.com/blog/write-tests)
1. [How Deep Are Your Unit Tests](https://stackoverflow.com/a/153565/8828382)
1. [Which Kinds of Tests Should I Write? Revisited](https://blog.thecodewhisperer.com/permalink/which-kinds-of-tests-should-i-write-revisited)
1. [Too much of a good thing: the trade-off we make with tests](https://ntietz.com/blog/too-much-of-a-good-thing-the-cost-of-excess-testing)
1. [Principles for Writing Valuable Unit Tests](https://techleadjournal.dev/episodes/58/)

[1]: https://abseil.io/resources/swe-book/html/ch11.html
[2]: https://abseil.io/resources/swe-book/html/ch12.html
[3]: https://abseil.io/resources/swe-book/html/ch13.html
[4]: https://martinfowler.com/articles/practical-test-pyramid.html
[5]: https://kentcdodds.com/blog/the-merits-of-mocking
[6]: https://kentcdodds.com/blog/write-tests
[7]: https://stackoverflow.com/a/153565/8828382
[8]: https://blog.thecodewhisperer.com/permalink/which-kinds-of-tests-should-i-write-revisited
[9]: https://ntietz.com/blog/too-much-of-a-good-thing-the-cost-of-excess-testing
[10]: https://techleadjournal.dev/episodes/58/
