---
title: "First impressions aws cdk"
authors: ["Geert Van Wauwe"]
draft: true
---

not a big fan of cdk, the construct and id passed in almost every constructor does not seem right to me as a developer.
why not use dependency injection of objects and expose them in higher level constructs.
having the ability to make reusable objects is great though, so functionally it does more than what it should.
However testing cdk resources is not really great, since it seems to lack DI.
the construct parameter is still vague, and what do you need an id for?