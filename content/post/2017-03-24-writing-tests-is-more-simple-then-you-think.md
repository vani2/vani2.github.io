---
categories: tests motivation
date: "2017-03-24T10:37:33Z"
title: Writing tests is more simple then you think
---

![Rocket on launch](/assets/2017-03-24/header.jpg)

> Do you write tests on your project(s)?

If not, you probably try to find reasons why you don’t. If you do, send this article to your friends who don’t.

<!--more-->

It’s the wrong point, because writing tests are simple, helps you personally, your team members, and your project, for sure.

I know some of you can tell that it’s a waste of time(equals $$$), you need to change your tests after you change a code an so on. But stop doing it. I bet you regret that fact. Every true programmer/developer/engineer wants to make solid things and be sure about its high quality.

I guess that some are not really happy about the TDD technique. It’s hard to write tests before implementation, especially if your business requirements or/and design change several times in a day. More about TDD [in this article](https://github.com/tpn/pdfs/blob/master/Realizing%20Quality%20Improvement%20Through%20Test%20Driven%20Development%20-%20Results%20and%20Experiences%20of%20Four%20Industrial%20Teams%20(nagappan_tdd).pdf). [This](http://citeseer.ist.psu.edu/viewdoc/summary?doi=10.1.1.93.5570) is also recommended to read.

If you don’t write tests you hope your manual(monkey) testing is full grade. But when you write tests you (must) concentrate on what you test. This helps you enumerate all edge scenarios and input data. When you test by hands it’s often not easy to reproduce all cases and you can skip them.

Writing tests help you to choose the right architecture. You can’t be good in testing if your code has no loose coupling (aka too many singletons, spaghetti, and [others](https://en.wikipedia.org/wiki/Anti-pattern)).

It helps you better understand teammates' code and your own especially after weeks/months/years of this inactive part of the functionality.

> Do you need to change tests after you change your code?

Yes. Why not, if you need to test it(manually) again after this change?

> How to estimate time for tasks if I want to write tests?

I estimate tasks with time for writing tests. On average, it costs about 10–20%. But your code will have fewer bugs therefore it’ll spend less time for bug fixing.

> How to start?

Try from some small formatters, any other helper functions/methods. Then extend your experience to UI-tests and API-tests (if applicable). Get familiar with stubs, mocks, BDD, and so on.

## What’s next?

Start writing tests right now. Not tomorrow, not next week. Right now.
